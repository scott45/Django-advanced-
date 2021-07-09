# Setting up Gluu server 4.x with Openshift (OCP-4.7) on Google Cloud. (Same should work for 4.xx versions)

1. Requirements;

```
Google Cloud billable account

gcloud cli 

openshift cli tools

pull secret

dns domain (bought from google or delegated to google)
```

2. Go to `https://www.openshift.com/try` and Login with your credentials or create a new account. 

4. Select cloud, and scroll down to `Run it yourself` section. 

6. Go to google cloud and select `User provisioned infrastructure`. 

8. Download the following from that page. (will ask for a directory for the installation program. Using `./ocp4` in this tutorial)

```
- Openshift installer for your OS. Currently only Linux and MacOS are supported. Note the directory where you'll store your confoguration files.

- Download and copy the pull secret. You'll be prompted for this during installation. 

- Download the commandline tools. After completion, you'll get the console url and credentials for accessing the cluster. There will also be an oc cli kubeconfig file downloaded for you. 
``` 

8. Download and install the gcloud-cli. Go to https://cloud.google.com/sdk/docs and get the installer for your OS. 

10. After installing gcloud, run `gcloud init` too intilialize, login and authorize gcloud for your account in the browser. 

12. Go to the google console and login.

12. Create a GCP project if you didn't have any. create a project with this command `gcloud projects create ${gcp_project}`

14. Set the region, zone, project etc using `gcloud config set project ${gcp_project}` etc

16. Run `gcloud config list` to verify your configuration values. 

18. Enable the following google api's in the UI for your installation to run.

```
Compute Engine API -->                       (compute.googleapis.com)

Google Cloud APIs -->                        (cloudapis.googleapis.com)

Cloud Resource Manager API -->               (cloudresourcemanager.googleapis.com)

Google DNS API -->                           (dns.googleapis.com)

Identity and Access Management (IAM) API --> (iam.googleapis.com)

IAM Service Account Credentials API -->      (iamcredentials.googleapis.com)

Identity and Access Management (IAM) API --> (iam.googleapis.com)

Service Management API -->                   (servicemanagement.googleapis.com)

Service Usage API -->                        (serviceusage.googleapis.com)

Google Cloud Storage JSON API -->            (storage-api.googleapis.com)

Cloud Storage -->                            (storage-component.googleapis.com)
```

13. Create a service account that grants authentication and authorization to access data in the Google APIs `gcloud iam service-accounts create ${gcp_sa}`

15. Grant the above service account appropiate permissions. This command will assign the owner role to it. This service account has to be handled with care as it has total control over your project. 

`gcloud projects add-iam-policy-binding ${gcp_project} --member=ServiceAccount:${gcp_sa}@${gcp_project}.iam.gserviceaccount.com --role="roles/owner"`

22. Store service account key locally for the installation.
- mkdir -p ~/.gcp

16. Create a service account key in json format, this is needed to create the cluster. Google cloud requires identity establishment of the service account if its to be used on other platforms

`gcloud iam service-accounts keys create ~/.gcp/sa-private-key.json --iam-account=${gcp_sa}@${gcp_project}.iam.gserviceaccount.com`

NOTE: After you download the key, you can't download it again. Store your key securely as it can be used to authenticate as your service account. 

17. (Optional) Set the service key in your path `GOOGLE_APPLICATION_CREDENTIALS=~/.gcp/sa-private-key.json`

19. Set up DNS. You can use an existing root domain and registrar or obtain a new one through GCP or another source. 

- Create a new managed zone that cloud DNS supports. Command for the format is; 
```
gcloud dns managed-zones create NAME_FOR_YOUR_ZONE \
    --description=DESCRIPTION_FOR_YOUR_ZONE \
    --dns-name=DNS_SUFFIX_FOR_YOUR_ZONE \
    --labels=LABELS_OPTIONAL_K-V_PAIR \
    --visibility=public
 ```

e.g `gcloud dns managed-zones create gluu-server-openshift --description="Gluu Openshift 4 Domain" --dns-name=demoexample.gluu.org --visibility=public`

- Get the dns servers for the domain by running the describe command. Register them with your DNS provider.

`gcloud dns managed-zones describe gluu-server-openshift` 

- Follow this link https://domains.google/ for information about purchasing domains through Google.

21. Before creating the cluster, we need to verify that the dns delegation has been propely propagated. 
`dig demoexample.gluu.org @xx.xx.xx.xx`

22. Create a folder to store all openshift installation artifacts
- `mkdir ocp4` - this will be used for storing files that the installation program creates. We specify this with the install command.
- `cd ./ocp4`  - this contains the instllation program from step 8. We run the below command from it. 
- `./openshift-install create install-config --dir=./ocp4`
- The above command usually will prompt for ssh-key (Optional), cloud platform, service account key, project-id, region, base-domain, cluster-name, pull secret.


- `vi ./install-config.yaml`

```
apiVersion: v1
baseDomain: demoexample.gluu.org 
controlPlane:  
  architecture: amd64
  hyperthreading: Enabled 
  name: master
  platform:
    gcp:
      type: e2-highcpu-8
      zones:
      - us-west1-a
      - us-west1-c
      osDisk:
        diskType: pd-standard
        diskSizeGB: 50
  replicas: 3
compute:   
- architecture: amd64
  hyperthreading: Enabled 
  name: worker
  platform:
    gcp:
      type: e2-highcpu-8
      zones:
      - us-west1-a
      - us-west1-c
      osDisk:
        diskType: pd-standard
        diskSizeGB: 50
  replicas: 3
metadata:
  name: gluu-ocp4-test-cluster 
networking:
  clusterNetwork:
  - cidr: 10.128.0.0/14
    hostPrefix: 23
  machineNetwork:
  - cidr: 10.0.0.0/16
  networkType: OpenShiftSDN
  serviceNetwork:
  - 172.30.0.0/16
platform:
  gcp:
    projectID: Scott-Assessment 
    region: us-central1 
pullSecret: '{"auths": ...}' 
fips: false 
publish: External
sshKey: ssh-ed25519 AAAA...
```

- sshkeys are optional. you could add them in the script as shown above to the agent for debugging and installation troubleshooting purposes. (~/.ssh/authorized_keys)

- Backup the `install-config.yaml` for re-use purpose if you must since its consumed during the installation process.

- Run `openshift-install create cluster --dir=./ocp4 --log-level=info` to create the cluster. 

- 25. Get the cluster browser url provided along with the `kubeadmin` username / password and login.

23. To interact with ocp cluster from the host, first set the kubeadmin credentials
```
mkdir -p $HOME/.kube
sudo cp -i ./ocp4/auth/kubeconfig $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

24. Login via cli
- oc login -u <user_name>

24. Verify it works on the commandline 
- oc whoami
- oc get nodes
- oc get sc. (this will return sttorage class)
- oc get clusterversion (openshift cluster version)

25. Create a user and assign them admin role.

- oc create user <user_name>
- `oc get users` to obtain the list of users
- kubectl create clusterrolebinding permissive-binding --clusterrole=cluster-admin --user=<user_name> --group=system:serviceaccounts

 


NB: You must not delete the installation program or the files that the installation program creates. Both are required to delete the cluster.
NB: An older version of oc cannot be used on OpenShift Container Platform 4.7. Download and install the new version of oc.
NB: The cluster access and credential information also outputs to `./ocp4/.openshift_install.log` when an installation succeeds.


















