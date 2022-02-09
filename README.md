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

### Monitor progress of the workflow run:
While the job is running you can see Cromwell logs live as they occur by doing this

```
journalctl -f -u cromwell

```

### Save information about the workflow run itself - Timing Diagram and Outputs List

After a workflow is run, before exiting and deleting your VM, make sure that the timing diagram and the list of outputs are available so you can make use of the data outside of the cloud.

First determine you WORKFLOW_ID. This can be done several ways. If the run was successful it should be reported at the bottom of the cromwell log as "$WORKFLOW_ID  completed with status Succeeded". Or you find it by the name of the directory where your run was stored in the Google bucket. Both of these approaches are illustrated here:

```
export GCS_BUCKET_PATH=gs://griffith-lab-test-immuno-pipeline
gsutil ls $GCS_BUCKET_PATH/cromwell-executions/immuno/

journalctl -u cromwell | tail | grep "Workflow actor"

```

Now save the workflow information in your google bucket
```
export WORKFLOW_ID=<id from above>
source /shared/helpers.sh
save_artifacts $WORKFLOW_ID gs://$GCS_BUCKET_PATH/workflow_artifacts/

```

This command will upload the workflow's artifacts to your google bucketS so they can be used after the VM is deleted. They can be found at paths:

```
gs://$GCS_BUCKET_PATH/workflow_artifacts/$WORKFLOW_ID/timing.html
gs://$GCS_BUCKET_PATH/workflow_artifacts/$WORKFLOW_ID/outputs.json

```

The file `outputs.json` will simply be a map of output names to their GCS locations. The `pull_outputs.py` script can be used to retrieve the actual files.


### Pulling the Outputs from Google Cloud Bucket back to your local system or cluster
After the work in your compute instance is all done, including `save_artifacts`, and you want to bring your results back to the cluster, leverage the `pull_outputs.py` script with the generated `outputs.json` to retrieve the files.

On compute1 cluster, jump into a docker container with the script available

```
bsub -Is -q general-interactive -G $GROUP -a "docker(jackmaruska/cloudize-workflow:latest)" /bin/bash

```

Execute the script

```
export WORKFLOW_ID=<id from above>
cd $WORKING_BASE
mkdir final_results
cd final_results

python3 /opt/scripts/pull_outputs.py --outputs-file=$GCS_BUCKET_PATH/workflow_artifacts/$WORKFLOW_ID/outputs.json --outputs-dir=$WORKING_BASE/final_results/

```

### Once the workflow is done and results retrieved, destroy the Cromwell VM on GCP to avoid wasting resources

```
gcloud compute instances delete $INSTANCE_NAME

```

