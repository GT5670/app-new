# GitOps Http Application Sample

## HTTP Application 
This Gitops sample provides a standard HTTP component consisting of a deployment, service and route. 

The following day 2 edit/update operations supported:
    set/get image - updates the image for this component 
    set/get replicas

127.0.0.1 - - [22/Jul/2024 15:08:24] "POST /analyze HTTP/1.1" 500 -
Traceback (most recent call last):
  File "/home/gtrivedi/Desktop/demo/demo_env/lib64/python3.12/site-packages/flask/app.py", line 1498, in __call__
    return self.wsgi_app(environ, start_response)
           ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "/home/gtrivedi/Desktop/demo/demo_env/lib64/python3.12/site-packages/flask/app.py", line 1476, in wsgi_app
    response = self.handle_exception(e)
               ^^^^^^^^^^^^^^^^^^^^^^^^
  File "/home/gtrivedi/Desktop/demo/demo_env/lib64/python3.12/site-packages/flask/app.py", line 1473, in wsgi_app
    response = self.full_dispatch_request()
               ^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "/home/gtrivedi/Desktop/demo/demo_env/lib64/python3.12/site-packages/flask/app.py", line 882, in full_dispatch_request
    rv = self.handle_user_exception(e)
         ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "/home/gtrivedi/Desktop/demo/demo_env/lib64/python3.12/site-packages/flask/app.py", line 880, in full_dispatch_request
    rv = self.dispatch_request()
         ^^^^^^^^^^^^^^^^^^^^^^^
  File "/home/gtrivedi/Desktop/demo/demo_env/lib64/python3.12/site-packages/flask/app.py", line 865, in dispatch_request
    return self.ensure_sync(self.view_functions[rule.endpoint])(**view_args)  # type: ignore[no-any-return]
           ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "/home/gtrivedi/Desktop/demo/app.py", line 50, in analyze
    yaml_contents = fetch_yaml_files(repo_url, branch)
                    ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "/home/gtrivedi/Desktop/demo/app.py", line 16, in fetch_yaml_files
    response.raise_for_status()
  File "/home/gtrivedi/Desktop/demo/demo_env/lib64/python3.12/site-packages/requests/models.py", line 1024, in raise_for_status
    raise HTTPError(http_error_msg, response=self)
requests.exceptions.HTTPError: 404 Client Error: Not Found for url: https://api.github.com/repos/https://github.com/GT5670/doctest/contents?ref=main
