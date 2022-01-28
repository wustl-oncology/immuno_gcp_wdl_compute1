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

### Interacting with Google buckets from your local system
Note that you can use this docker image to access `gsutil` for exploration of your google storage: docker(google/cloud-sdk)

### Set some Google Cloud and other environment variables

```
export GROUP=compute-oncology
export PROJECT=griffith-lab
export GCS_BUCKET_NAME=griffith-lab-test-immuno-pipeline
export GCS_BUCKET_PATH=gs://griffith-lab-test-immuno-pipeline
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
Run the following command and make note of the "Service Account" returned (e.g. "cromwell-server@griffith-lab.iam.gserviceaccount.com")

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
bsub -Is -q oncology-interactive -G $GROUP -a "docker(jackmaruska/cloudize-workflow:latest)" /bin/bash
```

Attempt to cloudize your workflow and inputs
```
export WORKFLOW_DEFINITION=$WORKING_BASE/git/analysis-wdls/definitions/immuno.wdl
export LOCAL_YAML=hcc1395_immuno_local.yaml
export CLOUD_YAML=hcc1395_immuno_cloud.yaml
python3 /opt/scripts/cloudize-workflow.py $GCS_BUCKET_NAME $WORKFLOW_DEFINITION $WORKING_BASE/yamls/$LOCAL_YAML --output=$WORKING_BASE/yamls/$CLOUD_YAML
```

### Start a Google VM that will run cromwell and orchestrate completion of the workflow

```
export INSTANCE_NAME=mg-immuno-test
export SERVER_ACCOUNT=cromwell-server@griffith-lab.iam.gserviceaccount.com

cd $WORKING_BASE/git
bash cloud-workflows/manual-workflows/start.sh $INSTANCE_NAME --server-account $SERVER_ACCOUNT

```

### Log into the VM and check status 

After logging in, use journalctl to see if the instance start up has completed, and cromwell launch has completed.

For details on how to recognize whether these processes have completed refer: [here](https://github.com/griffithlab/cloud-workflows/tree/main/manual-workflows#ssh-in-to-vm).

```
gcloud compute ssh $INSTANCE_NAME
journalctl -u google-startup-scripts -f
journalctl -u cromwell -f
exit

```

### Localize your inputs file

First **on your local system**, copy your cloudized YAML file to a google bucket

```
cd $WORKING_BASE/yamls/
gsutil cp $CLOUD_YAML $GCS_BUCKET_PATH/yamls/$CLOUD_YAML

```

Now log into Google instance again and copy the YAML file to its local file system
```
gcloud compute ssh $INSTANCE_NAME

export GCS_BUCKET_PATH=gs://griffith-lab-test-immuno-pipeline
export CLOUD_YAML=hcc1395_immuno_cloud.yaml

gsutil cp $GCS_BUCKET_PATH/yamls/$CLOUD_YAML .

``` 

### Run the immuno workflow using everything setup thus far

While logged into the google instance:
```
source /shared/helpers.sh
submit_workflow /shared/analysis-wdls/definitions/immuno.wdl $CLOUD_YAML

```




### Once the workflow is done and results retrieved, destroy the Cromwell VM on GCP to avoid wasting resources

```
gcloud compute instances delete $INSTANCE_NAME


```




