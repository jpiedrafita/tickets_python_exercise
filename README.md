Scenario
You've been asked to build a dashboard for your boss so that he can see the status of the various bug tickets that exist in your system. More information can be found in the "Instructions" section.

Additional Resources

You've been asked to build a dashboard for your boss so that he can see the status of the various bug tickets that exist in your system. Tickets are extremely simple and only have a few points of information:

    Name: Required.
    Status: Required, available options are:
        0 (Reported)
        1 (In Progress)
        2 (In Review)
        3 (Resolved)
    URL: Optional

With this information you'll need to do a few things:

    Render a listing view at /tickets
    Render an individual ticket's information at /tickets/:id
    Provide a JSON API for /api/tickets and /api/tickets/:id that returns the JSON serialized ticket information.

A coworker has provided the HTML templates and styles for you to use, but you'll need to add the Jinja portions so that the content can be rendered.

The PostgreSQL database that runs on a separate database server is accessible using the default port (5432) using the server's private IP address. These are the database details that you'll need:

    user - dashboard
    database - dashboard
    password - secure_password
    table - ticket
        id - (UUID, required, automatic) unique identifier
        name - (string, 100 character limit, required) name of ticket
        status - (integer, required, default 0) state of ticket (see information above)
        url - (string, 100 character limit, optional)

Useful Documentation and Packages

    Flask: https://flask.palletsprojects.com/en/1.0.x/
    Flask - Database Configuration: https://flask.palletsprojects.com/en/1.0.x/tutorial/database/
    Flask - Views https://flask.palletsprojects.com/en/1.0.x/tutorial/views/
    Flask - Templates https://flask.palletsprojects.com/en/1.0.x/tutorial/templates/
    Flask-SQLAlchemy https://flask-sqlalchemy.palletsprojects.com/en/2.x/quickstart/
    psycopg2 (as a database driver, likely not used directly) https://www.psycopg.org/

## Learning Objectives

Create project and virtualenv.

To get started, we're going to create a directory to hold onto our project. We'll call this tickets:

$ mkdir tickets
$ cd tickets

With this set up, we're going to need to create our Virtualenv and install some of the dependencies that we know we need:

$ pipenv --python=$(which python3.7) install flask

For the rest of our work, we'll want to make sure that we're using our active Virtualenv. Let's activate it now:

$ pipenv shell
(tickets) $

---

We're ready to create the initial layout of the application and set up our database configuration. To begin, we'll create an **init**.py script to generate our application. We'll take an approach very similar to the official Flask tutorial, setting up an application factory, starting with a file named **init**.py:

~/tickets/init.py

import os

from flask import Flask

def create_app(test_config=None):
app = Flask(**name**)
app.config.from_mapping(
SECRET_KEY=os.environ.get('SECRET_KEY', default='dev'),
)

    if test_config is None:
        app.config.from_pyfile('config.py', silent=True)
    else:
        app.config.from_mapping(test_config)

    return app

Now we can test the bare-bones application. We're going to change the port to 3000 because Linux Academy servers have that open by default:

Note: You'll want to do this in a separate terminal instance so that we can keep it running. It will auto reload the code as we make changes.

(tickets) $ export FLASK_ENV=development
(tickets) $ export FLASK_APP='.'
(tickets) $ flask run --host=0.0.0.0 --port=3000

- Serving Flask app "." (lazy loading)
- Environment: development
- Debug mode: on
- Running on http://0.0.0.0:3000/ (Press CTRL+C to quit)
- Restarting with stat
- Debugger is active!
- Debugger PIN: 112-739-965

Our next step will be to install a library for interacting with PostgreSQL and configuring our database connection. For this, we'll use psycopg2 and the Flask-SQLAlchemy plugin. Let's install these now:

(tickets) $ pipenv install psycopg2 Flask-SQLAlchemy
Installing psycopg2…
Adding psycopg2 to Pipfile's [packages]…
✔ Installation Succeeded
Installing Flask-SQLAlchemy…
Adding Flask-SQLAlchemy to Pipfile's [packages]…
✔ Installation Succeeded
Pipfile.lock (caf66b) out of date, updating to (662286)…
Locking [dev-packages] dependencies…
Locking [packages] dependencies…
✔ Success!
Updated Pipfile.lock (caf66b)!
Installing dependencies from Pipfile.lock (caf66b)…
▉▉▉▉▉▉▉▉▉▉▉▉▉▉▉▉▉▉▉▉▉▉▉▉▉▉▉▉▉▉▉▉ 9/9 — 00:00:02

We'll set up our database configuration with a new file called config.py:

~/tickets/config.py

import os

db_host = os.environ.get('DB_HOST', default='< DB_PRIVATE_IP >')
db_name = os.environ.get('DB_NAME', default='dashboard')
db_password = os.environ.get('DB_PASSWORD', default='secure_password')
db_port = os.environ.get('DB_PORT', default='5432')
db_user = os.environ.get('DB_USERNAME', default='dashboard')

SQLALCHEMY_DATABASE_URI = f"postgresql://{db_user}:{db_password}@{db_host}:{db_port}/{db_name}"

We're using environment variables and os.environ.get so that we can easily adjust these with environment variables. Next, let's create a file (models.py) to contain our database logic and classes wrapping database tables. We'll need to pull in the flask_sqlalchemy package to initialize a database object that won't actually connect to the application's database just yet:

~/tickets/models.py

from flask_sqlalchemy import SQLAlchemy

db = SQLAlchemy()

class Ticket(db.Model):
id = db.Column(db.Integer, primary_key=True)
name = db.Column(db.String(100), nullable=False)
status = db.Column(db.Integer, nullable=False)
url = db.Column(db.String(100), nullable=True)

    statuses_dict = {
        0: 'Reported',
        1: 'In Progress',
        2: 'In Review',
        3: 'Resolved',
    }

    def status_string(self):
        return self.statuses_dict[self.status]

Now we have a Ticket class that we can use to work with the information from our database and a db object that we can initialize with our application configuration. That initialization needs to happen within our create_app function in the **init**.py:

~/tickets/init.py

import os

from flask import Flask

def create_app(test_config=None):
app = Flask(**name**)
app.config.from_mapping(
SECRET_KEY=(os.getenv('SECRET_KEY') or 'dev'),
)

    if test_config is None:
        # Load configuration from config.py
        app.config.from_pyfile('config.py', silent=True)
    else:
        app.config.from_mapping(test_config)

    from .models import db
    db.init_app(app)

    return app

## Now we have a working application configuration that can communicate with our database.

For the HTML based views we're going to use some HTML templates that a co-worker created for us that have some comments on where to put the dynamic information. These files can be found within ~/templates and we also need to move over the styles from ~/static. Let's copy those directories into our application now:

(tickets) $ mv ~/templates .
(tickets) $ mv ~/static .

Before worrying about the templates themselves, let's create the request handler functions. These handlers will be defined as functions within our create_app function in the **init**.py file:

~/tickets/init.py

import os

from flask import Flask, redirect, render_template, url_for

def create_app(test_config=None):
app = Flask(**name**)
app.config.from_mapping(
SECRET_KEY=(os.getenv('SECRET_KEY') or 'dev'),
)

    if test_config is None:
        # Load configuration from config.py
        app.config.from_pyfile('config.py', silent=True)
    else:
        app.config.from_mapping(test_config)

    from .models import db
    db.init_app(app)

    @app.route('/')
    def index():
        return redirect(url_for('tickets'))

    @app.route('/tickets')
    def tickets():
        return render_template('tickets_index.html')

    @app.route('/tickets/<int:ticket_id>')
    def tickets_show(ticket_id):
        return render_template('tickets_show.html')

    return app

For now, we just want to make sure that our templates are being rendered out properly. If we reload the browser we should see them and if we visit the root URL we will be redirected to the /tickets URL. Our next step is to query our database for the tickets so that we can pass them to our templates. In the tickets function we'll fetch all of the items from the database to list out, and from the tickets_show function we'll fetch just one based on the ID that is passed into the URL. Let's add this logic now:

~/tickets/init.py

import os

from flask import Flask, abort, redirect, render_template, url_for

def create_app(test_config=None):
app = Flask(**name**)
app.config.from_mapping(
SECRET_KEY=(os.getenv('SECRET_KEY') or 'dev'),
)

    if test_config is None:
        # Load configuration from config.py
        app.config.from_pyfile('config.py', silent=True)
    else:
        app.config.from_mapping(test_config)

    from .models import db, Ticket
    db.init_app(app)

    from sqlalchemy.orm import exc

    @app.errorhandler(404)
    def page_not_found(e):
        return render_template('404.html'), 404

    @app.route('/')
    def index():
        return redirect(url_for('tickets'))

    @app.route('/tickets')
    def tickets():
        tickets = Ticket.query.all()
        return render_template('tickets_index.html', tickets=tickets)

    @app.route('/tickets/<int:ticket_id>')
    def tickets_show(ticket_id):
        try:
            ticket = Ticket.query.filter_by(id=ticket_id).one()
            return render_template('tickets_show.html', ticket=ticket)
        except exc.NoResultFound:
            abort(404)

    return app

Notice that we had to add a little error handling to the tickets_show view function because it's possible for someone to pass in an ID that doesn't exist. In this case, we're going to use the one function that will throw a sqlalchemy.orm.exc.NoResultFound error if there is not a matching row. We catch that error in our except and then Flask will automatically run the function that we've decorated with @app.errorhandler(404).

The last thing that we need to do is utilize the tickets and ticket variables within templates/tickets_index.html and templates/tickets_show.html respectively. Here are the lines that we adjusted in each:

~/tickets/templates/tickets_index.html

  <!-- Contents above this comment were omitted -->
  <!-- EXAMPLE ROW, substitue the real information from the tickets in the database -->

{% for ticket in tickets %}

<tr>
<th>{{ticket.id}}</th>
<td>{{ticket.name}}</td>
<td>{{ticket.status_string()}}</td>
<td>
<a href="{{ticket.url}}">{{ticket.url}}</a>
</td>
<td>
<a href="{{url_for('tickets_show', ticket_id=ticket.id)}}">Details</a>
</td>
</tr>
{% endfor %}

  <!-- Contens below this comment were omitted -->

~/tickets/templates/tickets_show.html

{% extends "layout.html" %}
{% block title %}Ticket - {{ticket.name}}{% endblock %}

{% block body %}

<div class="content">
  <p><strong>Name:</strong> {{ticket.name}}</p>
  <p><strong>Status:</strong> {{ticket.status_string()}}</p>
  <p><strong>URL:</strong> <a href="{{ticket.url}}" target="_blank">{{ticket.url}}</a></p>
</div>
{% endblock %}

## Note: It's important that we call the status_string method using parenthesis, otherwise we'll render a message about there being a method bound to the ticket variable on the page.

We had to do quite a bit to render our HTML views, but that lays the groundwork for us to pretty easily create our API view functions. Let's utilize the jsonify helper that Flask provides to create our final two view functions with **init**.py.

~/tickets/init.py

import os

from flask import Flask, abort, redirect, render_template, url_for, jsonify

def create_app(test_config=None):
app = Flask(**name**)
app.config.from_mapping(
SECRET_KEY=(os.getenv('SECRET_KEY') or 'dev'),
)

    if test_config is None:
        # Load configuration from config.py
        app.config.from_pyfile('config.py', silent=True)
    else:
        app.config.from_mapping(test_config)

    from .models import db, Ticket
    db.init_app(app)

    from sqlalchemy.orm import exc

    @app.errorhandler(404)
    def page_not_found(e):
        return render_template('404.html'), 404

    @app.route('/')
    def index():
        return redirect(url_for('tickets'))

    @app.route('/tickets')
    def tickets():
        tickets = Ticket.query.all()
        return render_template('tickets_index.html', tickets=tickets)

    @app.route('/tickets/<int:ticket_id>')
    def tickets_show(ticket_id):
        try:
            ticket = Ticket.query.filter_by(id=ticket_id).one()
            return render_template('tickets_show.html', ticket=ticket)
        except exc.NoResultFound:
            abort(404)

    @app.route('/api/tickets')
    def api_tickets():
        tickets = Ticket.query.all()
        return jsonify(tickets)

    @app.route('/api/tickets/<int:ticket_id>')
    def api_tickets_show(ticket_id):
        try:
            ticket = Ticket.query.filter_by(id=ticket_id).one()
            return jsonify(ticket)
        except exc.NoResultFound:
            return jsonify({'error': 'Ticket not found'}), 404

    return app

This looks great, but unfortunately, we receive an error when we try to access the /api/tickets URL in the browser because Object of type Ticket is not JSON serializable. This is something that we'll need to fix within our model by adding a to_json method that we can call before we pass our object to jsonify:

~/tickets/models.py

from flask_sqlalchemy import SQLAlchemy

db = SQLAlchemy()

class Ticket(db.Model):
id = db.Column(db.Integer, primary_key=True)
name = db.Column(db.String(100), nullable=False)
status = db.Column(db.Integer, nullable=False)
url = db.Column(db.String(100), nullable=True)

    statuses_dict = {
        0: 'Reported',
        1: 'In Progress',
        2: 'In Review',
        3: 'Resolved',
    }

    def status_string(self):
        return self.statuses_dict[self.status]

    def to_json(self):
        """
        Return the JSON serializable format
        """
        return {
            'id': self.id,
            'name': self.name,
            'status': self.status_string(),
            'url': self.url
        }

Now we can utilize this function in our view functions:

~/tickets/init.py

    # Extra code omitted

    @app.route('/api/tickets')
    def api_tickets():
        tickets = Ticket.query.all()
        return jsonify([ticket.to_json() for ticket in tickets])

    @app.route('/api/tickets/<int:ticket_id>')
    def api_tickets_show(ticket_id):
        try:
            ticket = Ticket.query.filter_by(id=ticket_id).one()
            return jsonify(ticket.to_json())
        except exc.NoResultFound:
            return jsonify({'error': 'Ticket not found'}), 404

    # Extra code omitted
