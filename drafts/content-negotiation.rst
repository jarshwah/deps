Content Negotiation Improvements
=================================

Created: 2014-04-14
Author: Mark Lavin
Status: Draft

Overview
=================================

Define additional hooks and utilities in the request/response cycle to manage
content negotiation with HTTP clients.

Rationale
=================================

Django provides a number of utilities for parsing form-urlencoded data and rendering
HTML templates. This fits a large use case for people building websites with Django.
As more users look to use Django to build webservices they have to deal with
`content negotiation <http://www.w3.org/Protocols/rfc2616/rfc2616-sec12.html>`_ without
similar utilities. Currently any parsing of a request body when the Content-Type
is not application/x-www-form-urlencoded or multipart/form-data is left up to the
user's application. Similarly handling requests which do not Accept text/html is
again left up to the user. Without some of the basic utilities in Django, users
instead look to external applications such as django-rest-framework, django-tastypie,
piston, django-nap or any number of applications from https://www.djangopackages.com/grids/g/api/.
Most of these reuseable applications have invented their own method for handling
this problem and each could benefit from standardization in this area while continuing
to provide utilities in related areas such as how resources are defined, linked
and validated. These applications will serve as a place for Django to start
on its implementation, building off of what is known to work in production and
improving where it can given its ability to potentially modify or clean up internal
APIs which could not otherwise be changed by a third party application.

This has been discussed before on django-developers

- `Proposal: better support for generating rest apis in django core <https://groups.google.com/d/msg/django-developers/Qr0EorpgYKk/W28GwS1qLe0J>`_
- `Support POST of application/json content type <https://groups.google.com/d/msg/django-developers/s8OZ9yNh-8c/yWeY138TpFEJ>`_

On each occasion Tom Christie, author of Django Rest Framework, chimed in with his
insight and support of the idea. He also created a ticket with a minimal proposal
which was accepted in https://code.djangoproject.com/ticket/21442. Some initial
work has been started by him https://github.com/tomchristie/django/tree/ticket_21442

Implementation
=================================

The proposed implementation builds off of the proposal from #21442

Request Parsing
---------------------------------

- Define an API for parsing incoming request bodies and registering them with the framework
- Break out the urlencoded and multipart parsers from HttpRequest according to this API
- Define a new request.DATA to be used by the parsing framework for the deserialized body

Response Types
---------------------------------

- Define a serialization framework for generating responses from data based on content type
- Allow serializers to be registered for use in negotiating responses
- Define a HttpResponse subclass which consumes the current request, view defined data, and potential serializers to be negotiated

Open Questions and Challenges
---------------------------------

- At what level should these parsers/serializers be defined? Globally? Per request/view?
- What additional parsers, if any, should be included in Django?
- What initial serializers, if any, should be included in Django?

Copyright
=================================

This document has been placed in the public domain.
