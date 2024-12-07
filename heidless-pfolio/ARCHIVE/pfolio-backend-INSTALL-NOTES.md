PFOLIO-BACKEND | DEPLOY


##################################################
# if on GAE

# init env
```
gcloud init

# app vars
source config/.env-vars

# initialise App Engine
gcloud app create --region=europe-west2

python -m venv venv
source venv/bin/activate

pip install -r requirements.txt

```

# script to handle ALL DB creation & config
```
./pfolio-CREATE-INSTANCE

````

# Download Cloud SQL Auth proxy to connect to Cloud SQL from your local machine
```
# init proxy
curl -o cloud-sql-proxy https://storage.googleapis.com/cloud-sql-connectors/cloud-sql-proxy/v2.7.0/cloud-sql-proxy.linux.amd64
chmod +x cloud-sql-proxy

```

# DB URL
## assemble link from the above info
```
postgres://<GCP_DB_USER:>:<GCP_INSTANCE_ROOT_PWD>@//cloudsql/<GCP_PROJECT>:<REGION>:<INSTANCE>/<DB>
--
postgres://pfolio-user-0:GJaUUsg_%RYnXVCB@//cloudsql/h-pfolio:europe-west2:pfolio-backend-instance-0/pfolio-backend-db-0

--

````

# service account(s)
```
# 'IAM & ADMIN'->Service Accounts
h-pfolio@appspot.gserviceaccount.com
-
edit principal

-

# add ROLES to allow access to DB & 'secrets'
--
Secret Manager Secret Accessor
Cloud SQL Admin
Storage Admin
--
```

# generate & install KEY file
```
##################################
# uncompress CREDENTIALS_ENCRYPTED
tar xvf backend_credentials.gz

# compress CREDENTIALS_ENCRYPTED
tar czf backend_credentials.gz CREDENTIALS_ENCRYPTED
rm -rf CREDENTIALS_ENCRYPTED

# generate new KEY
'IAM & ADMIN'->Service Accounts->'3 dots'->Manage Keys
'ADD KEY'->JSON

# Download & install json file

# copy to local project/app directory'
/home/heidless/projects/backend-live
```

# secrets setup
```
#######################################
# setup local environment - TEMPORARILY

cd config

echo DATABASE_URL=postgres://${GCP_DB_USER}:${GCP_INSTANCE_ROOT_PWD}@//cloudsql/${GCP_PROJECT}:${GCP_REGION}:${GCP_INSTANCE}/${GCP_DB_NAME} > .env

echo SECRET_KEY=$(cat /dev/urandom | LC_ALL=C tr -dc '[:alpha:]'| fold -w 50 | head -n1) >> .env

echo GS_BUCKET_NAME=${GCP_BUCKET} >> .env

echo FRONTEND_URL=${GCP_FRONTEND_URL} >> .env

#########################
# store in secret manager

gcloud secrets create ${GCP_SECRET_NAME} --data-file .env

# Grant access to the secret to the service account
gcloud secrets add-iam-policy-binding ${GCP_SECRET_NAME} \
    --member serviceAccount:${GCP_PROJECT}@appspot.gserviceaccount.com \
    --role roles/secretmanager.secretAccessor
		
# test - retrieve content of ${GCP_SECRET_NAME}
gcloud secrets versions access latest --secret ${GCP_SECRET_NAME} && echo ""

#######
# UTILS
#
# when reseting - delete SECRET
gcloud secrets delete ${GCP_SECRET_NAME}

gcloud secrets describe ${GCP_SECRET_NAME}


```

# settings.py
Need to link to the 'key' file you downloaded earlier.

### set GS_CREDENTIALS
```
GS_CREDENTIALS = service_account.Credentials.from_service_account_file(
    os.path.join(BASE_DIR, 'config/h-pfolio-5bed07c9d6bc')
)

```

### set STATIC_URL
```
GS_BUCKET_NAME = 'heidless-pfolio-bucket-6'

STATIC_URL = 'https://storage.cloud.google.com/heidless-pfolio-bucket-4/'

```

## disable local settings to force use of Google Secrets
```
mv config/.env config/.env-h-pfolio
```

# test cloud DB connection
```
# test - retrieve content of ${GCP_SECRET_NAME}
gcloud secrets versions access latest --secret ${GCP_SECRET_NAME} && echo ""


# login to instanece as 'postgres' password='postgres'

# check if can access DB directly
gcloud sql connect ${GCP_INSTANCE} --database ${GCP_DB_NAME} --user=postgres --quiet

##########
# utils
#
#sudo systemctl status postgresql.service
#sudo systemctl restart postgresql.service
#gcloud secrets versions access latest --secret ${HEIDLESS_SECRET} && echo

```

# upload STATIC files
```
cd media

echo GCP_BUCKET:: ${GCP_BUCKET:}

########
# images
#
gcloud storage cp --recursive images gs://${GCP_BUCKET}

#######
# icons
#
gcloud storage cp --recursive icons gs://${GCP_BUCKET}

```

# backup static files
```
#cd media
cd BACKUPS/milestones

# images
gsutil -m cp -r \
  "gs://pfolio-backend-bucket-6/images" \
  .

# icons
gsutil -m cp -r \
  "gs://pfolio-backend-bucket-6/icons" \
  .

````

# enable cloud proxy
```
source config/.env-vars

./cloud-sql-proxy --credentials-file config/${GCP_CREDENTIALS} ${GCP_PROJECT}:${GCP_REGION}:${GCP_INSTANCE}

# kill & restart - IF address already in use
#sudo lsof -i -P -n | grep LISTEN
sudo lsof -i :5432
sudo kill -9 458

```
# init & run backend
```
###########
source config/.env-vars

# using GCP
export GOOGLE_CLOUD_PROJECT=h-pfolio
export USE_CLOUD_SQL_AUTH_PROXY=True
export GOOGLE_APPLICATION_CREDENTIALS=config/<CREDENTIALS_FILE>.json

source ./venv/bin/activate

./manage.py makemigrations

./manage.py migrate

python manage.py createsuperuser
-
heidless
rob.lockhart@yahoo.co.uk
sdfsdgasgTHW66GDGdfdff
-

# mkdir ./build/static
./manage.py collectstatic

############
RUN
#
python manage.py runserver 8000

#################
# view site/admin
http://localhost:8000/admin

##########
# UTILS
#
# set executable version
sudo apt update && sudo apt upgrade
sudo apt install python-is-python3

```

# BACKUP/RESTORE
### prep
```
# set permissoons on INSTANCE service account
echo '####################################################'
echo '# SET PERMISSIONS - objectAdmin & legacyBucketWriter'
echo GCP_INSTANCE: ${GCP_INSTANCE}
export SQL_SVC_ACC=`gcloud sql instances describe ${GCP_INSTANCE} | grep serviceAccountEmailAddress`
echo SQL_SVC_ACC: ${SQL_SVC_ACC}

# extract email account
spl=(${(@s/ /)SQL_SVC_ACC})
SQL_SVC_EMAIL=${spl[2]}

# set PERMISSION on Svc Account
export DB_SVC_ACCOUNT=${SQL_SVC_EMAIL}
echo DB_SVC_ACCOUNT: ${DB_SVC_ACCOUNT}
echo GCP_BUCKET: ${GCP_BUCKET}

echo 'objectAdmin'
gsutil iam ch serviceAccount:${DB_SVC_ACCOUNT}:objectAdmin gs://${GCP_BUCKET}

echo gsutil iam ch serviceAccount:${DB_SVC_ACCOUNT}:objectAdmin gs://${GCP_BUCKET}

```

### Take Backup
```
export BK_COMMENT='-PFOLIO-RELEASE-1.1-'
export BK_TIMESTAMP=`date +%s`

export GCP_FILE=${GCP_DB_NAME}-${BK_COMMENT}-${BK_TIMESTAMP}.gz
echo GCP_FILE: ${GCP_FILE}
echo ' '

echo '#########'
echo 'BACKUP DB'
echo BK_PREFIX: ${BK_PREFIX}
echo GCP_INSTANCE: ${GCP_INSTANCE}
echo GCP_BUCKET: ${GCP_BUCKET}
echo GCP_FILE: ${GCP_FILE}
echo GCP_DB_NAME:: ${GCP_DB_NAME}
echo ' '

gcloud sql export sql ${GCP_INSTANCE} gs://${GCP_BUCKET}/backups/${GCP_FILE}    \
--database=${GCP_DB_NAME}	 \
--offload

```

### DOWNLOAD BACKUP
```
# cd <BACKUP DIR>

echo GCP_FILE: ${GCP_FILE}
gcloud storage cp gs://${GCP_BUCKET}/backups/${GCP_FILE} .

```

### UPLOAD BACKUP
```
source ../config/.env-vars

export GCP_FILE=pfolio-instance-0--snapshot--1731516745.gz
echo GCP_FILE: ${GCP_FILE}

echo GCP_BUCKET: ${GCP_BUCKET}
echo GCP_FILE: ${GCP_FILE}
gcloud storage cp ${GCP_FILE} gs://${GCP_BUCKET}/backups/${GCP_FILE}

```

### RE-CREATE DB
```
# pq: database "photo-app-0-db-0" is being accessed by other users
# CLOSE ALL TABS ACCESSING APP
#

echo 're-create DB'
echo GCP_DB_NAME: ${GCP_DB_NAME}
echo GCP_INSTANCE: ${GCP_INSTANCE}
echo ' '

gcloud sql databases delete ${GCP_DB_NAME} \
    --instance ${GCP_INSTANCE}

gcloud sql databases create ${GCP_DB_NAME} \
    --instance ${GCP_INSTANCE}

```

### RESTORE BACKUP
```
echo GCP_INSTANCE: ${GCP_INSTANCE}
echo GCP_DB_NAME: ${GCP_DB_NAME}
echo GCP_BUCKET: ${GCP_BUCKET}
echo GCP_FILE: ${GCP_FILE}
echo ' '

gcloud sql import sql ${GCP_INSTANCE} gs://${GCP_BUCKET}/backups/${GCP_FILE} \
--database=${GCP_DB_NAME}

```

# Deploy the app to the App Engine standard environment

```
# initialze app - creates 'app.yaml'
vi app/app.yaml
--
runtime: python39
env: standard
entrypoint: gunicorn -b :$PORT config.wsgi:application

handlers:
- url: /.*
  script: auto

runtime_config:
  python_version: 3
--

# deploy to app engine
gcloud app deploy

# view ADMIN 
https://h-pfolio.nw.r.appspot.com/admin

#################
# display APP URL
gcloud app describe --format "value(defaultHostname)"
-
https://h-pfolio.nw.r.appspot.com/admin
-

# monitor logs
gcloud app logs tail -s default
--
target url: https://pfolio-backend-2.ew.r.appspot.com
target service account: pfolio-backend-2@appspot.gserviceaccount.com

--

# re-deploy
gcloud app deploy

```


![abba-1974-billboard-1548.webp](resources/f95dff06bf7846dab8b816b01b7fe4fc.webp)

# Learn 'phonetically'
When ABBA had their first hits in the English Language - they did not speak English.

They learned their own songs phonetically and achieved huge success.

Subsequently, they mastered English and went on to further success & achieve iconic status.

Sadly, I cannot guarantee international fame & fortune!

However, If you're new to Google Cloud then it is feasible to learn by doing.

Might I humbly suggest, 'It's the Name of the Game' ;-)

My aim here it to enable a Google Cloud newbie to walk through the steps below to deploy an app to Google App Engine.

By focusing on the 'how' you learn the 'what' and so gain foundational knowlege upon which to build & re-enforce.

J.F.D.I 

One Love

===============================================================

# target audience

By new to Google Cloud I do NOT mean new to IT.

Pre-requisites:
- vScode
	- Linux via WSL (Ubuntu 22.04)
- Linux terminal
	- file system
		- navigation
		- modification
- Python (basic)
	- focus is on 'scripting'
		- variable creation & assignment
		- conditional statements

## shaving 'yaks'

Yak shaving refers to a task, that leads you to perform another related task and so on, and so on — all distracting you from your original goal. This is sometimes called “going down the rabbit hole.”

ORIGIN: 

https://americanexpress.io/yak-shaving/#:~:text=%E2%80%9CYak%20shaving%E2%80%9D%20(or%20%E2%80%9C,The%20Ren%20%26%20Stimpy%20Show.%E2%80%9D

![Yak_shaving.jpg](resources/9c0d9861931d42d588501ca1283c27af.jpg)

If you're unsure whether you fulfill these pre-requisites I'd suggest - Just Do It.

Start the Walkthrough.

Whenever you hit a gap in your knowledge - take some time to research the gap.

The amount of Yak Shaving is an excellent calibration of your actual skills as opposed to your own opinion of your skills - they rarely match!

Gaining 'new' knowledge builds on your accumulated knowledge. 

This can confirm your expertise AND it can also expose areas where you are NOT as expert as you believed yourself to be.

Diplomatically put - You Don't Know What You Don't Know.

This is a clinically researched phenomenon known as the Dunig-Kruger Effect.

Duning-Kruger should be your spirit guide. We ALL suffer from it! - Including YOU.

Duning-Kruger: https://www.youtube.com/watch?v=y50i1bI2uN4

Some Yak Shaving is inevitable & even beneficial in refining your expertise.

When learning - Yak-shaving is useful to calibrate where your at. 

If you're doing none & in the learning phase then I'd suggest you are not challenging yourself and tipping into procrastination.

However, if you spend most of your time doing so - you likely should focus on achieving the pre-requisites before resuming this tutorial.

Bottom line: Humility is HUGELY valuable.

**'Humility is not thinking Less of Yourself, It is Thinking of Yourself Less'**

Okay, enough cod-wisdom & jabber from me. Let's crack on!
...

# Deploy & Run Django on the App Engine standard environment
https://cloud.google.com/build/docs/deploying-builds/deploy-appengine

## GOAL
Deploy a dJango backend to Google App Engine.

## PURPOSE
Learn the mechanism(s) required to deploy an existing dJango app to Google App Engine.

### View Example
You can find an example of the fully deployed app here:

<span style="color: #ff00ff"># < link to live app></span>

Check it out.

```
# Open the deployed website:
gcloud app browse
-
https://pfolio-backend-2.ew.r.appspot.com/
-
# Alternatively, display the URL and open manually:
gcloud app describe --format "value(defaultHostname)"

```

Next time you see this - YOU will have deployed your own version of the same.

--- 

# create/configure app engine
## new project
```
'New Project'
Project: h-pfolio
ID: h-pfolio

```

## app environment
```

```

## enable billing

### WARNING!!: 
Be VERY careful that you understand the cost implications of the resources we will be using.

If new to Google Cloud then you are able to sign up with a generous 'FREE' period along with with a very useful Credit towards the use of PAID resources.

 MORE INFO: 
 - https://cloud.google.com/billing/docs
 - https://cloud.google.com/billing/docs/how-to/verify-billing-enabled
 
 ENABLE:
 - https://console.cloud.google.com/billing/linkedaccount?project=pfolio-backend-2

## 'local' install
Firstly, setup using local install.

This uses local assets for DB & Static files.

We will then migrate both to using Google Cloud assets.

# project deploy VARS
```
source ./config/.env-vars

```

# app
```
# Clone app

mkdir heidless-pfolio
cd heidless-pfolio

# github load raw Application
git clone https://github.com/heidless-stillwater/pfolio-backend.git

cd ./pfolio-backend

python -m venv venv
source ./venv/bin/activate

#################
# requirements.txt
pip install -r ./requirements.txt

#######
# UTILS
#
sudo apt install python3.12-venv

```

### LOCAL DB 
```
##############################
# Configure APP DB environment
#
sudo su postges
psql

# create db
CREATE DATABASE  portfolio;

# create user
CREATE USER arjuna11 WITH PASSWORD 'havana11';

# 'postgres' password
ALTER USER postgres with encrypted password 'postgres';

#####
UTILS
#
sudo systemctl status postgresql.service
sudo systemctl restart postgresql.service

```


## inialize cli (vsCode)
```
#PROJECT: heidless-portfolio-3

# home dir
cd <APP DIR>

# initialize to ensure working with correct project & ID
gcloud init

'Create a new configuration'
${GCP_PROJECT}
heidlessemail08@gmail.com
${GCP_PROJECT}

# initialise App Engine
gcloud app create --region=europe-west2 

```

## Download Cloud SQL Auth proxy to connect to Cloud SQL from your local machine
```
# init proxy
curl -o cloud-sql-proxy https://storage.googleapis.com/cloud-sql-connectors/cloud-sql-proxy/v2.7.0/cloud-sql-proxy.linux.amd64

chmod +x cloud-sql-proxy

```
## PostgreSQL instance
```
##################################################
# if on LOCAL
# restart server
sudo service postgresql restart

##################################################
# if on GAE

# init env
gcloud init

# script to handle ALL DB creation & config
./pfolio-CREATE-INSTANCE

# ensure correct project
gcloud config set project ${GCP_PROJECT_ID}

# initialise DB Instance (takes some time  - take a break and let it process)

#######################
# INSTANCE
#
gcloud sql instances create ${GCP_INSTANCE} \
    --project ${GCP_PROJECT_ID} \
    --database-version ${GCP_DB_VERSION} \
    --tier ${GCP_TIER} \
    --region ${GCP_REGION}
	
gcloud sql databases create ${GCP_DB_NAME} \
    --instance ${GCP_INSTANCE}

gcloud sql users create ${GCP_DB_USER} \
--instance ${GCP_INSTANCE} \
--password ${GCP_INSTANCE_ROOT_PWD}

# check status of instance
gcloud sql instances describe --project ${GCP_PROJECT_ID} ${GCP_INSTANCE}

#######################
# POSTGRES password
#
gcloud sql users set-password postgres \
--host=% \
--instance=${GCP_INSTANCE} \
--password=postgres

#########################
# CONNECT to new instance
#
gcloud sql connect ${GCP_INSTANCE} --database ${GCP_DB_NAME} --user=postgres --quiet

# DB URL
# assemble link from the above info
postgres://<USER>:<PWD>@//cloudsql/<PROJECT ID>:<REGION>:<INSTANCE>/<DB>
--
postgres://pfolio-user-0:GJaUUsg_%RYnXVCB@//cloudsql/heidless-portfolio-4:europe-west2:pfolio-instance-0/pfolio-db-0

--

#######
# UTILS
#
sudo ufw allow 5432

### If need to REBUILD SQL Instance

# disable deletion protection
https://console.cloud.google.com/sql/instances/pf-pg-instance-0/edit?project=xenon-pier-390513&supportedpurview=project
EDIT->DeletionProtection

# delete instance - if it exisrs
gcloud sql instances delete pf-pg-instance-0
##############################################################
```

## storage bucket
```

gcloud config set project ${GCP_PROJECT}

#gcloud storage ls

gcloud storage buckets create gs://${GCP_BUCKET} --location=${GCP_REGION}

###########
# UTILS
#
# initialise BUCKET
#gsutil mb -l europe-west2 gs://pfolio-bucket-0

```

### service account(s)
```
PROJECT: pfolio-backend-3
ID: pfolio-backend-3

# 'IAM & ADMIN'->Service Accounts
h-pfolio@appspot.gserviceaccount.com
-
edit principal

-

# add ROLES to allow access to DB & 'secrets'
--
Secret Manager Secret Accessor
Cloud SQL Admin
Storage Admin
--
```


generate & install KEY file
```
##################################
# uncompress CREDENTIALS_ENCRYPTED
tar xvf backend_credentials.gz

# compress CREDENTIALS_ENCRYPTED
tar czf backend_credentials.gz CREDENTIALS_ENCRYPTED
rm -rf CREDENTIALS_ENCRYPTED

# generate new KEY
'IAM & ADMIN'->Service Accounts->'3 dots'->Manage Keys
'ADD KEY'->JSON

# Download & install json file

# copy to local project/app directory'
/home/heidless/projects/backend-live
```

## secrets setup
```

#######################################
# setup local environment - TEMPORARILY

cd config

#DATABASE_URL=postgres://pfolio-user-0:GJaUUsg_%RYnXVCB@//cloudsql/heidless-portfolio-3:europe-west2:pfolio-instance-0/pfolio-db-0 > .env

#DATABASE_URL=postgres://${GCP_DB_USER}:${HEIDLESS_DB_USER_PWD}@//cloudsql/${GCP_PROJECT}:${GCP_REGION}:${GCP_INSTANCE}/${GCP_DB_NAME} > .env

#echo DATABASE_URL=postgres://pfolio-user-0:GJaUUsg_%RYnXVCB@//cloudsql/heidless-portfolio-3:europe-west2:pfolio-instance-0/pfolio-db-0 > .env

echo DATABASE_URL=postgres://${GCP_DB_USER}:${GCP_INSTANCE_ROOT_PWD}@//cloudsql/${GCP_PROJECT}:${GCP_REGION}:${GCP_INSTANCE}/${GCP_DB_NAME} > .env

echo SECRET_KEY=$(cat /dev/urandom | LC_ALL=C tr -dc '[:alpha:]'| fold -w 50 | head -n1) >> .env

echo GS_BUCKET_NAME=${GCP_BUCKET} >> .env

echo FRONTEND_URL=${GCP_FRONTEND_URL} >> .env

#########################
# store in secret manager

gcloud secrets create ${GCP_SECRET_NAME} --data-file .env

# Grant access to the secret to the service account
gcloud secrets add-iam-policy-binding ${GCP_SECRET_NAME} \
    --member serviceAccount:${GCP_PROJECT}@appspot.gserviceaccount.com \
    --role roles/secretmanager.secretAccessor
		
# test - retrieve content of 'pfolio_settings'
gcloud secrets versions access latest --secret ${GCP_SECRET_NAME} && echo ""

#######
# UTILS
#
# when reseting - delete SECRET
gcloud secrets delete ${GCP_SECRET_NAME}

gcloud secrets describe ${GCP_SECRET_NAME}


```

## WARNING

Now you have setup your Cloud Secrets based on you .env file you now have 2 SOURCES OF TRUTH.

Not good.

In the heat of battle you'll likely find yourself modifying one while using the other.

WASTE of Time & Effort. Demoralising. 

My recommendation at this stage is to disable your local .env. to force use of your shiny new Google Secret (django_settings).

DELETE or RENAME app/config/.env file. i.e. ``rm ./config/.env``

## revise secrets
RECOMMENDATION: When debugging, only change ONE thing at a time - then test.
- Otherwise you will NOT KNOW the impact of any individual change or their combination
- These settings are foundational to your app.
- BOTH accuracy & understanding is vital
- Rushing will cost you significantly MORE TIME
- Make One Chane. Test It. Repeat.

It's likely that you will be refining & modifying the definitions in your SECRETS settings i.e. 'django_settings'.
- You'll likely want to test different settings - particularly when refining/debugging/hardening
- This will involve your REPLACING  you current SECRETS - i.e. django_settings.

```
################### REBUILD IF NEEDED ####################
# ensure you are in the right PROJECT
gcloud config set project h_pfolio

edit the config/.env file as needed.

# remove existing SECRET
gcloud secrets delete pfolio_settings

# create new SECRET file
gcloud secrets create pfolio_settings --data-file .env

# Grant access to the secret to the App Engine standard service account
gcloud secrets add-iam-policy-binding pfolio_settings \
    --member serviceAccount:h-pfolio@appspot.gserviceaccount.com\
    --role roles/secretmanager.secretAccessor

```


# Run the app via your local computer
We will use your local installation but hook into the Google Cloud Resources.

The Goal here is to switch from using local definitions to using those you have configured on App Engine.

A key one is where your SECRETS are stored.

The settings.py priorities local '.env' over your Google Secrets.

AFTER we've updated the following in settings.py we will be 'diasbling' local definitions.

### settings.py
Need to link to the 'key' file you downloaded earlier.
set GS_CREDENTIALS
```
GS_CREDENTIALS = service_account.Credentials.from_service_account_file(
    os.path.join(BASE_DIR, 'config/h-pfolio-5bed07c9d6bc')
)

GS_BUCKET_NAME = 'heidless-pfolio-bucket-4'

```

set STATIC_URL
```
STATIC_URL = 'https://storage.cloud.google.com/heidless-pfolio-bucket-4/'

```

disable local settings to force use of Google Secrets
```
mv config/.env config/.env-orig
```

### on localhost - run in dedicated shell - <span style="color: #ff807f">DELETE!!!</span>
```
# configure access
# https://console.cloud.google.com/sql/instances/pf-pg-instance-0/connections/networking?project=xenon-pier-390513

# pfolio-backend-db-instance-0 -> connections -> add network
-
rob-laptop
2.99.25.205
-
# check if can access DB directly
gcloud sql connect pfolio-backend-instance-0 --database pfolio-backend-db-0 --user=postgres --quiet
```

## test cloud DB connection
```
# test - retrieve content of 'pfolio_settings'
gcloud secrets versions access latest --secret ${HEIDLESS_SECRET} && echo ""

# login to instanece as 'postgres' password='postgres'
gcloud sql connect ${HEIDLESS_DB_INSTANCE} --database ${HEIDLESS_DB_DATABASE} --user=postgres --quiet

##########
# utils
#
#sudo systemctl status postgresql.service
#sudo systemctl restart postgresql.service
#gcloud secrets versions access latest --secret ${HEIDLESS_SECRET} && echo

```

### enable cloud proxy
```

./cloud-sql-proxy --credentials-file config/${GCP_CREDENTIALS} ${GCP_PROJECT}:${GCP_REGION}:${GCP_INSTANCE}

# kill & restart - IF address already in use
#sudo lsof -i -P -n | grep LISTEN
sudo lsof -i :5432
sudo kill -9 458


```

### upload STATIC files
```
cd ./BACKUPS/08-01

echo HEIDLESS_BUCKET: ${HEIDLESS_BUCKET}

########
# images
#
gcloud storage cp --recursive images gs://${HEIDLESS_BUCKET}

#######
# icons
#
gcloud storage cp --recursive icons gs://${HEIDLESS_BUCKET}


```

### restore BACKUP - if needed
```
# set permissoons on INSTANCE service account
echo ' '
echo '#################################'
echo '#### SET PERMISSIONS - objectAdmin & legacyBucketWriter'
echo HEIDLESS_DB_INSTANCE: ${HEIDLESS_DB_INSTANCE}
export SQL_SVC_ACC=`gcloud sql instances describe ${HEIDLESS_DB_INSTANCE} | grep serviceAccountEmailAddress`
echo SQL_SVC_ACC: ${SQL_SVC_ACC}

# pfolio-backends
export DB_SVC_ACCOUNT=p865665966029-tikb0v@gcp-sa-cloud-sql.iam.gserviceaccount.com

# set PERMISSION on Svc Account
echo DB_SVC_ACCOUNT: ${DB_SVC_ACCOUNT}
echo HEIDLESS_BUCKET: ${HEIDLESS_BUCKET}

echo ' '
echo 'objectAdmin'
gsutil iam ch serviceAccount:${DB_SVC_ACCOUNT}:objectAdmin gs://${HEIDLESS_BUCKET}
echo gsutil iam ch serviceAccount:${DB_SVC_ACCOUNT}:objectAdmin gs://${HEIDLESS_BUCKET}


#############
# Take Backup
#
timestamp=`date +%s`
MSG=1st-ATTEMPT
GCP_BACKUP_PREFIX=backup-prefix

echo PROJECT: ${HEIDLESS_PROJECT_ID}
echo INSTANCE: ${HEIDLESS_DB_INSTANCE}
echo DB: ${HEIDLESS_DB_DATABASE}
echo USER: ${HEIDLESS_DB_USER}
echo BUCKET: $HEIDLESS_BUCKET

export BK_PREFIX=${GCP_BACKUP_PREFIX}
echo BK_PREFIX: ${BK_PREFIX}

export BK_COMMENT='-BACKUP-TEST-'
echo COMMENT: $BK_COMMENT

export BK_TIMESTAMP=`date +%s`
echo TIMESTAMP: $BK_TIMESTAMP

#export GCP_FILE=${GCP_DB_NAME}-${BK_COMMENT}-${BK_TIMESTAMP}.gz

export GCP_FILE=${BK_PREFIX}-${BK_COMMENT}-${BK_TIMESTAMP}.gz
echo FILE: ${GCP_FILE}

echo ' '

echo '######################'
echo 'BACKUP DB'
echo BK_PREFIX: ${BK_PREFIX}
echo HEIDLESS_DB_INSTANCE: ${HEIDLESS_DB_INSTANCE}
echo HEIDLESS_BUCKET: ${HEIDLESS_BUCKET}
echo GCP_FILE: ${GCP_FILE}
echo HEIDLESS_DB_DATABASE: ${HEIDLESS_DB_DATABASE}
echo ' '

gcloud sql export sql ${HEIDLESS_DB_INSTANCE} gs://${HEIDLESS_BUCKET}/backups/${GCP_FILE}    \
--database=${HEIDLESS_DB_DATABASE}	 \
--offload

#################
# DOWNLOAD BACKUP
# cd <BACKUP DIR>

echo HEIDLESS_BUCKET: ${HEIDLESS_BUCKET}
echo GCP_FILE: ${GCP_FILE}
gcloud storage cp gs://${HEIDLESS_BUCKET}/backups/${GCP_FILE} .

###############
# UPLOAD BACKUP
#
source ../../config/.env-vars

GCP_FILE=heidless-pfolio-6-backend-instance-heidless-pfolio-6-backend-db-BACKUP-08-01--1722508280.gz

echo HEIDLESS_BUCKET: ${HEIDLESS_BUCKET}
echo GCP_FILE: ${GCP_FILE}
gcloud storage cp ${GCP_FILE} gs://${HEIDLESS_BUCKET}/backups/${GCP_FILE}

##############
# RE-CREATE DB
# pq: database "photo-app-0-db-0" is being accessed by other users
# CLOSE ALL TABS ACCESSING APP
#
echo 're-create DB'
echo HEIDLESS_DB_DATABASE: ${HEIDLESS_DB_DATABASE}
echo HEIDLESS_DB_INSTANCE: ${HEIDLESS_DB_INSTANCE}
echo ' '
gcloud sql databases delete ${HEIDLESS_DB_DATABASE} \
    --instance ${HEIDLESS_DB_INSTANCE}

gcloud sql databases create ${HEIDLESS_DB_DATABASE} \
    --instance ${HEIDLESS_DB_INSTANCE}

################
# RESTORE BACKUP
#
echo HEIDLESS_DB_INSTANCE: ${HEIDLESS_DB_INSTANCE}
echo HEIDLESS_DB_DATABASE: ${HEIDLESS_DB_DATABASE}
echo HEIDLESS_BUCKET: ${HEIDLESS_BUCKET}
echo GCP_FILE: ${GCP_FILE}
echo ' '
gcloud sql import sql ${HEIDLESS_DB_INSTANCE} gs://${HEIDLESS_BUCKET}/backups/${GCP_FILE} \
--database=${HEIDLESS_DB_DATABASE}



```


### init & run backend
```
###########
# using GCP
export GOOGLE_CLOUD_PROJECT=h-pfolio
export USE_CLOUD_SQL_AUTH_PROXY=True

source ./venv/bin/activate

./manage.py makemigrations

./manage.py migrate

python manage.py createsuperuser
-
heidless
rob.lockhart@yahoo.co.uk
sdfsdgasgTHW66GDGdfdff
-

# mkdir ./build/static
./manage.py collectstatic

############
RUN
#
python manage.py runserver 8000

#################
# view site/admin
http://localhost:8000/admin

##########
# UTILS
#
# set executable version
sudo apt update && sudo apt upgrade
sudo apt install python-is-python3

```
---
---
qqqqq

# Deploy the app to the App Engine standard environment

```
# initialze app - creates 'app.yaml'
vi app/app.yaml
--
runtime: python39
env: standard
entrypoint: gunicorn -b :$PORT config.wsgi:application

handlers:
- url: /.*
  script: auto

runtime_config:
  python_version: 3
--

# deploy to app engine
gcloud app deploy

# view ADMIN 
https://pfolio-backend-2.ew.r.appspot.com/admin

### if ISSUES then app can be reset
===================
gcloud beta app repair
==================

# display APP URL
gcloud app describe --format "value(defaultHostname)"
-
https://pfolio-backend-2.ew.r.appspot.com
-

# monitor logs
gcloud app logs tail -s default
-
target url: https://pfolio-backend-2.ew.r.appspot.com
target service account: pfolio-backend-2@appspot.gserviceaccount.com
-

# Open app.yaml and update the value of APPENGINE_URL with your deployed URL:
vi app.yaml
-
env_variables:
	APPENGINE_URL: https://pfolio-backend-2.ew.r.appspot.com/

-

# re-deploy
gcloud app deploy

```

## Updating the application
To update your application, make changes to the code, then run the gcloud app deploy command again.

# pgadmin

## Create pgadmin dir
```
mkdir <project root>/pgadmin
cd !$
```

## Dockerfile
Create
```
FROM dpage/pgadmin4

ENV PGADMIN_DEFAULT_EMAIL=rob.lockhart@yahoo.co.uk

ENV PGADMIN_DEFAULT_PASSWORD=havana11

ENV PGADMIN_LISTEN_PORT=8080
```


Builld
```
gcloud builds submit --tag=gcr.io/pfolio-backend-deploy-2/pgadmin4
```

Deploy
```
gcloud run deploy --image=gcr.io/pfolio-backend-deploy-2/pgadmin4 --platform=managed
```

Access
https://pgadmin4-lt22gupljq-ew.a.run.app



Very slow to log in
- start: 10:49
- end: 




Connection issues checklist
	Connecting
		Private IP
			Have you enabled the Service Networking API for your project?
				enabled
			Are you using a Shared VPC?
				no
			Does your user or service account have the required IAM permissions to manage a private services access connection?
			Is private services access connection configured for your project?
			Did you allocate an IP address range for the private connection?
			Did your allocated IP address ranges contain at least a /24 space for every region where you are planning to create mysql instances?
			If you are specifying an allocated IP address range for your mysql instances, does the range contain at least a /24 space for every region where you are planning to create mysql instances in this range?
			Is the private connection created?
			If the private connection was changed, were the vpc-peerings updated?
			Do the VPC logs indicate any errors?
			Is your source machine's IP a non-RFC 1918 address?
		Public IP
			Is your source IP listed as an authorized network?
				Yes
			Are SSL/TLS certificates required?
				Don't believe so
			Does your user or service account have the required IAM permissions to connect to a Cloud SQL instance?
				Full Admin access: Cloud SQL Client
	Authorizing		
	Cloud SQL Auth proxy
		Is the Cloud SQL Auth proxy up to date?
			Yes
		Is the Cloud SQL Auth proxy running?
			Yes
		Is the instance connection name formed correctly in the Cloud SQL Auth proxy connection command?
			yes
		Have you checked the Cloud SQL Auth proxy output? Pipe the output to a file, or watch the Cloud Shell terminal where you started the Cloud SQL Auth proxy.
			Starts fine. Does not appear to being called though
		Does your user or service account have the required IAM permissions to connect to a Cloud SQL instance?
			Yes
		Have you enabled the Cloud SQL Admin API for your project?
			Yes
		If you have an outbound firewall policy, make sure it allows connections to port 3307 on the target Cloud SQL instance.
			???
		If you are connecting using UNIX domain sockets, confirm that the sockets were created by listing the directory specified with the -dir when you started the Cloud SQL Auth proxy.
Cloud SQL connectors and language-specific code
Is the connection string formed correctly?
Have you compared your code with the sample code for your programming language?
Are you using a runtime or framework for which we don't have sample code?
If so, have you looked to the community for relevant reference material?
Self-managed SSL/TLS certificates
Is the client certificate installed on the source machine?
Is the client certificate spelled correctly in the connection arguments?
Is the client certificate still valid?
Are you getting errors when connecting using SSL?
Is the server certificate still valid?
Authorized networks
Is the source IP address included?
Are you using a non-RFC 1918 IP address?
Are you using an unsupported IP address?
Connection failures
Are you authorized to connect?
Are you seeing connection limit errors?
Is your application closing connections properly?
Authenticating
Native database authentication (username/password)
Are you seeing access denied errors?
Are the username and password correct?
IAM database authentication
Have you enabled the cloudsql.iam_authentication flag on your instance?
Did you add a policy binding for the account?
Are you using the Cloud SQL Auth proxy with the -enable_iam_login or an Oauth 2.0 token as the database password?
If using a service account, are you using the shortened email name?
Learn more about IAM database authentication in PostgreSQL.
