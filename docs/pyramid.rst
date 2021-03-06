Pyramid Tutorial
================

.. note::
    The files created in this tutorial can be downloaded as a `.zip file
    <https://github.com/toscawidgets/tw2.core-docs-pyramid/zipball/master>`_,
    a `.tar file
    <https://github.com/toscawidgets/tw2.core-docs-pyramid/tarball/master>`_,
    or can be cloned from a `github repository
    <https://github.com/toscawidgets/tw2.core-docs-pyramid>`_.


Getting Set Up
--------------

First, we'll create a python ``virtualenv`` and install ``pyramid`` into it::

    $ mkvirtualenv tw2-and-pyramid
    $ pip install pyramid
    $ pcreate -t alchemy myapp
    $ cd myapp

Open ``setup.py`` and add the following to the ``requires = [...]`` entry::

    requires = [

        ...

        "tw2.core",
        "tw2.forms",
        "tw2.dynforms",
        "tw2.sqla",
        "tw2.jqplugins.jqgrid",
        ],

Once that's done.  Install your dependencies and initialize your
database by running::

    $ python setup.py develop
    $ initialize_myapp_db development.ini

Enabling ToscaWidgets2
----------------------

Easy!  Make a couple edits to ``development.ini``.

If you look at the top, there should be a section heading labeled
``[app:main]``.  Change that to ``[app:myapp]``.

Next, just above the ``[server:main]`` section, add a new section that looks
like::

    [pipeline:main]
    pipeline =
        tw2
        myapp

We'll add another section where pyramid will take configuration values for the
tw2.core middleware itself.  Add it just below the ``[pipeline:main]`` section::

    [filter:tw2]
    use = egg:tw2.core#middleware

We'll add more configuration values later.  For now,
check that this worked by running the following and visiting
http://localhost:6543::

    $ pserve --reload development.ini

Building a Form
---------------
We'll create a movie database as in the :doc:`standalone` example.  We'll need
to create the widget, expose it in a pyramid view, and render it in a template.

First, create a new file ``myapp/widgets.py`` with the contents::

    import tw2.core
    import tw2.forms


    class MovieForm(tw2.forms.FormPage):
        title = 'Movie'

        class child(tw2.forms.TableForm):
            id = tw2.forms.HiddenField()
            title = tw2.forms.TextField(validator=tw2.core.Required)
            director = tw2.forms.TextField()
            genres = tw2.forms.CheckBoxList(options=[
                'Action', 'Comedy', 'Romance', 'Sci-fi'])

            class cast(tw2.forms.GridLayout):
                extra_reps = 5
                character = tw2.forms.TextField()
                actor = tw2.forms.TextField()

Second, modify ``myapp/views.py`` and add a new view callable like so::

    import myapp.widgets
    @view_config(route_name='movie', renderer='templates/widget.pt')
    def view_widget(request):
        return {'widget': myapp.widgets.MovieForm}

Thirdly, add a new template ``myapp/templates/widget.pt`` (which is a `chameleon
<http://pypi.python.org/pypi/Chameleon>`_ template) with the following
contents::

    <!DOCTYPE html PUBLIC "-//W3C//DTD XHTML 1.0 Strict//EN" "http://www.w3.org/TR/xhtml1/DTD/xhtml1-strict.dtd">
    <html xmlns="http://www.w3.org/1999/xhtml" xml:lang="en" xmlns:tal="http://xml.zope.org/namespaces/tal">
    <head>
      <title>The Pyramid Web Application Development Framework</title>
      <meta http-equiv="Content-Type" content="text/html;charset=UTF-8"/>
      <meta name="keywords" content="python web application" />
      <meta name="description" content="pyramid web application" />
      <link rel="shortcut icon" href="${request.static_url('myapp:static/favicon.ico')}" />
    </head>
    <body>
      <div id="wrap">
        <div tal:content="structure widget.display()"></div>
      </div>
      <div id="footer">
        <div class="footer">&copy; Copyright 2008-2011, Agendaless Consulting.</div>
      </div>
    </body>
    </html>

Having added a new view
callable to ``myapp/views.py``, added the new template
``myapp/templates/widget.pt``, and having added the widget definition
itself to ``myapp/widgets.py``, all that's left is to wire it all together.
Edit your applications configuration in ``myapp/__init__.py`` and add the
view to the application registry with the following call::

    config.add_route('movie', '/movie')

With those four file edits in place, you should be able to restart the
application with ``pserve development.ini`` (there is a ``--reload``
option for convenience) and point your browser
at http://localhost:6543/movie.

You should see the form, but it doesn't look very appealing.  To try to
improve this, lets add some CSS.  We'll start with something simple;
create ``myapp/static/myapp.css`` with the following::

    th {
        vertical-align: top;
        text-align: left;
        font-weight: normal;
    }

    ul {
        list-style-type: none;
    }

    .required th {
        font-weight: bold;
    }

Notice the use of the "required" class. TableForm applies this to rows that
contain a field that is required.

Before TableForm will inject ``myapp.css`` into the page, we'll have to add
it to the list of resources. Add the following to the top of the `MovieForm`
class definition in ``myapp/widgets.py`` just above the line
``title = 'Movie'``::

    resources = [tw2.core.CSSLink(link='static/myapp.css')]

Restart ``pserve`` and browse to http://localhost:6543/movie
to see the new css in action.

Connecting to a Database
------------------------

The next step is to save movies to a database.  To do this, we'll use only
`SQLAlchemy <http://www.sqlalchemy.org/>`_ just like in the :doc:`turbogears`
tutorial (and not `elixir <http://elixir.ematia.de/trac/wiki>`_ as in
the :doc:`standalone` tutorial).  SQLAlchemy is built into our pyramid app
from the get-go by way of us using the alchemy paster template.
Edit ``development.ini`` and modify the ``[filter:tw2.core]`` section like
so::

    [filter:tw2.core]
    use = egg:tw2.core#middleware
    controller_prefix = /tw2_controllers/
    serve_controllers = True

Next, edit ``myapp/models.py`` with the following changes.  Add this set of
imports to the top::

    from sqlalchemy import Table, Unicode
    from sqlalchemy import ForeignKey
    from sqlalchemy.orm import relation
    from sqlalchemy.orm import backref

Just above the definition of ``class MyModel(Base):`` add::

    Base.query = DBSession.query_property()

    movie_genre_table = Table('movie_genre', Base.metadata,
        Column('movie_id', Integer, ForeignKey('movies.id',
            onupdate="CASCADE", ondelete="CASCADE"), primary_key=True),
        Column('genre_id', Integer, ForeignKey('genres.id',
            onupdate="CASCADE", ondelete="CASCADE"), primary_key=True)
    )

    class Movie(Base):
        __tablename__ = 'movies'
        id = Column(Integer, primary_key=True)
        title = Column(Unicode(255))
        director = Column(Unicode(255))

    class Genre(Base):
        __tablename__ = 'genres'
        id = Column(Integer, primary_key=True)
        name = Column(Unicode(255))
        movies = relation('Movie', secondary=movie_genre_table, backref='genres')
        def __unicode__(self):
            return unicode(self.name)

    class Cast(Base):
        __tablename__ = 'casts'
        id = Column(Integer, primary_key=True)
        movie_id = Column(Integer, ForeignKey(Movie.id))
        movie = relation(Movie, backref=backref('cast'))
        character = Column(Unicode(255))
        actor = Column(Unicode(255))

Go off now to ``myapp/scripts/initializedb.py`` where we have to make a couple
edits.  First, add ``Genre`` to the list of things you're importing from
``..models``.  Second, modify the call to ``get_appsettings(config_uri)`` to
look like ``get_appsettings(config_uri, name="myapp")``.  Thirdly, inside the
``with transaction.manager:`` line, add the following::

    for name in ['Action', 'Comedy', 'Romance', 'Sci-fi']:
        DBSession.add(Genre(name=name))

Now done with ``myapp/models.py`` and ``myapp/scripts/initializedb.py``, edit
``myapp/views.py`` and replace the definition of ``def view_widget(request):`` with::

    import tw2.core
    import myapp.widgets
    @view_config(route_name='movie', renderer='templates/widget.pt')
    def view_widget(request):
        widget = myapp.widgets.MovieForm.req()
        widget.fetch_data(request)
        tw2.core.register_controller(widget, 'movie_submit')
        return {'widget': widget}

Lastly, edit ``myapp/widgets.py`` and add::

    import tw2.sqla
    import myapp.models

Change ``class MovieForm(tw2.forms.FormPage):`` to::

    class MovieForm(tw2.sqla.DbFormPage):
        entity = myapp.models.Movie

To the body of ``class child(tw2.forms.TableForm):`` add::

    action = '/tw2_controllers/movie_submit'

And the last for the `MovieForm`, change ``genres = tw2.forms.CheckBoxList( ... )`` to::

    genres = tw2.sqla.DbCheckBoxList(entity=myapp.models.Genre)

Now, in your command prompt run::

    rm myapp.sqlite
    initialize_myapp_db development.ini
    pserve development.ini

This will recreate and initialize your database in a sqlite DB.

We're almost done, but not quite.  Nonetheless, this is a good point to restart
your app and test to see if any mistakes have cropped up.  Restart `pserve`
and visit http://localhost:6543/movie.  Submit your first entry.  It
should give you an `Error 404`, but don't worry.  Point your browser now to
http://localhost:6543/movie?id=1 and you should see the same
movie entry that you just submitted.

Great -- we can write to the database and read back an entry, now how about
a list of entries?

Add a whole new class to ``myapp/widgets.py``::

    class MovieList(tw2.sqla.DbListPage):
        entity = myapp.models.Movie
        title = 'Movies'
        newlink = tw2.forms.LinkField(link='/movie', text='New', value=1)

        class child(tw2.forms.GridLayout):
            title = tw2.forms.LabelField()
            id = tw2.forms.LinkField(link='/movie?id=$', text='Edit', label=None)

In ``myapp/widgets.py`` also add the following line just inside the definition
of ``MovieForm``::

    redirect = '/list'

Add another route to your top level Pyramid configuration by editing
``myapp/__init__.py``::

    config.add_route('list', '/list')

And add the corresponding view to ``myapp/views.py``::

    @view_config(route_name='list', renderer='templates/widget.pt')
    def view_list(request):
        widget = myapp.widgets.MovieList.req()
        widget.fetch_data(request)
        return {'widget': widget}

Now restart `paster` and browse to http://localhost:6543/list

Getting Fancy
-------------

We could also make things dynamic by editing ``myapp/widgets.py`` and adding at the top::

    import tw2.dynforms

replacing ``class child(tw2.forms.TableForm):`` with::

    class child(tw2.dynforms.CustomisedTableForm):

and replacing::

    class cast(tw2.forms.GridLayout):
        extra_reps = 5

with::

    class cast(tw2.dynforms.GrowingGridLayout):

Getting Fancier
---------------

There are a lot of `non-core` TW2 widget libraries out there, and just to give
you a taste, we'll use one to add one more view to our Movie app.

Edit ``myapp/widgets.py`` and add the following to the top::

    import tw2.jqplugins.jqgrid

Add the following class definition to the same file::

    class GridWidget(tw2.jqplugins.jqgrid.SQLAjqGridWidget):
        id = 'grid_widget'
        entity = myapp.models.Movie
        excluded_columns = ['id']
        prmFilter = {'stringResult': True, 'searchOnEnter': False}
        pager_options = { "search" : True, "refresh" : True, "add" : False, }
        options = {
            'url': '/tw2_controllers/db_jqgrid/',
            'caption': 'A grid!',
            'rowNum':15,
            'rowList':[15,30,50],
            'viewrecords':True,
            'imgpath': 'scripts/jqGrid/themes/green/images',
            'width': 900,
            'height': 'auto',
        }

Add the following to your view configuration in ``myapp/__init__.py``::

    config.add_route('grid', '/grid')


Add that view to ``myapp/views.py`` itself::

    @view_config(route_name='grid', renderer='templates/widget.pt')
    def view_grid(request):
        widget = myapp.widgets.GridWidget.req()
        tw2.core.register_controller(widget, 'db_jqgrid')
        return {'widget': widget}

Redirect your browser to http://localhost:6543/grid and you should
see the sortable, searchable jQuery grid.
