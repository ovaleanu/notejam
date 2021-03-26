# GitOps for Notejam on Google Cloud Platform

This document provides a Kubernetes cloud native infrastructure on Google Cloud Platform and GitOps style for a monolithic web application. For this use case, we will use a Python/Flask implementation of the [Notejam](https://github.com/komarserjio/notejam) sample application.

### Current Architecture

Notejam is currently build as monolithic containing built-in webserver and SQLite database.

![](https://github.com/ovaleanujnpr/notejam/blob/master/images/notejam.png)

### Requirements

- The Application must serve variable amount of traffic. Most users active during business hours. Traffic could be 4 times more tan usual sometimes.
_The Application will be deployed on a Google Kubernetes Engine. The pods running the application will be able to scale dynamically based on the inbound web requests._

- Data should be preserved and available for 3 years.
_Automated backups and point-in-time recovery are enabled and we will store Cloud SQL database exports on Cloud Storage using Cloud Scheduler._

- Continuity in case of datacentre failures.
_We will use Google Kubernetes Engine and MySQL DB HA Cloud SQL instance with a FailOver replica. There's also set up an automatic daily backups._

- The Service must be able to be migrated to different regions in case of emergency.
_We will not cover this use case in this example, but this is possible migrating the GKE cluster to another region and Cloud SQL replication for the database._

- More than 100 developers needs to be able to deliver new versions of the app continuously without downtime.
_We will use Google Cloud Build CI/CD pipeline integrated to GitHub to test and deploy the app automatically to Google Kubernetes Engine._

- Provision separated environments to support their development process for development, testing, production in the near future.
_We will create the environment for production. Another two environments for development ansd testing can be created similar. Each environment should be run in it's own Google Cloud Project to ensure isolation of resources._

- Collection of relevant metrics and logs from the infrastructure.
_Strackdriver will be used for monitoring and also for consolidating all the logs from application pods running on Google Kubernetes Engine and Cloud SQL._


### Diagram



To achieve all these described above the following tasks need to be performed:

- Install and configure gcloud on local workstation or use Google Cloud Shell
- Create a project
- Enable the APIs required for the project
- Create a Virtual Private Network
- Create a Google Kubernetes Engine Cluster
- Create a MySQL instance on Cloud SQL
- Create or use and existing git repo to upload the application
- Create a container image and a CI pipeline with Cloud Build. Store the image in Container Registry
- Create CD pipeline with Cloud Build for automatic deployment
- Build a monitoring dashboard on Stackdriver for application pods and Cloud SQL


### Github account and application repo

A GitHub account is needed to fork of the application at https://github.com/ovaleanujnpr/notejam.

### Google Cloud Platform prerequisites

- [Create a Google Cloud Platform account](https://console.cloud.google.com/freetrial)
- [Create a Google Cloud Billing account](https://cloud.google.com/billing/docs/how-to/manage-billing-account)
- [Create a Google Group for your developers](https://groups.google.com/my-groups)

#### Setup the environment
```
git clone https://github.com/ovaleanujnpr/notejam projects/
cd projects
```

Create a .env file based on the provided .env.example as a guide and source the file.
```
cp .env.example .env
vim .env
source .env
```

#### Create configuration and authenticate to GCP
```
gcloud config configurations create $PROJECT_NAME
gcloud auth login $ACCOUNT --update-adc
```

#### Create a project and link to billing acccount
```
gcloud projects create $PROJECT_ID --name=$PROJECT_NAME --set-as-default
gcloud beta billing projects link $PROJECT_ID --billing-account=$BILLING_ACCOUNT
```

#### Enable the APIs services
```
gcloud services enable \
 --project=${PROJECT_ID} \
 container.googleapis.com \
 gkehub.googleapis.com \
 iamcredentials.googleapis.com \
 monitoring.googleapis.com \
 logging.googleapis.com \
 sqladmin.googleapis.com \
 cloudbuild.googleapis.com \
 stackdriver.googleapis.com \
 cloudresourcemanager.googleapis.com
```

#### Create a DNS zone
```
gcloud dns managed-zones create ${ZONE} --dns-name="$DOMAIN." --description="$DOMAIN DNS zone"
gcloud dns managed-zones update ${ZONE} --dnssec-state on
```

#### Register a domain
```
cat <<EOF > contacts.yaml
allContacts:
  email: '$DOMAIN_EMAIL'
  phoneNumber: '$DOMAIN_PHONE'
  postalAddress:
    regionCode: '$DOMAIN_REGION'
    postalCode: '$DOMAIN_POSTAL'
    locality: '$DOMAIN_LOCALITY'
    addressLines: ['$DOMAIN_ADDRESS']
    recipients: ['$DOMAIN_RECIPIENT']
EOF
gcloud alpha domains registrations get-register-parameters $DOMAIN
gcloud alpha domains registrations register $DOMAIN \
--contact-data-from-file=contacts.yaml \
--contact-privacy=private-contact-data \
--cloud-dns-zone=${ZONE} \
--notices=hsts-preloaded \
--yearly-price="12.00 USD"
gcloud alpha domains registrations describe $DOMAIN
```

#### Create the Virtual Private Network the associated networking services
```
gcloud compute networks create ${NETWORK_PROD} --subnet-mode=auto
gcloud compute firewall-rules create ${NETWORK_PROD}-allow-internal --network ${NETWORK_PROD} --allow tcp,udp,icmp --source-ranges 10.154.0.0/20
gcloud compute firewall-rules create ${NETWORK_PROD}-allow-external --network ${NETWORK_PROD} --allow tcp:22,tcp:3306,icmp --source-ranges 0.0.0.0/0
gcloud compute addresses create google-managed-services-${NETWORK_PROD} --global --purpose=VPC_PEERING --prefix-length=24 --network=${NETWORK_PROD}
gcloud services vpc-peerings connect --service=servicenetworking.googleapis.com --ranges=google-managed-services-${NETWORK_PROD} --network=${NETWORK_PROD}
```

### Create the GKE cluster
```
gcloud container clusters create ${GKE_PROD} \
--region ${GCP_REGION} \
--machine-type "n1-standard-1" \
--cluster-version "1.18.15-gke.1501" \
--network "projects/${PROJECT_ID}/global/networks/${NETWORK_PROD}" \
--subnetwork "projects/${PROJECT_ID}/regions/${GCP_REGION}/subnetworks/${NETWORK_PROD}"
```

![](https://github.com/ovaleanujnpr/notejam/blob/master/images/notejam2.png)


#### Create the Git repositories in Cloud Source Repositories

```
cd ~/projects/notejam-flask-app
git init
git add .
git config --global user.email "${ACCOUNT}"
git config --global user.name "${account}"
git commit -m "Initial commit"
gcloud source repos create notejam-flask-app
git config --global credential.https://source.developers.google.com.helper gcloud.sh
git remote add google https://source.developers.google.com/p/${PROJECT_ID}/r/notejam-flask-app
git push --all google
```

#### Create a container image using Cloud Build

```
cd ~/projects/notejam-flask-app
COMMIT_ID="$(git rev-parse --short=7 HEAD)"
gcloud builds submit --tag="gcr.io/${PROJECT_ID}/notejam-flask:${COMMIT_ID}" .
```


#### Pipeline Architecture

![](https://github.com/ovaleanujnpr/notejam/blob/master/images/notejam4.png)

#### Create the CI pipeline

We will you configure Cloud Build to build the container image, and then push it to Container Registry. Pushing a new commit to Cloud Source Repositories automatically triggers this pipeline. The cloudbuild.yaml file included in the code is the pipeline's configuration.

```
cat cloudbuild.yaml

steps:
# This step builds the container image.
- name: 'gcr.io/cloud-builders/docker'
  id: Build
  args:
  - 'build'
  - '-t'
  - 'gcr.io/$PROJECT_ID/notejam-flask:$SHORT_SHA'
  - '.'

# This step pushes the image to Container Registry
# The PROJECT_ID and SHORT_SHA variables are automatically
# replaced by Cloud Build.
- name: 'gcr.io/cloud-builders/docker'
  id: Push
  args:
  - 'push'
  - 'gcr.io/$PROJECT_ID/notejam-flask:$SHORT_SHA'
```

1. Open the Cloud Build Triggers page.
2. Go to Triggers
3. Click Create trigger.
4. Fill out the following options:
- In the Name field, type hello-cloudbuild.
- Under Event, select Push to a branch.
- Under Source, select hello-cloudbuild-app as your Repository and ^master$ as your Branch.
- Under Build configuration, select Cloud Build configuration file.
- In the Cloud Build configuration file location field, type cloudbuild.yaml after the /.
5. Click Create to save your build trigger.

Push the application code to Cloud Source Repositories to trigger the CI pipeline in Cloud Build.

```
cd ~/projects/notejam-flask-app
git push google master
```

#### Configure MySQL database with Cloud SQL
The built-in db SQLite from the monolytic app of Notejam will be replaced with a MySQL instance running on Cloud SQL. To connect to MySQL instance, Google Cloud Platform provides CloudSQL proxy sidecar container. We will need to add this container to our pipeline.

```
docker pull gcr.io/cloudsql-docker/gce-proxy:1.17
docker tag gcr.io/cloudsql-docker/gce-proxy:1.17 gcr.io/${PROJECT_ID}/gce-proxy:1.17
docker push gcr.io/${PROJECT_ID}/gce-proxy:1.17
```

#### Create the MySQL instance

```
gcloud sql instances create ${DB_INSTANCE_NAME} \
--assign-ip \
--backup-start-time "03:00" \
--failover-replica-name "${DB_INSTANCE_NAME}-failover" \
--enable-bin-log \
--database-version=MYSQL_5_7 \
--region=europe-west2 \
```

![](https://github.com/ovaleanujnpr/notejam/blob/master/images/notejam3.png)


Set the root password

```
gcloud sql users set-password root \
--host=% \
--instance=${DB_INSTANCE_NAME} \
--password=${DB_ROOT_PASS} \
```

Set up MySQL service account, that will be used with CloudSQL Proxy in Kubernetes and bind the cloudsql.admin role to this service account

```
gcloud iam service-accounts create mysql-service-account \
--display-name "mysql service account"

gcloud projects add-iam-policy-binding notejam-project \
--member serviceAccount:mysql-service-account@notejam-project.iam.gserviceaccount.com \
--role roles/cloudsql.admin
```

Create the secret in Google Kubernetes Engine Cluster, that will be used in CloudSQL Proxy

```
kubectl create secret generic cloudsql-instance-credentials \
--from-file=credentials.json=./mysql-key.json
```

#### Create the CD pipeline
Cloud Build is also used for the continuous delivery pipeline. The pipeline applies the new version of the manifest to the Kubernetes cluster and, if successful, copies the manifest over to the production branch. This process has the following properties:

- The production branch is a history of the successful deployments.
- You have a view of successful and failed deployments in Cloud Build.
- We can rollback to any previous deployment by re-executing the corresponding build in Cloud Build. A rollback also updates the production branch to truthfully reflect the history of deployments.

First we need to grant Cloud Build Access to GKE

```
PROJECT_NUMBER="$(gcloud projects describe ${PROJECT_ID} --format='get(projectNumber)')"
gcloud projects add-iam-policy-binding ${PROJECT_NUMBER} \
    --member=serviceAccount:${PROJECT_NUMBER}@cloudbuild.gserviceaccount.com \
    --role=roles/container.developer
```

Initialise notejam-flask-k8s repository

```
cd ~/project/notejam-flask-k8s
git init
git checkout -b production
git add .
git commit -m "Create cloudbuild.yaml for deployment"

cat cloudbuild.yaml

steps:
# This step deploys the new version of our container image
# in the notejam-prod Kubernetes Engine cluster.
- name: 'gcr.io/cloud-builders/kubectl'
  id: Deploy
  args:
  - 'apply'
  - '-f'
  - 'notejam.yaml'
  env:
  - 'CLOUDSDK_COMPUTE_ZONE=europe-west2-c'
  - 'CLOUDSDK_CONTAINER_CLUSTER=notejam-prod'

# This step copies the applied manifest to the production branch
# The COMMIT_SHA variable is automatically
# replaced by Cloud Build.
- name: 'gcr.io/cloud-builders/git'
  id: Copy to production branch
  entrypoint: /bin/sh
  args:
  - '-c'
  - |
    set -x && \
    # Configure Git to create commits with Cloud Build's service account
    git config user.email $(gcloud auth list --filter=status:ACTIVE --format='value(account)') && \
    git fetch origin production && git checkout production && \
    git checkout $COMMIT_SHA notejam.yaml && \
    # Commit the notejam.yaml file with a descriptive commit message
    git commit -m "Manifest from commit $COMMIT_SHA
    $(git log --format=%B -n 1 $COMMIT_SHA)" && \
    # Push the changes back to Cloud Source Repository
    git push origin production


git push origin production
```

Grant the Source Repository Writer IAM role to the Cloud Build service account for the notejam-flask-k8s repository.

```
PROJECT_NUMBER="$(gcloud projects describe ${PROJECT_ID} \
    --format='get(projectNumber)')"
cat >/tmp/notejam-flask-k8s-policy.yaml <<EOF
bindings:
- members:
  - serviceAccount:${PROJECT_NUMBER}@cloudbuild.gserviceaccount.com
  role: roles/source.writer
EOF
gcloud source repos set-iam-policy \
    notejam-flask-k8s /tmp/notejam-flask-k8s-policy.yaml
```

Create the trigger for the continuous delivery pipeline

In this section, you configure Cloud Build to be triggered by a push to the candidate branch of the hello-cloudbuild-env repository.

1. Open the Triggers page of Cloud Build.
2. Go to Triggers
3. Click Create trigger.
4. Fill out the following options:

- In the Name field, type notejam-flask-deploy.
- Under Event, select Push to a branch.
- Under Source, select notejam-flask-k8s as your Repository
- Under Configuration, select Cloud Build configuration file (yaml or json).
- In the Cloud Build configuration file location field, type cloudbuild.yaml after the /.
5. Click Create.
