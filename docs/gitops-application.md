# GitOps Http Application Sample

## HTTP Application 
This Gitops sample provides a standard HTTP component consisting of a deployment, service and route. 

The following day 2 edit/update operations supported:
    set/get image - updates the image for this component 
    set/get replicas

from flask import Blueprint, render_template, request, redirect, url_for, flash, session as flask_session, current_app, abort
from datetime import datetime
from sqlalchemy import func, or_, text
from forms import MyForm, SearchForm, EditForm
from models import db, Product, ProductType, ProductTypeMap, ProductPortfolios, ProductPortfolioMap, ProductNotes, ProductReferences, ProductAlias, ProductMktLife, ProductPartners, Partner, ProductComponents, ProductLog
from sqlalchemy.exc import OperationalError, ProgrammingError
import time
import permissions
import logging
import os

# Configure logging
logging.basicConfig(level=logging.DEBUG)
logger = logging.getLogger(__name__)

# Environment variables for retry logic
RETRY_ATTEMPTS = int(os.getenv('RETRY_ATTEMPTS', 5))
RETRY_WAIT_TIME = int(os.getenv('RETRY_WAIT_TIME', 2))

SERIALIZABLE_ERROR_CODE = 'XX000'

edit_routes = Blueprint('edit_routes', __name__)

def retry_db_operation(operation, attempts=RETRY_ATTEMPTS, wait_time=RETRY_WAIT_TIME):
    for attempt in range(attempts):
        try:
            return operation()
        except (OperationalError, ProgrammingError) as e:
            if isinstance(e, ProgrammingError):
                if hasattr(e.orig, 'pgcode') and e.orig.pgcode != SERIALIZABLE_ERROR_CODE:
                    raise
            logger.error(f"Attempt {attempt + 1} failed with error: {e}")
            time.sleep(wait_time * (2 ** attempt))  # Exponential backoff
    logger.error(f"All attempts to perform the operation failed after {attempts} attempts.")
    raise Exception("Database operation failed after multiple attempts.")

@edit_routes.route('/opl/search-to-edit-products', methods=['GET', 'POST'])
@permissions.opl_editor_permission.require()
def edit_products():
    session = current_app.config['Session']()
    try:
        # Initialize the search form
        form = SearchForm()

        # Define the base query for products, including the portfolio name
        products_query = retry_db_operation(lambda: session.query(
            Product.product_id,
            Product.product_name,
            Product.product_status,
            Product.last_updated,
            ProductPortfolios.category_name,
            ProductType.product_type
        ).outerjoin(ProductPortfolioMap, Product.product_id == ProductPortfolioMap.product_id)
          .outerjoin(ProductPortfolios, ProductPortfolioMap.category_id == ProductPortfolios.category_id)
          .join(ProductTypeMap, ProductTypeMap.product_id == Product.product_id)
          .join(ProductType, ProductType.type_id == ProductTypeMap.type_id)
          .outerjoin(ProductAlias, ProductAlias.product_id == Product.product_id)
          .order_by(Product.product_name))

        if form.validate_on_submit():
            if form.product_name.data:
                search_term = f"%{form.product_name.data}%"
                products_query = products_query.filter(
                    or_(
                        Product.product_name.ilike(search_term),
                        ProductAlias.alias_name.ilike(search_term)
                    )
                )
            if form.product_status.data and form.product_status.data != 'Select':
                products_query = products_query.filter(Product.product_status == form.product_status.data)
            if form.product_type.data and form.product_type.data:
                type_id = form.product_type.data
                if isinstance(type_id, list):
                    type_id = type_id[0]
                products_query = products_query.filter(ProductType.type_id == type_id)

        # Get the list of products based on the filtered and now sorted query
        products = retry_db_operation(lambda: products_query.all())

        # Process the results in Python
        product_dict = {}
        for product in products:
            product_id = product.product_id
            if product_id not in product_dict:
                product_dict[product_id] = {
                    'product_id': product.product_id,
                    'product_name': product.product_name,
                    'product_status': product.product_status,
                    'last_updated': product.last_updated,
                    'portfolio_names': set(),
                    'product_types': set()
                }
            if product.category_name:
                product_dict[product_id]['portfolio_names'].add(product.category_name)
            if product.product_type:
                product_dict[product_id]['product_types'].add(product.product_type)

        # Convert sets to sorted lists
        products_with_portfolios = [
            {
                'product_id': prod['product_id'],
                'product_name': prod['product_name'],
                'product_status': prod['product_status'],
                'last_updated': prod['last_updated'],
                'portfolio_names': sorted(prod['portfolio_names']),
                'product_types': sorted(prod['product_types'])
            }
            for prod in product_dict.values()
        ]

        # Check if a specific product is selected to display detailed information
        selected_product_id = request.args.get('product_id')
        selected_product = retry_db_operation(lambda: session.query(Product).get(selected_product_id)) if selected_product_id else None

        return render_template('opl/edit_search.html', form=form, products=products, selected_product=selected_product, products_with_portfolios=products_with_portfolios)
    finally:
        session.close()

# Flask route for resetting the form
@edit_routes.route('/reset-edit-search-form', methods=['GET'])
def reset_search_form():
    form = SearchForm()  # Creates a new form instance with default values
    return render_template('opl/edit_search.html', form=form)

@edit_routes.route('/opl/edit-product/<string:product_id>', methods=['GET', 'POST'])
@permissions.opl_editor_permission.require()
def edit_product_details(product_id):
    session = current_app.config['Session']()
    try:
        logger.debug(f"Route accessed with method: {request.method}")
        
        # Manual get_or_404 replacement
        product = retry_db_operation(lambda: session.query(Product).get(product_id))
        if product is None:
            abort(404)

        product_notes = retry_db_operation(lambda: session.query(ProductNotes).filter_by(product_id=product_id).first())
        form = EditForm(obj=product)
        success_message = None
        show_form = True

        # Populate the form with existing data
        if request.method == 'GET':
            form.product_note.data = product_notes.product_note if product_notes else ""

        # Fetch the associated product types
        product_types_map = retry_db_operation(lambda: session.query(ProductTypeMap).filter_by(product_id=product_id).all())
        all_product_types = retry_db_operation(lambda: session.query(ProductType).all())
        form.product_type.choices = [(ptype.type_id, ptype.product_type) for ptype in all_product_types]
        form.product_type.data = [ptype_map.type_id for ptype_map in product_types_map]

        # Fetch the associated product portfolios
        product_portfolios_map = retry_db_operation(lambda: session.query(ProductPortfolioMap).filter_by(product_id=product_id).all())
        all_product_portfolios = retry_db_operation(lambda: session.query(ProductPortfolios).all())
        form.product_portfolio.choices = [(portfolio.category_id, portfolio.category_name) for portfolio in all_product_portfolios]
        form.product_portfolio.data = [pportfolio_map.category_id for pportfolio_map in product_portfolios_map]

        # Fetch existing product references
        logger.info(f"Fetching references for product ID: {product.product_id}")
        existing_references = retry_db_operation(lambda: session.query(ProductReferences).filter_by(product_id=product.product_id).all())
        logger.info(f"Existing references fetched: {existing_references}")
        reference_forms = [EditForm(obj=reference) for reference in existing_references]


        # Fetch existing product aliases
        existing_aliases = retry_db_operation(lambda: session.query(ProductAlias).filter_by(product_id=product_id).all())

        product_mkt_life = retry_db_operation(lambda: session.query(ProductMktLife).get(product_id))

        if request.method == 'GET':
            if product_mkt_life:
                form.product_release.data = product_mkt_life.product_release
                form.product_release_detail.data = product_mkt_life.product_release_detail
                form.product_release_link.data = product_mkt_life.product_release_link
                form.product_eol.data = product_mkt_life.product_eol
                form.product_eol_detail.data = product_mkt_life.product_eol_detail
                form.product_eol_link.data = product_mkt_life.product_eol_link
            else:
                form.product_release.data = ""
                form.product_release_detail.data = ""
                form.product_release_link.data = ""
                form.product_eol.data = ""
                form.product_eol_detail.data = ""
                form.product_eol_link.data = ""

        # Fetch the associated product partners
        existing_product_partners = retry_db_operation(lambda: session.query(ProductPartners).filter_by(product_id=product_id).all())
        all_partners = retry_db_operation(lambda: session.query(Partner).all())
        form.partner.choices = [(partner.partner_id, partner.partner_name) for partner in all_partners]
        form.partner.data = [ppartner.partner_id for ppartner in existing_product_partners]

        # Fetch existing product components
        existing_components = retry_db_operation(lambda: session.query(ProductComponents).filter_by(component_id=product_id).all())

        form.product_id.choices = [('', 'Select')] + [(str(prod.product_id), prod.product_name) for prod in retry_db_operation(lambda: session.query(Product).order_by(Product.product_name).all())]

        sql_query = text("SELECT * FROM brand_opl.product_log WHERE product_id = :product_id")
        existing_logs = retry_db_operation(lambda: session.execute(sql_query, {"product_id": product_id}).fetchall())

        if request.method == 'POST':
            logger.debug("Form data received: %s", request.form)
            if form.validate_on_submit() or 'submit' in request.form:
                try:
                    logger.debug("Starting session transaction")
                    form.populate_obj(product)
                    product.last_updated = datetime.now()

                    if not product_notes:
                        product_notes = ProductNotes(product_id=product_id, product_note=form.product_note.data)
                        retry_db_operation(lambda: session.add(product_notes))
                    else:
                        product_notes.product_note = form.product_note.data

                    existing_product_type_ids = [ptype_map.type_id for ptype_map in product_types_map]
                    selected_product_type_ids = request.form.getlist('product_type')

                    for product_type_id in selected_product_type_ids:
                        if product_type_id not in existing_product_type_ids:
                            new_product_type_map = ProductTypeMap(product_id=product.product_id, type_id=product_type_id)
                            retry_db_operation(lambda: session.add(new_product_type_map))

                    for ptype_map in product_types_map:
                        if ptype_map.type_id not in selected_product_type_ids:
                            retry_db_operation(lambda: session.delete(ptype_map))

                    existing_product_portfolio_ids = [pportfolio_map.category_id for pportfolio_map in product_portfolios_map]
                    selected_product_portfolio_ids = request.form.getlist('product_portfolio')

                    for product_portfolio_id in selected_product_portfolio_ids:
                        if product_portfolio_id not in existing_product_portfolio_ids:
                            new_product_portfolio_map = ProductPortfolioMap(product_id=product.product_id, category_id=product_portfolio_id)
                            retry_db_operation(lambda: session.add(new_product_portfolio_map))

                    for pportfolio_map in product_portfolios_map:
                        if pportfolio_map.category_id not in selected_product_portfolio_ids:
                            retry_db_operation(lambda: session.delete(pportfolio_map))

                    # Collect new references from the form
                    product_references_data = []
                    for i in range(len(request.form.getlist('product_link'))):
                        product_link = request.form.getlist('product_link')[i]
                        link_description = request.form.getlist('link_description')[i]
                        if product_link and link_description:
                            product_references_data.append({'product_link': product_link, 'link_description': link_description})

                    # Update or add new references
                    existing_references_dict = {ref.product_link: ref for ref in existing_references}
                    for reference_data in product_references_data:
                        product_link = reference_data['product_link']
                        if product_link in existing_references_dict:
                            existing_reference = existing_references_dict[product_link]
                            if existing_reference.link_description != reference_data['link_description']:
                                existing_reference.link_description = reference_data['link_description']
                                retry_db_operation(lambda: session.add(existing_reference))
                            # Remove the processed reference from the existing dictionary
                            del existing_references_dict[product_link]
                        else:
                            new_reference = ProductReferences(
                                product_id=product.product_id,
                                product_link=product_link,
                                link_description=reference_data['link_description']
                            )
                            retry_db_operation(lambda: session.add(new_reference))

                    # Delete references that were not included in the form
                    for reference in existing_references_dict.values():
                        retry_db_operation(lambda: session.delete(reference))

                    for alias in existing_aliases:
                        alias_name = request.form.get(f'alias_name_{alias.alias_id}')
                        alias_type = request.form.get(f'alias_type_{alias.alias_id}')
                        alias_approved = f'alias_approved_{alias.alias_id}' in request.form
                        previous_name = f'previous_name_{alias.alias_id}' in request.form
                        tech_docs = f'tech_docs_{alias.alias_id}' in request.form
                        tech_docs_cli = f'tech_docs_cli_{alias.alias_id}' in request.form
                        alias_notes = request.form.get(f'alias_notes_{alias.alias_id}')

                        if alias_name:
                            alias.alias_name = alias_name
                            alias.alias_type = alias_type
                            alias.alias_approved = alias_approved
                            alias.previous_name = previous_name
                            alias.tech_docs = tech_docs
                            alias.tech_docs_cli = tech_docs_cli
                            alias.alias_notes = alias_notes
                        else:
                            retry_db_operation(lambda: session.delete(alias))

                    for key in request.form:
                        if key.startswith('new_alias_name_'):
                            index = key.split('_')[-1]
                            new_alias = ProductAlias(
                                product_id=product_id,
                                alias_name=request.form.get(f'new_alias_name_{index}'),
                                alias_type=request.form.get(f'new_alias_type_{index}'),
                                alias_approved='new_alias_approved_' + index in request.form,
                                previous_name='new_previous_name_' + index in request.form,
                                tech_docs='new_tech_docs_' + index in request.form,
                                tech_docs_cli='new_tech_docs_cli_' + index in request.form,
                                alias_notes=request.form.get(f'new_alias_notes_{index}')
                            )
                            retry_db_operation(lambda: session.add(new_alias))

                    if product_mkt_life:
                        logger.debug("Updating existing product marketing life.")
                        logger.debug(f"Current product release: {product_mkt_life.product_release}")
                        logger.debug(f"Form product release: {form.product_release.data}")
                        product_mkt_life.product_release = form.product_release.data
                        product_mkt_life.product_release_detail = form.product_release_detail.data
                        product_mkt_life.product_release_link = form.product_release_link.data
                        product_mkt_life.product_eol = form.product_eol.data
                        product_mkt_life.product_eol_detail = form.product_eol_detail.data
                        product_mkt_life.product_eol_link = form.product_eol_link.data
                        logger.debug(f"Updated product release: {product_mkt_life.product_release}")
                    
                    else:
                        logger.debug(f"Creating a new product marketing life entry.")
                        product_mkt_life = ProductMktLife(
                            product_id=product_id,
                            product_release = form.product_release.data,
                            product_release_detail=form.product_release_detail.data,
                            product_release_link=form.product_release_link.data,
                            product_eol=form.product_eol.data,
                            product_eol_detail= form.product_eol_detail.data,
                            product_eol_link=form.product_eol_link.data
                        )
                    
                        session.add(product_mkt_life)
                        logger.debug(f"New product marketing life created: {product_mkt_life}")
                        

                    existing_partner_ids = [ppartner.partner_id for ppartner in existing_product_partners]
                    selected_partner_ids = request.form.getlist('partner')

                    for partner_id in selected_partner_ids:
                        if partner_id not in existing_partner_ids:
                            new_product_partner = ProductPartners(product_id=product.product_id, partner_id=partner_id)
                            retry_db_operation(lambda: session.add(new_product_partner))

                    for ppartner in existing_product_partners:
                        if ppartner.partner_id not in selected_partner_ids:
                            retry_db_operation(lambda: session.delete(ppartner))

                    existing_component_ids = [component.component_id for component in product.components]
                    parent_product_ids = request.form.getlist('product_id[]')
                    component_types = request.form.getlist('component_type[]')

                    retry_db_operation(lambda: session.query(ProductComponents).filter(
                        ProductComponents.component_id == product_id,
                        ProductComponents.product_id.notin_(parent_product_ids)
                    ).delete(synchronize_session=False))

                    for parent_id, comp_type in zip(parent_product_ids, component_types):
                        if parent_id and parent_id != 'Select':
                            existing_component = retry_db_operation(lambda: session.query(ProductComponents).filter_by(
                                component_id=product_id, product_id=parent_id
                            ).first())

                            if existing_component:
                                existing_component.component_type = comp_type
                            else:
                                retry_db_operation(lambda: session.add(ProductComponents(
                                    component_id=product_id,
                                    product_id=parent_id,
                                    component_type=comp_type
                                )))

                    deleted_log_ids = request.form.getlist('deleted_log_ids')
                    for log_id in deleted_log_ids:
                        retry_db_operation(lambda: session.query(ProductLog).filter_by(log_id=log_id).delete())

                    existing_log_ids = request.form.getlist('existing_log_id')
                    for log_id in existing_log_ids:
                        if log_id not in deleted_log_ids:
                            log = retry_db_operation(lambda: session.query(ProductLog).filter_by(log_id=log_id, product_id=product_id).first())
                            if log:
                                edit_notes_field = f'edit_notes_{log_id}'
                                edit_date_field = f'edit_date_{log_id}'
                                if edit_notes_field in request.form and edit_date_field in request.form:
                                    log.edit_notes = request.form[edit_notes_field]
                                    log.edit_date = datetime.strptime(request.form[edit_date_field], '%Y-%m-%d').date()
                                    retry_db_operation(lambda: session.add(log))
                            else:
                                logger.debug(f"No log found for log_id: {log_id}")

                    new_edit_notes = request.form.getlist('new_edit_notes')
                    new_edit_dates = request.form.getlist('new_edit_date')
                    for notes, date_str in zip(new_edit_notes, new_edit_dates):
                        if notes:
                            new_log_date = datetime.strptime(date_str, '%Y-%m-%d').date() if date_str else datetime.utcnow().date()
                            new_log = ProductLog(
                                product_id=product_id,
                                edit_notes=notes,
                                edit_date=new_log_date,
                                username=flask_session.get('username')
                            )
                            retry_db_operation(lambda: session.add(new_log))

                    retry_db_operation(lambda: session.commit())

                    product_id = product_id
                    view_link = url_for('view_routes.view_product_details', product_id=product_id)
                    success_message = f'Successfully updated the product: <a href="{view_link}">{form.product_name.data}</a>'
                    show_form = False

                except Exception as e:
                    logger.error(f"Error during form submission: {e}")
                    session.rollback()

        product_references = retry_db_operation(lambda: session.query(ProductReferences).filter_by(product_id=product_id).all())
        existing_logs = retry_db_operation(lambda: session.query(ProductLog).filter_by(product_id=product_id).all())
        existing_aliases = retry_db_operation(lambda: session.query(ProductAlias).filter_by(product_id=product_id).all())
        form.product_id.choices = [('', 'Select')] + [(str(prod.product_id), prod.product_name) for prod in retry_db_operation(lambda: session.query(Product).order_by(Product.product_name).all())]

        return render_template('opl/edit.html', form=form, product=product, success_message=success_message, show_form=show_form, existing_aliases=existing_aliases, existing_product_partners=existing_product_partners, existing_components=existing_components, existing_logs=existing_logs, reference_forms=reference_forms, user=flask_session.get('username'))
    finally:
        session.close()


DEBUG:redshift_connector.cursor:Cursor.paramstyle=format
DEBUG:redshift_connector.cursor:Cursor.paramstyle=format
DEBUG:redshift_connector.cursor:Cursor.paramstyle=format
DEBUG:redshift_connector.cursor:Cursor.paramstyle=format
DEBUG:redshift_connector.cursor:Cursor.paramstyle=format
DEBUG:redshift_connector.cursor:Cursor.paramstyle=format
DEBUG:redshift_connector.cursor:Cursor.paramstyle=format
DEBUG:routes.edit_routes:Form data received: ImmutableMultiDict([('csrf_token', 'IjlhOGM1MTExNDE4YTIxN2QzOThkNDNiYTM4OWVkZTFkNzM3NTZlMzMi.Zp3hPg.fCyS57txQL37TNZu1aO8voeCKRI'), ('product_name', 'Red Hat build of OptaPlanner'), ('product_type', '949d12eb-dcb3-4380-b991-cb9ed01d48cb'), ('product_type', '26f925da-c7c3-450d-ae3b-595bebedaafa'), ('product_description', 'OptaPlanner is a lightweight, embeddable planning engine that helps programmers solve optimization problems efficiently. Often used with Red Hat Fuse (Apache Camel).'), ('product_portfolio', '559799df-404e-4c36-872e-65015cadc06f'), ('product_note', 'No short names are approved for this component, as Legal does not approve starting short names with a third-party trademark.\r\n\r\nDo not use "OptaPlanner" alone, to avoid confusion with the open source project.'), ('product_link', 'https://www.redhat.com/en/blog/red-hat-build-optaplanner-now-available-red-hat-application-foundations'), ('link_description', 'Red Hat build of OptaPlanner blog post on ...
INFO:flask_wtf.csrf:The CSRF token has expired.
DEBUG:routes.edit_routes:Starting session transaction
DEBUG:routes.edit_routes:Updating existing product marketing life.
DEBUG:routes.edit_routes:Current product release: 2023-01-09
DEBUG:routes.edit_routes:Form product release: 2023-01-09
DEBUG:routes.edit_routes:Updated product release: 2023-01-09
DEBUG:redshift_connector.cursor:Cursor.paramstyle=format
DEBUG:redshift_connector.cursor:Cursor.paramstyle=format
ERROR:routes.edit_routes:Error during form submission: UPDATE statement on table 'product_alias' expected to update 4 row(s); 6 were matched.
