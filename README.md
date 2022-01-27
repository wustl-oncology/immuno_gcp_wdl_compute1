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

### Set some Google Cloud and other environment variables

```
export PROJECT=griffith-lab
export GCS_BUCKET=griffith-lab-test-immuno-pipeline
export WORKING_BASE=/storage1/fs1/mgriffit/Active/griffithlab/gcp_wdl_test
export TUTORIAL_GIT=/home/mgriffit/git/immuno_gcp_wdl
```

## Local setup

### First create a working directory on your local system

```bash
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

### Login to GCP and set the desired project

```
gcloud auth login
gcloud config set project $PROJECT
```

### Set up cloud account and bucket

```
cd $WORKING_BASE/git
bash cloud-workflows/manual-workflows/resources.sh init-project --project griffith-lab --bucket griffith-lab-test-immuno-pipeline
```

### Gather input data and reference files to your local system
Create a directory for YAML files and create one for the desired pipeline that points to the location of input files on your local system

```
cd $WORKING_BASE
mkdir yamls
cd yamls
```

### Setup input data and reference files

Download RAW data for hcc1395
```
mkdir -p $WORKING_BASE/raw_data/hcc1395
cd $WORKING_BASE/raw_data/hcc1395
wget http://genomedata.org/pmbio-workshop/fastqs/all/Exome_Norm.tar
wget http://genomedata.org/pmbio-workshop/fastqs/all/Exome_Tumor.tar
wget http://genomedata.org/pmbio-workshop/fastqs/all/RNAseq_Tumor.tar
tar -xvf Exome_Norm.tar Exome_Tumor.tar RNAseq_Tumor.tar
rm -f Exome_Norm.tar Exome_Tumor.tar RNAseq_Tumor.tar

```


Setup yaml files for an example run
```
cp $TUTORIAL_GIT/example_yamls/hcc1395_immuno_local.yaml $WORKING_BASE/yamls/
```



### Stage input files to cloud bucket

Start an interactive docker session capable of running the "cloudize" scripts
```
bsub -Is -q general-interactive -G $GROUP -a "docker(jackmaruska/cloudize-workflow:latest)" /bin/bash
```

Attempt to cloudize your workflow and inputs
```
export WORKFLOW_DEFINITION=$WORKING_BASE/git/analysis-wdls/definitions/immuno.wdl
export LOCAL_INPUT=$WORKING_BASE/yamls/somatic_exome_hcc1395_local.yaml
export CLOUD_INPUT=$WORKING_BASE/yamls/somatic_exome_hcc1395_cloud.yaml
python3 /opt/scripts/cloudize-workflow.py $GCS_BUCKET $WORKFLOW_DEFINITION $LOCAL_INPUT --output=$CLOUD_INPUT
```



