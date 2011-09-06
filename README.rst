=======================================
 django-piston (PBS Education Version)
=======================================

:Info: piston is a REST API framework for use with django projects
:Keywords: REST, API, django
:Original Documentation: https://bitbucket.org/jespern/django-piston/wiki/Home

Changes in PBS Education Version
================================

In order to implement best practices in API development, we have incorporated the following new features

.. Resource definition subsystem
.. Pluggable envelopes
.. Form error feedback

Example of API building
=======================

Here we demonstrate how to use the new features along with the existing ones

We'll start by selecting a few models

models.py::

    from django.db import models
    
    class Author(models.Model):
        name = models.CharField(max_length=200)
        biography = models.CharField(required=False)
        url = models.URLField(verify_exists=False, required=False)
    
    
    class Book(models.Model):
        title = models.CharField(max_length=200)
        summary = models.CharField(max_length=2000, required=False)
        isbn10 = models.CharField(max_length=10)
        isbn13 = models.CharField(max_length=13)
        pages = models.IntegerField(required=False)
        language = models.CharField(max_length=200)
        authors = models.ManyToManyField(Author, related_name='books')
        time_created = models.DateTimeField(auto_now_add=True)


    class Publisher(models.Model):
        name = models.CharField(max_length=200)
        url = models.URLField(verify_exists=False, required=False)


    class Edition(models.Model):
        book = models.ForeignKey(Book, related_name='editions')
        publisher = models.ForeignKey(Book, related_name='editions')
        number = models.IntegerField()
        date_published = models.DateTimeField()


    class Award(models.Model):
        name = models.CharField(max_length=200) 
        book = models.ForeignKey(Book, related_name='awards')
        date_awarded = models.DateTimeField()


Introducing PistonView
======================

One of the major new features added to piston by PBS Education is the PistonView.

.. PistonView allows you to templatize definition of resources, detaching them completely from Models
.. It allows you to add arbitrary attributes to any of your resources
.. You can start with an object instance and use values yielded by it's class members to be attributes of the desired resource
.. If the object has a class member of type list/ tuple/ set of other objects (homogenous), you can assign other PistonViews to render them


views/piston.py::

    import datetime
    from piston.handler import PistonView, Field

    class EditionSummaryView(PistonView):
        fields = [
                'id',
                'number',
                'publisher.name',
                'date_published',
                ]


    class AwardSummaryView(PistonView):
        fields = [
                'id',
                'name',
                'date_awarded',
                ]


    class BookSummaryView(PistonView):
        fields = [
                'id',
                'title',
                'isbn10',
                Field('', lambda x: [y.name for y in x.authors.all()], destination='authors'),
                ]


    class BookDetailedView(PistonView):
        fields = [
                'id',
                'title',
                'isbn10',
                'isbn13',
                'language',
                'pages',
                Field('', lambda x: [y.name for y in x.authors.all()], destination='authors'),
                Field('', lambda x: [EditionSummaryView(y) for y in x.editions.all()], destination='editions'),
                Field('', lambda x: [AwardSummaryView(y) for y in x.awards.all()], destination='awards'),
                Field('', lambda x: datetime.datetime.now().strftime("%m/%d/%y"), destination='time_retrieved'),
                ]


Let's also write a PaginationView while we're at it.
It takes the django page object and some relevant information:: 

    from piston.handler import PistonView, Field

    class PaginationView(PistonView):
        fields = [
                Field('number', destination='page'),
                Field('paginator.num_pages', destination='pages'),
                Field('paginator.count', destination='count'),
                Field('paginator.per_page', destination='per_page'),
                Field('has_next'),
                Field('has_previous'),
                Field('start_index', destination='start'),
                Field('end_index', destination='end'),
                ]


Now let's write some Piston handlers.

handlers.py::

    from piston.handler import BaseHandler
    from piston.resource import PistonNotFoundException
    from myproject.utils.forms import PaginationForm


    class BooksHandler(BaseHandler):
        allowed_methods = ('GET', 'POST', 'PUT', 'DELETE',)

    def read(self, request, id=None):
        if id is None:
            return self.list(request)
        return BookDetailedView(self.get(request, id))

    def list(self, request):
        form = PaginationForm(request.GET)
        per_page, page_num = form.get_pagination_params()

        paginator = Paginator(Book.objects.all(), per_page)
        page = paginator.page(page_num)
        return {
            'pagination': PaginationView(page),
            'books': BookSummaryView([x for x in page.object_list]),
            }

    def get(self, request, id):
        try:
            book = Book.objects.get(id=id)
        except (ValidationError, Book.DoesNotExist):
            raise PistonNotFoundException('Error retrieving book with ID %s' % id)
        return book

    @login_required()
    def create(self, request, id=None):
        if id is not None:
            raise PistonNotFoundException('ID not expected when creating books')

        form = BookForm(request.data)
        if not form.is_valid():
            raise FormValidationError(form)

        book = form.save()

        return BookDetailedView(book)

    @login_required()
    def update(self, request, id):
        form = BookForm(request.data)
        if not form.is_valid():
            raise FormValidationError(form)

        book = form.save()

        return BookDetailedView(book)

    @login_required()
    def delete(self, request, id):
        book = self.get(request, id)
        book.delete()

        return rc.DELETED
