
# [Quickstart: Deploy a Python service to Cloud Run](https://cloud.google.com/run/docs/quickstarts/build-and-deploy/deploy-python-service)

## init gCloud project
```
gcloud init

PROJECT:
heidless-portfolio-4

gcloud config set project heidless-portfolio-4

gcloud services enable run.googleapis.com \
    cloudbuild.googleapis.com

gcloud projects add-iam-policy-binding heidless-portfolio-4 \
    --member=serviceAccount:865665966029-compute@developer.gserviceaccount.com \
    --role=roles/cloudbuild.builds.builder

```

## init Applicat

vi main.py
--
import os

from flask import Flask

app = Flask(__name__)


@app.route("/")
def hello_world():
    """Example Hello World route."""
    name = os.environ.get("NAME", "World")
    return f"Hello {name}!"


if __name__ == "__main__":
    app.run(debug=True, host="0.0.0.0", port=int(os.environ.get("PORT", 8080)))

--

```

# deploy
```
gcloud run deploy --source .



```