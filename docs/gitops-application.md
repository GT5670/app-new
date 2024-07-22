# GitOps Http Application Sample

## HTTP Application 
This Gitops sample provides a standard HTTP component consisting of a deployment, service and route. 

The following day 2 edit/update operations supported:
    set/get image - updates the image for this component 
    set/get replicas

 python app.py
None of PyTorch, TensorFlow >= 2.0, or Flax have been found. Models won't be available and only tokenizers, configuration and file/data utilities can be used.
config.json: 100%|████████████████████████████████████████████████████████████████| 1.58k/1.58k [00:00<00:00, 4.22MB/s]
Traceback (most recent call last):
  File "/home/gtrivedi/Desktop/demo/app.py", line 7, in <module>
    summarizer = pipeline("summarization", model="facebook/bart-large-cnn")
                 ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "/home/gtrivedi/.local/lib/python3.12/site-packages/transformers/pipelines/__init__.py", line 895, in pipeline
    framework, model = infer_framework_load_model(
                       ^^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "/home/gtrivedi/.local/lib/python3.12/site-packages/transformers/pipelines/base.py", line 234, in infer_framework_load_model
    raise RuntimeError(
RuntimeError: At least one of TensorFlow 2.0 or PyTorch should be installed. To install TensorFlow 2.0, read the instructions at https://www.tensorflow.org/install/ To install PyTorch, read the instructions at https://pytorch.org/.
 gtrivedi@gtrivedi-thinkpadt14sgen1  ~/Desktop/demo  
