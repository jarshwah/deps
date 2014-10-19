DEP X: New implementation for ORM expressions
=============================================

:Created: 2014-09-09
:Author: Anssi Kääriäinen
:Status: Draft

Abstract
========

This DEP details a new and simplified way of building query expressions for the
ORM. The aim is to simplify the internal implementation of expressions, and to
detail a public API that will allow user defined expressions.

Query expressions allow SQL constructs to be used from Django. F() expressions
and aggregate functions are examples of existing expressions, but they each use
their own internal implementations.

Current Implementation
======================

Currently Django's ORM splits expression implementations into two parts:

  - Public facing API: for example django.db.models.expressions.F()
  - Evaluation: for example django.db.models.sql.expressions.SQLEvaluator

Lets consider as an example the way that F('foo') + F('bar') is implemented.

First, F('foo') + F('bar') creates the following datastructure::

    ExpressionNode('+')
      children:
        F('foo')
        F('bar')

The leaf nodes are usually F() objects (they could be other types of
ExpressionNodes too) and the internal nodes are usually plain ExpressionNodes.
We call the result of this computation an F-expression.

When the F-expression created by F('foo') + F('bar') is given to .filter(),
the resolution of the F-expression happens in a couple of stages. First, the
node is added to the query. The steps when adding to the query are:

  - Query.prepare_lookup_value() detects that the given value is an instance of
    django.db.models.expressions.ExpressionNode
  - A SQLEvaluator is created which has references to both the F-expression
    and the query the F-expression is added to.
  - The creation of SQLEvaluator calls the F-expressions prepare() method.
    The prepare method takes the evaluator and Query as parameters.
  - The F-expressions prepare() method calls back to the evaluators
    prepare_node() or prepare_leaf() method.
  - The prepare_node() method is called for internal expression nodes
    (ExpressionNode('+') in the example). The evaluators prepare_node()
    method then calls prepare() for each child (F('foo') and F('bar')
    in the example).
  - The prepare_leaf() method is called from each leaf nodes prepare() method.
    The prepare method is responsible for resolving references and doing
    other essential setup. For example F('foo') would ask the query to
    resolve the 'foo' reference to something that can be used in the query.
  - The total call stack is then as follows (starting from
    prepare_lookup_value())::

      SQLEvaluator(node, query)  # node in this example is ExpressionNode('+')
                                 # with children F('foo') and F('bar')
        -> node.prepare(evaluator=self, query)
             -> evaluator.prepare_node(node=self, query)
               -> for all children in node: # node's children are F('foo')
                                            # and F('bar')
                     children.prepare(evaluator=self, query)
                         -> evaluator.prepare_leaf(node=self, query)
                            # SQLEvaluator.prepare_leaf() is called with
                            # node=F('foo') or node=F('bar')

The execution of a node happens in similar fashion. The compiler calls as_sql()
of the SQLEvaluator, then SQLEvaluator calls evaluate() on the node, which
calls evaluate_node() or evaluate_leaf() similar to prepare() above.

The problem with this setup is that the evaluator needs to know exactly how to
add a node into a query, and it needs to generate SQL for the node. In
particular this means that the generated SQL is controlled by the evaluator,
not the node, and that the evaluator has to know every possible node type used
in ORM queries.

To add new expressions, the evaluator and expression nodes need to be modified
in tandem. For example, DateModifierNode needs a method
evaluate_date_modifier_node() in the evaluator. So, to implement a new expression,
one needs to alter the SQLEvaluator class. Unfortunately this is practically
impossible because the SQLEvaluator is private API, and the SQLEvaluator is
global. So every node type needs to be supported by the same SQLEvaluator
class.

Another complexity of the current approach is found from the evaluator <->
expression dance. As seen above, the call graph is somewhat complex making
it hard to understand what exactly is happening.

Finally, if one wants to mix ExpressionNode functionality with other SQL
expressions (like aggregates) the current way doesn't allow for that.
Aggregates aren't ExpressionNodes at all, but they implement a similar but
disparate flow.

The reasoning why the SQLEvaluator <-> ExpressionNode flow is performed is the
ability to customize the generated query string by implementing a custom
Evaluator. For example, MongoDB ORM backend could have MongoEvaluator that
knows how to add F() objects to query, and how to produce a valid query string
for F('foo') + F('bar'). Thus the user can use the same public API for
MongoDB. The custom evaluator would need to change in lockstep with the existing
evaluator if it wanted to support the same public expressions.

It should be noted that the Evaluator API isn't documented and isn't public, and
so it can't be relied upon between versions.

Improvement proposal
====================

This DEP suggests to rewrite the expression handling code in the ORM.
Especially, there will be no SQLEvaluator class at all. The new implementation
is based on two ideas:

  1. New ExpressionNode base class is added to Django.
  2. Anything with resolve_expression() and refs_aggregate() methods can be
     used as expressions, as long as resolve_expression returns an
     implementation of ExpressionNode. In particular, F-objects aren't
     ExpressionNode instances, they just resolve to one.

The basic funtionality of the ExpressionNode class is as follows:

  - The class defines the base methods and attributes needed by all
    expressions. The set of methods and attributes is known as the SQL
    expression API.
  - A new CombinableMixin is added to the ORM. This mixin allows objects to
    be combined with +, - and similar operators by implementing `__add__`,
    `__sub__` and similar methods.
  - Combining two combinables returns a Expression instances. The
    Expression instance combines two nodes with an operator.
  - A bit surprisingly F-objects aren't subclasses of ExpressionNode, but they
    resolve to expressions which refer directly to a database column or other
    existing expression. For example F('somecol') resolves to a
    Col instance referencing database column "somecol". F('max_id') resolves
    to the left hand side of an existing aggregate Max('id') where
    qs.annotate(max_id=Max('id')) must have been defined first.
  - Python values resolve to ValueNode instances. That is, F('foobar') + 10
    will resolve to Col('foobar') + Value(10).
  - The expression returned from resolve_expression is added to the query.
  - Aggregates will be subclasses of Expression.
  - All Expressions can be used in .annotate() calls. This includes expressions
    other than aggregates.

As an example, lets consider the case of F('foo') + F('bar'). The `__add__`
method of F('foo') will create a new Expression(F('foo'), '+', F('bar'))
expression. When the expressions resolve_expression method is called,
the call tree looks like::

    expression.resolve_expression(query):
        self.lhs.resolve_expression(query)  # F('foo')
        self.rhs.resolve_expression(query)  # F('bar')

The lhs and rhs nodes will resolve their respective database columns from
the query. End result would be Expression(Col('foo'), '+', Col('bar')).

Execution would happen by calling as_sql(). Each col returns just
"table_ref"."colname", and the BinaryExpression then combines them with +::

    expression.as_sql(compiler, connection):
        # params not handled for brevity
        sql = [self.lhs.as_sql(), self.rhs.as_sql()]
        return connection.ops.combine_sql(
            self.operator, sql)

When compared to the call tree produced by Djangos current code, it is
immediately obvious that new expressions are much easier to understand.

There is currently very limited support for combining arbitrary types of
expressions (for example, doing F('textfield') + F('anothertext') doesn't
resolve to CONCAT() SQL). This proposal doesn't aim to solve arbitrary type
combination problem (though doing so should be possible later on). But it is
possible to write a custom ConcatNode::

    class ConcatNode(Expression):
        def __init__(self, lhs, rhs)
            super().__init__(lhs, rhs)

        def as_sql(self, compiler, connection):
            all_sql = []
            lhs_sql = self.lhs.as_sql()
            rhs_sql = self.rhs.as_sql()
            all_sql.append(lhs_sql)
            all_sql.append(rhs_sql)
            return 'CONCAT(%s)' % ', '.join(all_sql), params


Rationale of the changes
========================

This chapter summarizes why the changes are necessary for the ORM. Currently
the following things aren't possible:

  - The current code doesn't allow one to write custom expressions through
    a public API. While it is possible to write custom expressions using
    private API it is painful to do so (need to alter the global SQLEvaluator
    class).
  - Aggregates aren't expressions. For that reason Sum('foo') + Sum('bar')
    isn't possible.
  - Arbitrary expressions can't be used in .annotate() calls. Annotate only
    supports aggregates.
  - The current code is hard to understand.

The new expressions API allows writing custom expressions based on a public
API, the call graphs are easier to understand, aggregates are expression
subclasses and annotating a query with arbitrary expressions is fully supported
by the ORM. Expressions can't be used directly in other calls yet, but it is
possible, and has been demonstrated, that support for order_by can be achieved::

  .order_by(NullsLast(F('height') / F('weight')))

Possible problems
=================

The main identified problem is that SQLEvaluator class has remained mostly
stable from 1.0 days on (if not earlier). Similarly, the implementation of
aggregates has remained mostly stable from the addition of aggregation support
in the ORM. The suggested changes could cause problems for users who have
relied on this private API.

Possible ways to make the transition easier include:

  - Keep backwards compatibility for SQLEvaluator
  - Add django.db.models.sql.deprecations and django.db.models.deprecations
    modules. These would containt for example old-style implementations of F()
    objects, aggregates and other changed object classes.

Adding a backwards compatibility module will require a lot of work. Users have
been asked a couple of times for feedback about the suggested changes, but no
replies were given.

There is also a possibility that the changes will make it harder to write
"NoSQL" ORM implementations. Currently one can (at least theoretically) write
a custom evaluator for a NoSQL backend. The evaluator is responsible for
generating the correct query string for any node type used in the project.

There are a couple of ideas which should work equally well for the new approach.

The first approach is that whenever the NoSQL ORM sees an expression it
converts it to new type of specialized expression (for example, Concat is
converted to NoSQLConcat). This could be made even easier if we add
Query.convert_expression(expression) method. This method is called always for
any expression used in ORM queries. The default implementation will return
self, but for NoSQL ORM the method could return a converted node. Converting
the node will require knowledge of the internal structure of the node, but
that same problem exists when SQLEvaluator prepares or generates a query
string for given node.

The second approach is similar to the first approach, but instead of
generating different node types, it wraps the node with a generic
NoSQLExpressionWrapper. The NoSQLExpressionWrapper does conversions
between the ORM and the original node implementation.

The third approach is to just use the as_vendor approach for the nodes. This
is the easiest approach to implement, but without trying it is hard to say
if this approach is sufficient.

In any case the first two approaches are sufficient to implement similar
functionality than what SQLEvaluator gives. Of course, existing projects
(django-nonrel for example) will need to be updated.

Implementation
==============

Pull request https://github.com/django/django/pull/2496/ implements all suggested
changes in this DEP.

Copyright
=========

This document has been placed in the public domain per the Creative Commons
CC0 1.0 Universal license (http://creativecommons.org/publicdomain/zero/1.0/deed).
