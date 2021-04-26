# Devops Chanllenge by Crabi.
# K8S Section.

You can start by cloning the repo:

`cd ${HOME}`
`git clone https://github.com/widolite/devops-challenge`

Very important, "Workload Identity" must be enabled in your cluster before you start, you can do this by creating the cluster with this option set or you can edit,but if you edit your cluster it going to update all nodes, so be ware. 


# Installing nginx Ingress into the cluster. 

Initialize your user as a cluster-admin with the following command:

`kubectl create clusterrolebinding cluster-admin-binding \
  --clusterrole cluster-admin \
  --user $(gcloud config get-value account)`

# Get it the yaml manifest to deploy nginx ingress controller

`kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v0.45.0/deploy/static/provider/cloud/deploy.yaml`


# Set up environtment variables 
Run the following command to configure the environment variables

```
#Change to suit your values.
export PATH_TO_PROJECT=${HOME}/devops-challenge/

export PROJECT_NAME=`gcloud config get-value project`

#Change to suit your values.
export CUSTOM_DOMAIN=uatx.xyz

#Change to suit your values.
export CLUSTER_NAME=devops-challenge

#Change to suit your values.
export CLUSTER_ZONE=us-central1-c

#Cloud DNS Service Account Name.
export CLOUD_DNS_SA=cloud-dns-admin

#K8s Admin Service Account Name.
export CLOUD_K8S_SA=k8s-admin


gcloud --project $PROJECT_NAME iam service-accounts \
    create $CLOUD_DNS_SA \
    --display-name "Service Account to support ACME DNS-01 challenge."

# Fully-qualified service account name also has project-id information.
export CLOUD_DNS_SA=${CLOUD_DNS_SA}@${PROJECT_NAME}.iam.gserviceaccount.com
```

Bind the role dns.admin to the newly created service account.

`gcloud projects add-iam-policy-binding $PROJECT_NAME \
    --member serviceAccount:$CLOUD_DNS_SA \
    --role roles/dns.admin`

Link the ExternalDNS GSA (Google service account) to the Kubernetes service account (KSA) that external-dns will run under, example, the external-dns KSA in the external-dns namespaces.
```
gcloud iam service-accounts add-iam-policy-binding ${CLOUD_DNS_SA} \
    --member="serviceAccount:<workload_identity_namespace>[external-dns/external-dns]" \
    --role=roles/iam.workloadIdentityUser

gcloud iam service-accounts add-iam-policy-binding ${CLOUD_DNS_SA} \
    --member="serviceAccount:silver-retina-233802.svc.id.goog[external-dns/external-dns]" \
    --role=roles/iam.workloadIdentityUser
```

We are going to create our zone in DNS Cloud. 
Put you own dns zone name:
```
export DNS_ZONE_NAME=uatx

gcloud dns managed-zones create $DNS_ZONE_NAME \
    --dns-name $CUSTOM_DOMAIN \
    --description "Automatically managed zone by kubernetes.io/external-dns"
```
We are gonna make note of the nameservers that  were assigned to your new zone:

```
gcloud dns record-sets list \
    --zone $DNS_ZONE_NAME \
    --name $CUSTOM_DOMAIN \
    --type NS
```
In my case, the DNS nameservers are ns-cloud-{c1-c4}.googledomains.com. Yours could differ slightly, e.g. {a1-a4}, {b1-b4} just be careful.

Adding a new record to test our zone in Cloud DNS:

```
gcloud dns record-sets transaction start --zone ${DNS_ZONE_NAME}

gcloud dns record-sets transaction add "1.2.3.4" \
            --name "external-dns-test.${CUSTOM_DOMAIN}" --ttl=300 \
            --type A --zone ${DNS_ZONE_NAME}
```

You can take a peak to the file transaction.yaml it is optional.
  
`gcloud dns record-sets transaction execute --zone ${DNS_ZONE_NAME}`

Apply the following manifest file to deploy external-dns 

`kubectl apply -f ${PATH_TO_PROJECT}/cloudns-external-dns.yaml`

Then add the proper workload identity annotation to the cert-manager service account.

```
kubectl annotate serviceaccount --namespace=external-dns external-dns \
    "iam.gke.io/gcp-service-account=${CLOUD_DNS_SA}"
```
Please use this pod to test the service account 
```
kubectl run -it \
  --image google/cloud-sdk:slim \
  --serviceaccount external-dns \
  --namespace external-dns \
  workload-identity-test
```

You are now connected to an interactive shell within the created Pod. Run the following command inside the Pod. If the service accounts are correctly configured, the Google service account email address is listed as the active (and only) identity in our case "CLOUD_DNS_SA" . This demonstrates that by default, the Pod uses the Google service account's authority when calling Google Cloud APIs.

`gcloud auth list`

Also you can look the logs of the pods running in the external-dns ns 

`kubectl logs pods external-dns-<this-is-random-numbers-letter> -f -n external-dns`


# Installing Cert-Manager.

Install the CustomResourceDefinitions and cert-manager itself:


`kubectl apply -f https://github.com/jetstack/cert-manager/releases/download/v1.3.1/cert-manager.yaml`

Download the secret key file for your service account.

```
gcloud iam service-accounts keys create ${HOME}/key.json \
    --iam-account=$CLOUD_DNS_SA
```
Upload the service account credential to your cluster. This command uses the secret name cloud-dns-key, but you can choose a different name.

```
kubectl create secret generic cloud-dns-key \
    --from-file=key.json=$HOME/key.json -n cert-manager
```
Delete the local secret

`rm ~/key.json`

We need to create the namespace where we gonna deploy our simple go-app.

```
kubectl create ns http-echo-ns

kubectl apply -f ${PATH_TO_PROJECT}/http-echo-deployment.yaml

kubectl apply -f ${PATH_TO_PROJECT}/http-echo-service.yaml

kubectl apply -f ${PATH_TO_PROJECT}/https-echo-ingress.yaml
```

# Git HUB Action CI/CD Section
The project is run in flask + Python, I worked with python here because I had trouble working with the go app, I didn't want to waste time debugging, so I sticked with Python, and for the deployment in K8s cluster, I used Helm, it worked like a charm. 

# Enabling the APIs

Enable the Kubernetes Engine and Container Registry APIs. For example:

```
gcloud services enable \
	containerregistry.googleapis.com \
	container.googleapis.com
```

Create a new service account:

```
gcloud iam service-accounts create $CLOUD_K8S_SA

export CLOUD_K8S_SA=${CLOUD_K8S_SA}@${PROJECT_NAME}.iam.gserviceaccount.com
```
Add roles to the service account. if when you run your github action it ask for permission, recreate this role in the portal.

```
gcloud projects add-iam-policy-binding $PROJECT_NAME \
  --member=serviceAccount:$CLOUD_K8S_SA \
  --role=roles/container.admin \
  --role=roles/storage.admin \
  --role=roles/compute.storageAdmin

gcloud iam service-accounts keys create key.json --iam-account=$CLOUD_K8S_SA
```
We need to convert this key.json in base64 and adding into the Environments Secrets in Git Hub in Environments Production's.  

cat key.json | base64

# You need to do this in GitHUB.
I just add this two secrets in settings > Environments > production. if the environments dint's exists, create a new one and add the vars: GKE_PROJECT, GKE_SA_KEY = < here put your key in base64 format into the git hub >.

And finally just push your changes into the code

# The final result of the project is here auto DNS record and SSL Cert + GitHuB Actions CI/CD.

https://flash-app.uatx.xyz

# Explanation of the deployment strategy.
My deployment strategy in this environment would be base in PR (Pull Request), where the developers have to create a new branch and make a Pull Request and the PR have to be view for any of the developers in the teams and made their feedback to the author (if is any) of the PR and then be approved, then be merge into the main branch to trigger the git hub action that will put the app into production. This can be improve a lot.
