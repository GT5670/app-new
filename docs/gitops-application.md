# GitOps Http Application Sample

## HTTP Application 
This Gitops sample provides a standard HTTP component consisting of a deployment, service and route. 

The following day 2 edit/update operations supported:
    set/get image - updates the image for this component 
    set/get replicas

DEBUG:routes.edit_routes:Route accessed with method: POST
ERROR:app:Exception on /opl/edit-product/ef200820-3c1a-4263-ba4e-d8b3491c3c8e [POST]
Traceback (most recent call last):
  File "/home/gtrivedi/.local/lib/python3.12/site-packages/flask/app.py", line 2190, in wsgi_app
    response = self.full_dispatch_request()
               ^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "/home/gtrivedi/.local/lib/python3.12/site-packages/flask/app.py", line 1486, in full_dispatch_request
    rv = self.handle_user_exception(e)
         ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "/home/gtrivedi/.local/lib/python3.12/site-packages/flask/app.py", line 1484, in full_dispatch_request
    rv = self.dispatch_request()
         ^^^^^^^^^^^^^^^^^^^^^^^
  File "/home/gtrivedi/.local/lib/python3.12/site-packages/flask/app.py", line 1469, in dispatch_request
    return self.ensure_sync(self.view_functions[rule.endpoint])(**view_args)
           ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "/home/gtrivedi/.local/lib/python3.12/site-packages/flask_principal.py", line 199, in _decorated
    rv = f(*args, **kw)
         ^^^^^^^^^^^^^^
  File "/home/gtrivedi/git/gitlab/opl-ui/routes/edit_routes.py", line 156, in edit_product_details
    if form.validate_on_submit() or 'submit' in request.form:
       ^^^^^^^^^^^^^^^^^^^^^^^^^
  File "/home/gtrivedi/.local/lib/python3.12/site-packages/flask_wtf/form.py", line 86, in validate_on_submit
    return self.is_submitted() and self.validate(extra_validators=extra_validators)
                                   ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "/home/gtrivedi/.local/lib/python3.12/site-packages/wtforms/form.py", line 329, in validate
    return super().validate(extra)
           ^^^^^^^^^^^^^^^^^^^^^^^
  File "/home/gtrivedi/.local/lib/python3.12/site-packages/wtforms/form.py", line 146, in validate
    if not field.validate(self, extra):
           ^^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "/home/gtrivedi/.local/lib/python3.12/site-packages/wtforms/fields/core.py", line 235, in validate
    self.pre_validate(form)
  File "/home/gtrivedi/.local/lib/python3.12/site-packages/wtforms/fields/choices.py", line 202, in pre_validate
    raise TypeError(self.gettext("Choices cannot be None."))
TypeError: Choices cannot be None.
