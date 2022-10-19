# Running the WASHU Immunogenomics Workflow on Google Cloud - compute1 version

## Step-by-step instructions
The following assumes that you have already authenticated to GCP and initialized the project. The following is the minimum needed to stage data, run the workflow, pull down the results and clean up.


### Set some Google Cloud and other environment variables
The following environment variables are used merely for convenience and should be customized to produce intuitive labeling for your own analysis:
```bash

#SET COMPARISON SPECIFIC ENVIRONMENT VARIABLES
#e.g. /storage1/fs1/carreno-upenn/Active/adhoc/melanoma_cell_line_vaccines/pipeline_runs/MEL21/MEL21_Tumor_Skin1/MEL21_Tumor_Skin1_immuno_local-WDL.envs
#export GROUP=compute-oncology
#export GCS_PROJECT=griffith-lab
#export GCS_SERVICE_ACCOUNT=cromwell-server@$GCS_PROJECT.iam.gserviceaccount.com
#export GCS_BUCKET_NAME=griffith-lab-carreno
#export GCS_BUCKET_PATH=gs://$GCS_BUCKET_NAME
#export GCS_INSTANCE_NAME=malachi-immuno-carreno
#export BASE=/storage1/fs1/carreno-upenn/Active/adhoc/melanoma_cell_line_vaccines/pipeline_runs
#export PATIENT=MEL21
#export TUMOR=MEL21_Tumor_Skin1
#export WORKING_BASE=$BASE/$PATIENT/$TUMOR
#export WORKFLOW=immuno.wdl
#export WORKFLOW_PATH=$BASE/git/analysis-wdls/definitions/$WORKFLOW
#export LOCAL_YAML="$TUMOR"_immuno_local-WDL.yaml
#export CLOUD_YAML="$TUMOR"_immuno_cloud-WDL.yaml

```

## Local setup

### First create a working directory on your local system
```bash
mkdir $WORKING_BASE
cd $WORKING_BASE
```

### Gather input data and reference files to your local system
Create a directory for YAML files and create one for the desired pipeline that points to the location of input files on your local system

```bash
cd $WORKING_BASE
mkdir yamls
cd yamls
```

Setup yaml files for an example run.
```bash
cp $WORKING_BASE/$LOCAL_YAML $WORKING_BASE/yamls/
```

### Stage input files to cloud bucket

Start an interactive docker session capable of running the "cloudize" scripts
```bash
bsub -Is -q oncology-interactive -G $GROUP -a "docker(mgibio/cloudize-workflow:latest)" /bin/bash
```

Attempt to cloudize your workflow and inputs
```bash
python3 /opt/scripts/cloudize-workflow.py $GCS_BUCKET_NAME $WORKFLOW_PATH $WORKING_BASE/yamls/$LOCAL_YAML --output=$WORKING_BASE/yamls/$CLOUD_YAML
```

### Start a Google VM that will run Cromwell and orchestrate completion of the workflow

```bash
cd $BASE/git/cloud-workflows/manual-workflows/
bash start.sh $GCS_INSTANCE_NAME --server-account $GCS_SERVICE_ACCOUNT --project $GCS_PROJECT --boot-disk-size=250GB --boot-disk-type=pd-standard --machine-type=e2-standard-2
exit #leave the docker session
```

### Log into the VM and check status 

In this step we will confirm that we can log into our Cromwell VM with `gcloud compute ssh` and make sure it is ready for use.

After logging in, use journalctl to see if the instance start up has completed, and cromwell launch has completed.

For details on how to recognize whether these processes have completed refer: [here](https://github.com/griffithlab/cloud-workflows/tree/main/manual-workflows#ssh-in-to-vm).

```bash
gcloud compute ssh $GCS_INSTANCE_NAME
journalctl -u google-startup-scripts -f
journalctl -u cromwell -f
exit #leave the google VM
```

### Localize your inputs file

First **on your local system**, copy your cloudized YAML file to a google bucket

```bash
cd $WORKING_BASE/yamls/
gsutil cp $CLOUD_YAML $GCS_BUCKET_PATH/yamls/$PATIENT/$TUMOR/$CLOUD_YAML
```

Now log into Google Cromwell VM instance again and copy the YAML file to its local file system
```bash
gcloud compute ssh $GCS_INSTANCE_NAME

#ADD ENVS TO BASHRC. e.g.
/storage1/fs1/carreno-upenn/Active/adhoc/melanoma_cell_line_vaccines/pipeline_runs/MEL21/MEL21_Tumor_Skin1/MEL21_Tumor_Skin1_immuno_cloud-WDL.envs
#export GCS_BUCKET_NAME=griffith-lab-carreno
#export GCS_BUCKET_PATH=gs://$GCS_BUCKET_NAME
#export PATIENT=MEL21
#export TUMOR=MEL21_Tumor_Skin1
#export CLOUD_YAML="$TUMOR"_immuno_cloud-WDL.yaml
#export WORKFLOW=immuno.wdl

gsutil cp $GCS_BUCKET_PATH/yamls/$PATIENT/$TUMOR/$CLOUD_YAML .

``` 

### Merge in the class ii PR
```bash
cd /shared/analysis-wdls/
sudo git fetch origin pull/57/head:classii
sudo git merge classii
refresh_zip_deps
cd ~

```

### Run the immuno workflow using everything setup thus far

While logged into the Google Cromwell VM instance:
```bash
source /shared/helpers.sh
submit_workflow /shared/analysis-wdls/definitions/$WORKFLOW $CLOUD_YAML

```

### Monitor progress of the workflow run:

While the job is running you can see Cromwell logs live as they occur by doing this
```bash
journalctl -f -u cromwell

```

You can also query the cromwell server using your $WORKFLOW_ID. This ID will have been printed out when you launched the worflow. It will also be the top level UUID in the output files being generated in your Google Bucket.

Example commands that query the Cromwell server and get the current status, current timing diagram, and then save that to the Google Bucket for easy access:
```bash
export WORKFLOW_ID=<id from above>
curl http://localhost:8000/api/workflows/v1/$WORKFLOW_ID/status
curl http://localhost:8000/api/workflows/v1/$WORKFLOW_ID/timing > $WORKFLOW_ID.timing.html
gsutil cp $WORKFLOW_ID.timing.html $GCS_BUCKET_PATH/workflow_artifacts/$WORKFLOW_ID/$WORKFLOW_ID.timing.html

```

Note that if you have been testing and have several workflows that have been run on the same server it can get confusing to tell which workflow is which and in what order they were started.  The following command will parse the cromwell log for records of workflows that were started, in order and at what time:
```bash
journalctl -u cromwell | grep "Starting workflow UUID" | perl -ne 'if ($_ =~ /(\w+\s+\d+\s+\d+\:\d+\:\d+).*UUID\((\S+)\)/){print "$2\tstarted $1\n"}'
```

### Save information about the workflow run itself - Timing Diagram and Outputs List

After a workflow is run, before exiting and deleting your VM, make sure that the timing diagram and the list of outputs are available so you can make use of the data outside of the cloud.

First determine you WORKFLOW_ID. This can be done several ways. If the run was successful it should be reported at the bottom of the cromwell log as "$WORKFLOW_ID  completed with status Succeeded". Or you find it by the name of the directory where your run was stored in the Google bucket. Both of these approaches are illustrated here:

```bash
gsutil ls $GCS_BUCKET_PATH/cromwell-executions/immuno/

journalctl -u cromwell | tail | grep "Workflow actor"
```

Now save the workflow information in your google bucket. Include the YAML and WDLs used for this run for analysis provenance.
```bash
export WORKFLOW_ID=<id from above>
source /shared/helpers.sh
save_artifacts $WORKFLOW_ID $GCS_BUCKET_PATH/workflow_artifacts/$WORKFLOW_ID
gsutil cp $GCS_BUCKET_PATH/yamls/$CLOUD_YAML $GCS_BUCKET_PATH/workflow_artifacts/$WORKFLOW_ID/$CLOUD_YAML
gsutil cp /shared/analysis-wdls/workflows.zip $GCS_BUCKET_PATH/workflow_artifacts/$WORKFLOW_ID/workflows.zip
gsutil cp /shared/analysis-wdls/definitions/$WORKFLOW $GCS_BUCKET_PATH/workflow_artifacts/$WORKFLOW

```

This command will upload the workflows artifacts to your google bucket so they can be used after the VM is deleted. They can be found at paths:

```bash
$GCS_BUCKET_PATH/workflow_artifacts/$WORKFLOW_ID/timing.html
$GCS_BUCKET_PATH/workflow_artifacts/$WORKFLOW_ID/outputs.json
```

Confirm that they were successfully transferred and logout of the Cromwell VM on GCP:
```bash
gsutil ls $GCS_BUCKET_PATH/workflow_artifacts/$WORKFLOW_ID
exit #leave the google VM
```

The file `outputs.json` will simply be a map of output names to their GCS locations. The `pull_outputs.py` script can be used to retrieve the actual files.


### Pulling the Outputs from Google Cloud Bucket back to your local system or cluster
After the work in your compute instance is all done, including `save_artifacts`, and you want to bring your results back to the cluster, leverage the `pull_outputs.py` script with the generated `outputs.json` to retrieve the files.

On compute1 cluster, jump into a docker container with the script available

```bash
export WORKFLOW_ID=<id from above>
bsub -Is -q general-interactive -G $GROUP -a "docker(mgibio/cloudize-workflow:latest)" /bin/bash
```

Execute the script:

```bash
cd $WORKING_BASE
mkdir final_results
cd final_results

python3 /opt/scripts/pull_outputs.py --outputs-file=$GCS_BUCKET_PATH/workflow_artifacts/$WORKFLOW_ID/artifacts/outputs.json --outputs-dir=$WORKING_BASE/final_results/
exit #leave the docker session
```

Examine the outputs briefly:
```bash 
cd $WORKING_BASE/final_results
ls -l
du -h

```

### Store a local copy of workflow artifacts
```bash
export WORKFLOW_ID=<id from above>
bsub -Is -q general-interactive -G $GROUP -a "docker(mgibio/cloudize-workflow:latest)" /bin/bash

cd $WORKING_BASE/final_results/
mkdir workflow_artifacts
cd workflow_artifacts

gsutil cp -r  $GCS_BUCKET_PATH/workflow_artifacts/$WORKFLOW_ID/* .

exit
```

### Once the workflow is done and results retrieved, destroy the Cromwell VM on GCP to avoid wasting resources

```bash
gcloud compute instances delete $GCS_INSTANCE_NAME
```

Note that this does NOT empty your bucket. Once you are satisfied that your results are acceptable and you will no longer need to access the cromwell-executions files in your bucket you can clean it up to prevent billing for cloud bucket storage.

