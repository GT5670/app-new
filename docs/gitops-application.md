# GitOps Http Application Sample

## HTTP Application 
This Gitops sample provides a standard HTTP component consisting of a deployment, service and route. 

The following day 2 edit/update operations supported:
    set/get image - updates the image for this component 
    set/get replicas

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
                            logger.debug(f"Updating alias: {alias.alias_id} with name: {alias_name}")
                        else:
                            logger.debug(f"Deleting alias: {alias.alias_id} with name: {alias.alias_name}")
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
                            logger.debug(f"Adding new alias: {new_alias.alias_name}")
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
