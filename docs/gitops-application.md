# GitOps Http Application Sample

## HTTP Application 
This Gitops sample provides a standard HTTP component consisting of a deployment, service and route. 

The following day 2 edit/update operations supported:
    set/get image - updates the image for this component 
    set/get replicas

from flask import Flask, render_template, flash, session, request, redirect, url_for, g, current_app
from flask_sqlalchemy import SQLAlchemy
import flask_saml
import flask_principal
from forms import MyForm, SearchForm, EditForm
from datetime import date
from models import db, Product, ProductType, ProductTypeMap, ProductPortfolios, ProductPortfolioMap, ProductNotes, ProductReferences, ProductAlias, ProductMktLife, ProductPartners, Partner, ProductComponents, ProductLog
from datetime import datetime
from routes.view_routes import view_routes
from routes.add_routes import add_routes
from routes.edit_routes import edit_routes
from user_details import user_details
import os
from sqlalchemy.orm import scoped_session, sessionmaker
from sqlalchemy import create_engine

def determine_greeting():
    current_hour = datetime.now().hour
    if 5 <= current_hour < 12:
        return "Good Morning"
    elif 12 <= current_hour < 18:
        return "Good Afternoon"
    else:
        return "Good Evening"

def create_app(test_config=None):
    app = Flask(__name__)

    principals = flask_principal.Principal(app)

    app.config['SECRET_KEY'] = 'Dev'
    app.config['SAML_METADATA_URL'] = os.environ["SAML_METADATA_URL"]
    app.config['SAML_DEFAULT_REDIRECT'] = '/'

    # Config settings taken from environment variables
    db_driver = os.environ["DBDRIVER"]
    pg_user = os.environ["PGUSER"]
    pg_pass = os.environ["PGPASS"]
    pg_host = os.environ["PGHOST"]
    pg_port = os.environ["PGPORT"]
    pg_db = os.environ["PGDB"]

    # Database URI with SSL configuration
    app.config['SQLALCHEMY_DATABASE_URI'] = f"{db_driver}://{pg_user}:{pg_pass}@{pg_host}:{pg_port}/{pg_db}"
    if "REDSHIFT_SSL" in os.environ and os.environ["REDSHIFT_SSL"] == "True":
        app.config['SQLALCHEMY_ENGINE_OPTIONS'] = {
            'connect_args': {
                'sslmode': 'verify-ca',
            }
        }
    
    # Introduce connection pooling with create_engine
    engine = create_engine(app.config['SQLALCHEMY_DATABASE_URI'], pool_size=20,  # Maximum of 20 concurrent connections
                            max_overflow=5,  # Allow 5 additional connections during peak usage
                            pool_recycle=3600)  # Recycle unused connections after 1 hour

    # Initialize the database (optional, for improved clarity)
    db.init_app(app)

    # Configure scoped session
    with app.app_context():
      Session = scoped_session(sessionmaker(bind=db.engine))
      app.config['Session'] = Session

    flask_saml.FlaskSAML(app)

    # Actions to perform when a user logs in
    @flask_saml.saml_authenticated.connect_via(app)
    def on_saml_authenticated(sender, subject, attributes, auth):
        # Store the username in the session
        session['username'] = subject
        
        # We have a logged in user, inform Flask-Principal
        flask_principal.identity_changed.send(
            current_app._get_current_object(), identity=get_identity())
        
    # Actions to perform when a user logs out
    @flask_saml.saml_log_out.connect_via(app)
    def on_saml_logout(sender):
        # Let Flask-Principal know the user is gone
        flask_principal.identity_changed.send(
            current_app._get_current_object(), identity=get_identity())

    # Set the user identity for the application
    @principals.identity_loader
    def get_identity():
        if 'saml' in session:
            return flask_principal.Identity(session['saml']['subject'])
        else:
            return flask_principal.AnonymousIdentity()

    # Actions to perform after setting the user identity
    @flask_principal.identity_loaded.connect_via(app)
    def on_identity_loaded(sender, identity):
        # Define the permission need for all users
        identity.provides.add(
            flask_principal.TypeNeed("all")
        )
        if not isinstance(identity, flask_principal.AnonymousIdentity):
            # Define the permission need for the user's roles
            if "Role" in session['saml']['attributes'].keys():
                for role in session['saml']['attributes']['Role']:
                    identity.provides.add(
                        flask_principal.RoleNeed(role)
                    )
            # Define the permission need for the user's groups
            if "group" in session['saml']['attributes'].keys():
                for group in session['saml']['attributes']['group']:
                    identity.provides.add(
                        flask_principal.Need('group', group)
                    )
            # Define the permission need for an authenticated user
            identity.provides.add(
                flask_principal.TypeNeed("authenticated")
            )
        else:
            # Define the permission need for an anonymous user
            identity.provides.add(
                flask_principal.TypeNeed("anonymous")
            )

    # Set the variable for the username, which we use in our masthead for an
    # authenticated user.
    @app.context_processor
    def get_current_user():
        needs={}
        for need in g.identity.provides:
            if need.method not in needs:
                needs[need.method]=[]
            needs[need.method].append(need.value)
        return dict(user=g.identity.id, needs=needs)

    # If a user does not have access to a particular resource, send them
    # to our 403 page.
    @app.errorhandler(flask_principal.PermissionDenied)
    def handle_permission_denied(error):
        return render_template('403.html'), 403

    # handle 404 errors
    @app.errorhandler(404)
    def page_not_found(error):
        return render_template('404.html'), 404

    @app.route("/")
    def home():
        greeting = determine_greeting()
        if 'saml' in session and 'firstName' in session['saml']['attributes']:
            first_name = session['saml']['attributes']['firstName'][0]
        else:
            first_name = "anonymous user"
        return render_template("index.html", greeting=greeting, first_name=first_name)

    # The route definition to view a product.
    app.register_blueprint(view_routes)

    # The route definition to add a product.
    app.register_blueprint(add_routes)

    # The route definition to edit a product.
    app.register_blueprint(edit_routes)

    app.register_blueprint(user_details)

    # Run the application.
    return app

if __name__ == '__main__':
    app = create_app()
    app.run(debug=True)
