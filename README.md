# gcp-tableau-server-container
Logic to build Tableau Server container image on GCP using Cloud Shell

## Step 1 - Set the neccessary variables in Cloud Shell
Run the following in Cloud Shell to configure variables needed throughout these steps
```
INSTANCE=centos-vm
REGION=europe-west2
ZONE=europe-west2-c
export PROJECTID=$(gcloud info --format='value(config.project)')
REGISTRYNAME=tableau-server-images
SANAME=artifactregistry
SAEMAIL=$SANAME@$PROJECTID.iam.gserviceaccount.com
```

## Step 2 - Create VM with centos os
Create a basic Compute Enginer VM, using centos image that will be used to build the container image
```
gcloud compute instances create $INSTANCE \
    --project=$PROJECTID \
    --zone=$ZONE \
    --machine-type=e2-medium \
    --tags=http-server,https-server \
    --create-disk=auto-delete=yes,boot=yes,device-name=instance-1,image=projects/centos-cloud/global/images/centos-7-v20231115,mode=rw,size=20,type=projects/$PROJECTID/zones/$ZONE/diskTypes/pd-balanced
```

## Step 3 - Create Artifact Registry
Create an Artifact Registry for the docker images to be pushed to
```
gcloud artifacts repositories create $REGISTRYNAME --repository-format=docker \
--location=$REGION --description="Docker repository for Tableau images"
```

## Step 4 - Create service account for Artifact Registry and copy private key to VM
Create a service account with permissions to write to the Artifact Registry
```
gcloud iam service-accounts create $SANAME \
    --description="Artifact registry writer" \
    --display-name="Artifact Registry"
```
Bind the correct permission to the service account
```
gcloud projects add-iam-policy-binding $PROJECTID \
    --member="serviceAccount:$SAEMAIL" \
    --role=roles/artifactregistry.writer
```
Generate keys for the service account
```
gcloud iam service-accounts keys create private-key.json \
    --iam-account $SAEMAIL
```
Copy the key to the VM for use in Step 7. Make sure to follow the prompts to generate SSH keys with the VM
```
gcloud compute scp private-key.json $INSTANCE:. --zone=$ZONE
```

## Step 5 - SSH into VM and install docker
SSH into the VM

```
gcloud compute ssh --zone $ZONE $INSTANCE --project $PROJECTID

```
Install docker and add yourself into the docker group
```
sudo yum install -y yum-utils
sudo yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
sudo yum install -y docker-ce
sudo usermod -aG docker $USER
```
Reboot the VM, failing to do this leads to errors in the build step due to it incorrectly detecting docker version
```
sudo reboot
```

## Step 6 - SSH into VM, download neccessary files and build image
SSH back into the VM after the reboot has been completed
```
gcloud compute ssh --zone $ZONE $INSTANCE --project $PROJECTID
```
Start docker on the VM
```
sudo systemctl start docker
```
Download the relevant files needed to be Tableau Server image
```
curl -O https://downloads.tableau.com/esdalt/2023.1.7/tableau-server-2023-1-7.x86_64.rpm
curl -O https://downloads.tableau.com/esdalt/2023.1.3/tableau-server-container-setup-tool-2023.1.3.tar.gz
```
Unpack the container setup tool
```
tar -xzf tableau-server-container-setup-tool-2023.1.3.tar.gz
```
Replace the initialize-tsm with the -f flag (this allows us to build the image even thought our VM doesn't meet the min Tableau Server requirements)
```
sed -i "s/initialize-tsm/initialize-tsm -f/g" tableau-server-container-setup-tool-2023.1.3/image/docker/install-process-manager
```
Build the container image
```
tableau-server-container-setup-tool-2023.1.3/build-image \
--accepteula \
-i tableau-server-2023-1-7.x86_64.rpm
```

## Step 7 - Tag and push image to Artifact repository
Run the following in Cloud Shell to configure variables needed throughout these steps
```
INSTANCE=centos-vm
REGION=europe-west2
export PROJECTID=$(gcloud info --format='value(config.project)')
REGISTRYNAME=tableau-server-images
SANAME=artifactregistry
SAEMAIL=$SANAME@$PROJECTID.iam.gserviceaccount.com
```
Run the following command to find the image ID of the container
```
docker image list
```
Create a variable using the image ID found above and use this to tag the image
```
IMAGEID=<insert image ID from above here>
docker tag 48ff05d557b7 $REGION-docker.pkg.dev/$PROJECTID/$REGISTRYNAME/tab-server-23.1.7
```
Authenticate as the Service account we created, failing to do this will prevent us from pushing the image
```
gcloud auth activate-service-account $SAEMAIL --key-file=private-key.json
```
Configure gcloud as the credential helper for the Artifact Registry domain associated with this repository's location
```
gcloud auth configure-docker \
    $REGION-docker.pkg.dev
```
Push the image to the Artifact Registry we created
```
docker push $REGION-docker.pkg.dev/$PROJECTID/$REGISTRYNAME/tab-server-23.1.7
```
Exit the VM
```
exit
```

## Step 8 - Destroy centos VM
Cleanup the build VM by destroying it
```
gcloud compute instances delete $INSTANCE --zone=$ZONE
```