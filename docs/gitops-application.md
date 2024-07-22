# GitOps Http Application Sample

## HTTP Application 
This Gitops sample provides a standard HTTP component consisting of a deployment, service and route. 

The following day 2 edit/update operations supported:
    set/get image - updates the image for this component 
    set/get replicas

Traceback (most recent call last):
  File "/home/gtrivedi/Desktop/demo/demo_env/lib64/python3.12/site-packages/huggingface_hub/utils/_errors.py", line 304, in hf_raise_for_status
    response.raise_for_status()
  File "/home/gtrivedi/Desktop/demo/demo_env/lib64/python3.12/site-packages/requests/models.py", line 1024, in raise_for_status
    raise HTTPError(http_error_msg, response=self)
requests.exceptions.HTTPError: 401 Client Error: Unauthorized for url: https://huggingface.co/ssleifer/distilbart-cnn-12-6/resolve/main/config.json

The above exception was the direct cause of the following exception:

Traceback (most recent call last):
  File "/home/gtrivedi/Desktop/demo/demo_env/lib64/python3.12/site-packages/transformers/utils/hub.py", line 402, in cached_file
    resolved_file = hf_hub_download(
                    ^^^^^^^^^^^^^^^^
  File "/home/gtrivedi/Desktop/demo/demo_env/lib64/python3.12/site-packages/huggingface_hub/utils/_deprecation.py", line 101, in inner_f
    return f(*args, **kwargs)
           ^^^^^^^^^^^^^^^^^^
  File "/home/gtrivedi/Desktop/demo/demo_env/lib64/python3.12/site-packages/huggingface_hub/utils/_validators.py", line 114, in _inner_fn
    return fn(*args, **kwargs)
           ^^^^^^^^^^^^^^^^^^^
  File "/home/gtrivedi/Desktop/demo/demo_env/lib64/python3.12/site-packages/huggingface_hub/file_download.py", line 1240, in hf_hub_download
    return _hf_hub_download_to_cache_dir(
           ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "/home/gtrivedi/Desktop/demo/demo_env/lib64/python3.12/site-packages/huggingface_hub/file_download.py", line 1347, in _hf_hub_download_to_cache_dir
    _raise_on_head_call_error(head_call_error, force_download, local_files_only)
  File "/home/gtrivedi/Desktop/demo/demo_env/lib64/python3.12/site-packages/huggingface_hub/file_download.py", line 1854, in _raise_on_head_call_error
    raise head_call_error
  File "/home/gtrivedi/Desktop/demo/demo_env/lib64/python3.12/site-packages/huggingface_hub/file_download.py", line 1751, in _get_metadata_or_catch_error
    metadata = get_hf_file_metadata(
               ^^^^^^^^^^^^^^^^^^^^^
  File "/home/gtrivedi/Desktop/demo/demo_env/lib64/python3.12/site-packages/huggingface_hub/utils/_validators.py", line 114, in _inner_fn
    return fn(*args, **kwargs)
           ^^^^^^^^^^^^^^^^^^^
  File "/home/gtrivedi/Desktop/demo/demo_env/lib64/python3.12/site-packages/huggingface_hub/file_download.py", line 1673, in get_hf_file_metadata
    r = _request_wrapper(
        ^^^^^^^^^^^^^^^^^
  File "/home/gtrivedi/Desktop/demo/demo_env/lib64/python3.12/site-packages/huggingface_hub/file_download.py", line 376, in _request_wrapper
    response = _request_wrapper(
               ^^^^^^^^^^^^^^^^^
  File "/home/gtrivedi/Desktop/demo/demo_env/lib64/python3.12/site-packages/huggingface_hub/file_download.py", line 400, in _request_wrapper
    hf_raise_for_status(response)
  File "/home/gtrivedi/Desktop/demo/demo_env/lib64/python3.12/site-packages/huggingface_hub/utils/_errors.py", line 352, in hf_raise_for_status
    raise RepositoryNotFoundError(message, response) from e
huggingface_hub.utils._errors.RepositoryNotFoundError: 401 Client Error. (Request ID: Root=1-669e1540-2284731e28602ccd414a2658;839d4244-5674-42f2-822c-4655f6bdf752)

Repository Not Found for url: https://huggingface.co/ssleifer/distilbart-cnn-12-6/resolve/main/config.json.
Please make sure you specified the correct `repo_id` and `repo_type`.
If you are trying to access a private or gated repo, make sure you are authenticated.
Invalid username or password.

The above exception was the direct cause of the following exception:

Traceback (most recent call last):
  File "/home/gtrivedi/Desktop/demo/app.py", line 7, in <module>
    summarizer = pipeline("summarization", model="ssleifer/distilbart-cnn-12-6")
                 ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "/home/gtrivedi/Desktop/demo/demo_env/lib64/python3.12/site-packages/transformers/pipelines/__init__.py", line 768, in pipeline
    resolved_config_file = cached_file(
                           ^^^^^^^^^^^^
  File "/home/gtrivedi/Desktop/demo/demo_env/lib64/python3.12/site-packages/transformers/utils/hub.py", line 425, in cached_file
    raise EnvironmentError(
OSError: ssleifer/distilbart-cnn-12-6 is not a local folder and is not a valid model identifier listed on 'https://huggingface.co/models'
If this is a private repository, make sure to pass a token having permission to this repo either by logging in with `huggingface-cli login` or by passing `token=<your_token>`
