# immuno_gcp_wdl
Tutorial to run immuno.wdl on Google Cloud

## Source of instructions
https://github.com/griffithlab/cloud-workflows/tree/main/manual-workflows

## Prerequisites
- google-cloud-sdk
- git

## Local setup

### First create a working directory on your local system

```bash
export WORKING_BASE=/storage1/fs1/mgriffit/Active/griffithlab/gcp_wdl_test
mkdir $WORKING_BASE
cd $WORKING_BASE
```

### Clone git repositories that have the workflows (pipelines) and scripts to help run them

mkdir git
cd git
git clone git@github.com:griffithlab/cloud-workflows.git
git clone git@github.com:griffithlab/analysis-wdls.git

### Set some Google Cloud environment variables
export PROJECT=griffith-lab
export GCS_BUCKET=griffith-lab-test-immuno-pipeline

### Login to GCP and set the desired project
gcloud auth login
gcloud config set project $PROJECT

### Set up cloud account and bucket
sh cloud-workflows/gms/resources.sh init-project --project $PROJECT --bucket $GCS_BUCKET

### Gather input data and reference files to your local system
...



### Stage input files to cloud bucket
bsub -Is -q general-interactive -G $GROUP -a "docker(jackmaruska/cloudize-workflow:latest)" /bin/bash

export WORKFLOW_DEFINITION=$WORKING_BASE/git/analysis-wdls/definitions/immuno.wdl
export LOCAL_INPUT=/storage1/fs1/mgriffit/Active/griffithlab/adhoc/somatic_exome_local.yaml
export CLOUD_INPUT=$WORKING_BASE/somatic_exome_cloud.yaml
python3 /opt/scripts/cloudize-workflow.py $GCS_BUCKET $WORKFLOW_DEFINITION $LOCAL_INPUT --output=$CLOUD_INPUT





