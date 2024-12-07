PFOLIO_FRONTEND | DEPLOY


# Deploy a React Web Application to Google Cloud
https://www.youtube.com/watch?v=IjUnQ9kMnVo

# hero
```
I’m long term IT - BSc Comp Science - 1989. Started career with Sun Microsystems startup in UK - 1990. UNIX based Env (Solaris). Contracted in The City over the Millenium. In early 2000s added career in the Arts - Actor, Writer, Producer, Director. Mixed both since then.

```

# local DEPLOY
```
# git clone
git clone git@github.com:heidless-stillwater/pfolio-frontend.git

# build
npm install

# run locally
npm run dev

```

# gCloud DEPLOY
## project
```
source .env-vars

```

## storage bucket
```
# Set up a Cloud Storage bucket
source .env-vars
echo ' '
echo GCP_BUCKET: ${GCP_BUCKET}
echo GCP_REGION: ${GCP_REGION}

export bucket_path='gs://'${GCP_BUCKET}
echo bucket_path: ${bucket_path}

gsutil mb -l ${GCP_REGION} ${bucket_path}

## [bucket permissions]()
gsutil iam ch allUsers:objectViewer $bucket_path

```

# Cloud Run

## push to repository
```
source .env-vars
gcloud builds submit --tag gcr.io/${GCP_PROJECT}/${GCP_SERVICE_NAME} .

```

## build service
```
source .env-vars
gcloud run deploy ${GCP_SERVICE_NAME} \
     --platform managed \
     --region ${GCP_REGION} \
     --image gcr.io/${GCP_PROJECT}/${GCP_SERVICE_NAME} \
     --add-cloudsql-instances ${GCP_PROJECT}:${GCP_REGION}:${GCP_INSTANCE} \
     --allow-unauthenticated

# URL
https://pfolio-frontend-svc-1089619978780.europe-west2.run.app

```

# UTILS
## gCloud upload images
```
# local
IMAGES_DIR=./public/images
LOGO_DIR=./public/logo




# gsCloud

GCP_IMAGES_DIR=gs://${GCP_BUCKET}
GCP_LOGO_DIR=gs://${GCP_BUCKET}


# logos
gsutil cp -r ${IMAGES_DIR} ${GCP_BUCKET}

# images
gsutil cp -r ${IMAGES_DIR} ${GCP_BUCKET}




echo ICONS_DIR: ${LOGO_DIR}
echo IMAGES_DIR: ${IMAGES_DIR}
echo BUCKET: ${GCP_BUCKET}
echo GCP_IMAGES_DIR: ${GCP_IMAGES_DIR}
echo GCP_LOGO_DIR: ${GCP_LOGO_DIR}

# upload
gsutil cp -r ${IMAGES_DIR} ${GCP_IMAGES_DIR}

gsutil cp -r ${LOGO_DIR} ${GCP_LOGO_DIR}

```

## create Serice
```
-> deploy uploaded Container Registry
-> i.e  NOT continuous deployment

CLOUD RUN->DeployContainer->Service->'Select'
--
IMAGE:
gcr.io/heidless-portfolio-0
- heidless-pfolio-frontend-0
	- 'latest'
--

REGION:
--
europe-west2
--

AUTHENTICATION:
--
Allow unauthenticated invocations
--

CREATE

```





## Access app
### https://pf-frontend-svc-0-tr6lwdigpa-ew.a.run.app

## setup

### package.json

```
# ensure 'start' references $PORT
# allows cloud run to manage random PORT
  "scripts": {
    "dev": "next dev",
    "build": "next build",
    "start": "next start -- --port 3000"
  },
    
npm install 

# test by specifying port
npm run build
npm start 5000
```


## docker
### Dockerfile
```
vi Dockerfile
# base image
FROM node:16.15.1-alpine

# Create and change to the app directory.
WORKDIR /usr/app

COPY . .

# Install production dependencies.

# use 'clean install' (ci)
RUN npm ci --only=production

RUN npm run build

CMD ["npm", "start"]
```

### build | run
```
docker build . -t pfolio-frontend
docker run -p 3000:3000 -e=PORT=3000 pfolio-frontend
```

---
---
---
##
##

### github secrets
FIGURE out continuous deployment
```
# encode service account key
base64 -i pfolio-backend-deploy-2-d2290096b8e5.json > pfolio-key.base64

# configure github
github.com
'heidless-stillwater'->frontend-west1->Settings->Secrets & Variables->Actions
'New Repository Secret'
NAME: FRONTEND_WEST1_SECRET
< paste contents of 'pfolio-frontend-key.base64'>
'Add Secret'
-
```

### configure
```
https://console.cloud.google.com/run/create?enableapi=true&authuser=0&hl=en_GB&project=pfolio-backend-deploy-0-380215

'Cloud Run'->Create Service
-
pfolio-frontend-service
'Setup Cloud Build'
< pick repository >
'save'
-

# register container
'Container Registry'-> enable if not already done so

# service account
'IAM & ADMIN'->Service Accounts
-
NAME: pfolio-frontend-svcaccount
DESC: service account for github actions
ROLES:
Editor
Cloud Run Admin
Storage Admin
Service Account User
'DONE'

# newly created instance
'Service Accounts'->pfolio-frontend-svcaccount->
'3 dots'->Manage Keys->Add Key->JSON

# keyfile auto downloads
# copy to project root dir
i.e. pfolio-backend-deploy-0-380215-8f9cc9423783.json

# enable Registry API - IF NEEDED
'Container Registery'-'enable'

# ename Cloud Run API - if NECESSARY
'Cloud Run API'->'enable'

```

## configure github workflows
```
# in project root dir
vi .github/workflows/cloud-run.yml
-
europ-west2
-

# refresh gitub
git add .
git commit -m "added .github structure"

# github secrets
# encode service account key
i.e.base64 -i pfolio-backend-deploy-0-380215-8f9cc9423783.json

<github repository>->Settings->Secrets & Variables->Actions
'New Repository Secret'
-
CLOUD_RUN_PROJECT_NAME=pfolio-frontend-deploy-0
CLOUD_RUN_SERVICE_ACCOUNT=`base64 -i pfolio-backend-deploy-0-380215-8f9cc9423783.json`
-

# now trigger a build to confirm success/failure
<github repository>->Actions-><workflow of interest>
```

## refresh install of SDK
https://cloud.google.com/sdk/docs/install#deb
```
sudo apt-get install apt-transport-https ca-certificates gnupg

echo "deb [signed-by=/usr/share/keyrings/cloud.google.gpg] https://packages.cloud.google.com/apt cloud-sdk main" | sudo tee -a /etc/apt/sources.list.d/google-cloud-sdk.list

curl https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key --keyring /usr/share/keyrings/cloud.google.gpg add -

sudo apt-get update && sudo apt-get install google-cloud-cli

```

## Cloud Run: enable permissions
```
'Cloud Run'-><app service>->Triggers->Allow Unauthenticated invocations

gcloud run services add-iam-policy-binding nextjs-test \
    --member="allUsers" \
    --role="roles/run.invoker"
```

## continuous deployment
```
'Cloud Run'->Service Details->Setup Continuous Deployment
-
'Enable Cloud Build API'
<set repository>
-
```

## access frontend app
```
PROJECT: pfolio-frontend-deploy-0->Cloud Run->pfolio-frontend-service
```
https://pfolio-frontend-service-7dd5pbcoiq-nw.a.run.app


## backups
### frontend
```
STORAGE BUCKET: pfolio-frontend-deploy-0->'Cloud Storage'
-
frontend-bucket-0
-
```
