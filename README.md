# immuno_gcp_wdl
A tutorial to demonstrate how to run the WASHU immunogenomics pipeline (immuno.wdl) on Google Cloud.
This will be done by first setting up the workflow definitions, input data and reference files and a 
YAML config file on a user's local system. The user will also set up a Google Cloud environment
and all the inputs will be staged to this environment.  Next Cromwell will be used to execute the pipeline
using the specified input and reference files, and finally the results will be pulled back to the local system
and cloud resources will be cleaned up. 

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

```
mkdir git
cd git
git clone git@github.com:griffithlab/cloud-workflows.git
git clone git@github.com:griffithlab/analysis-wdls.git
```

### Set some Google Cloud environment variables

```
export PROJECT=griffith-lab
export GCS_BUCKET=griffith-lab-test-immuno-pipeline
```

### Login to GCP and set the desired project

```
gcloud auth login
gcloud config set project $PROJECT
```

### Set up cloud account and bucket

```
sh cloud-workflows/gms/resources.sh init-project --project $PROJECT --bucket $GCS_BUCKET
```

### Gather input data and reference files to your local system
Create a directory for YAML files and create one for the desired pipeline that points to the location of input files on your local system

```
cd $WORKING_BASE
mkdir yamls
cd yamls
```


### Stage input files to cloud bucket

Start an interactive docker session capable of running the "cloudize" scripts
```
bsub -Is -q general-interactive -G $GROUP -a "docker(jackmaruska/cloudize-workflow:latest)" /bin/bash
```

Attempt to cloudize your workflow and inputs
```
export WORKFLOW_DEFINITION=$WORKING_BASE/git/analysis-wdls/definitions/immuno.wdl
export LOCAL_INPUT=$WORKING_BASE/yamls/somatic_exome_local.yaml
export CLOUD_INPUT=$WORKING_BASE/yamls/somatic_exome_cloud.yaml
python3 /opt/scripts/cloudize-workflow.py $GCS_BUCKET $WORKFLOW_DEFINITION $LOCAL_INPUT --output=$CLOUD_INPUT
```



