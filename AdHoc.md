# Simple tutorial for simple ad hoc analysis within Docker on a Google VM

## Preamble
The following tutorial explains how to do a project setup that will configure Google credentials and security settings for someone working at WASHU. However, it is designed to work from your personal computer and not rely on compute1/storage1 access. The example analysis in this tutorial will download some protein sequences and use pVACtools to perform neoantigen predictions on these sequences.

### Prerequisites
- google-cloud-sdk (for things like `gcloud` and `gsutil`)
- git

## Step-by-step instructions

### Set some Google Cloud and other environment variables
The following environment variables are used merely for convenience and should be customized to produce intuitive labeling for your own analysis:
```bash
export GCS_PROJECT=griffith-lab
export GCS_SERVICE_ACCOUNT=cromwell-server@$GCS_PROJECT.iam.gserviceaccount.com
export GCS_BUCKET_NAME=griffith-lab-malachi-adhoc
export GCS_BUCKET_PATH=gs://griffith-lab-malachi-adhoc
export GCS_INSTANCE_NAME=malachi-adhoc
export WORKING_BASE=~/gcp_adhoc
```

## Local setup

### First create a working directory on your local system
The following directory on the local system will contain: (a) a git repository for this tutorial, (b) git repository for tools that help provision and manage our workflow runs on the cloud, (c) example data that we will download for this tutorial.
```bash
mkdir $WORKING_BASE
cd $WORKING_BASE
```


### Clone git repositories that have the workflows (pipelines) and scripts to help run them
The following repositories contain: this tutorial (immuno_gcp_wdl) and tools for running these on the cloud (cloud-workflows). Note that the command below will clone the main branch of each repo.

```bash
cd $WORKING_BASE
mkdir git
cd git
git clone git@github.com:griffithlab/immuno_gcp_wdl_compute1.git
git clone git@github.com:griffithlab/cloud-workflows.git
```






### Use `gget` to obtain test sequences


### Use pVACbind to perform neoantigen analysis on the protein sequences obtained




