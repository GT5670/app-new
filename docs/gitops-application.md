# GitOps Http Application Sample

## HTTP Application 
This Gitops sample provides a standard HTTP component consisting of a deployment, service and route. 

The following day 2 edit/update operations supported:
    set/get image - updates the image for this component 
    set/get replicas

from flask import Blueprint, request, jsonify, render_template
import markdown
from .mstp_rules import analyze_content

main = Blueprint('main', __name__)

feedback_list = []
document_content = ""

@main.route('/')
def index():
    return render_template('index.html')

@main.route('/upload', methods=['POST'])
def upload_file():
    global document_content
    if 'file' not in request.files:
        return jsonify({"error": "No file part"})
    file = request.files['file']
    if file.filename == '':
        return jsonify({"error": "No selected file"})
    if file:
        content = file.read().decode('utf-8')
        document_content = content
        html_content = markdown.markdown(content)
        suggestions = analyze_content(content)
        return jsonify({"message": "File uploaded successfully", "content": html_content, "suggestions": suggestions})

@main.route('/feedback', methods=['POST'])
def submit_feedback():
    data = request.get_json()
    feedback = data.get('feedback')
    feedback_list.append(feedback)
    return jsonify({"message": "Feedback submitted successfully", "feedback_list": feedback_list})

@main.route('/feedbacks', methods=['GET'])
def get_feedbacks():
    return jsonify({"feedback_list": feedback_list})


python run.py
Traceback (most recent call last):
  File "/home/gtrivedi/Desktop/style.io/run.py", line 3, in <module>
    app = create_app()
          ^^^^^^^^^^^^
  File "/home/gtrivedi/Desktop/style.io/app/__init__.py", line 5, in create_app
    from .app import main
  File "/home/gtrivedi/Desktop/style.io/app/app.py", line 3, in <module>
    from .mstp_rules import analyze_content
  File "/home/gtrivedi/Desktop/style.io/app/mstp_rules.py", line 4, in <module>
    nlp = spacy.load("en_core_web_sm")
          ^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "/home/gtrivedi/Desktop/style.io/venv/lib64/python3.12/site-packages/spacy/__init__.py", line 51, in load
    return util.load_model(
           ^^^^^^^^^^^^^^^^
  File "/home/gtrivedi/Desktop/style.io/venv/lib64/python3.12/site-packages/spacy/util.py", line 472, in load_model
    raise IOError(Errors.E050.format(name=name))
OSError: [E050] Can't find model 'en_core_web_sm'. It doesn't seem to be a Python package or a valid path to a data directory.
