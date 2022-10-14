## Build the container

```
git clone --recurse-submodules https://github.com/mathieu-benoit/myblog
docker build -t blog .
```

## Run locally

```
docker run -d -p 8080:8080 blog
```

## Deploy on Kubernetes

```
imageNameInRegistry=myblog
kubectl create ns myblog
kubectl config set-context --current --namespace myblog
kubectl create deployment myblog --image=$imageNameInRegistry --port=8080
kubectl expose deployment myblog --port=8080 --target-port=8080
```

## Configure GitHub action

```
projectId=FIXME
gcloud config set project $projectId

# Create the Service Account
saName=container-images-builder
gcloud iam service-accounts create $saName
saId="${saName}@${projectId}.iam.gserviceaccount.com"

# Enable the IAM Credentials API
gcloud services enable iamcredentials.googleapis.com

# Create a Workload Identity Pool
poolName=container-images-builder-wi-pool
gcloud iam workload-identity-pools create $poolName \
  --location global \
  --display-name $poolName
poolId=$(gcloud iam workload-identity-pools describe $poolName \
  --location global \
  --format='get(name)')

# Create a Workload Identity Provider with GitHub actions in that pool:
attributeMappingScope=repository
gcloud iam workload-identity-pools providers create-oidc $poolName \
  --location global \
  --workload-identity-pool $poolName \
  --display-name $poolName \
  --attribute-mapping "google.subject=assertion.${attributeMappingScope},attribute.actor=assertion.actor,attribute.aud=assertion.aud,attribute.repository=assertion.repository" \
  --issuer-uri "https://token.actions.githubusercontent.com"
providerId=$(gcloud iam workload-identity-pools providers describe $poolName \
  --location global \
  --workload-identity-pool $poolName \
  --format='get(name)')

# Allow authentications from the Workload Identity Provider to impersonate the Service Account created above
gitHubRepoName="mathieu-benoit/myblog"
gcloud iam service-accounts add-iam-policy-binding $saId \
  --role "roles/iam.workloadIdentityUser" \
  --member "principalSet://iam.googleapis.com/${poolId}/attribute.${attributeMappingScope}/${gitHubRepoName}"

# Allow the GSA to write container images in Artifact Registry
artifactRegistryName=containers # FIXME
artifactRegistryLocation=us-east4 # FIXME
gcloud artifacts repositories add-iam-policy-binding $artifactRegistryName \
    --location $artifactRegistryLocation \
    --member "serviceAccount:$saId" \
    --role roles/artifactregistry.writer

# Allow the GSA to scan container images on-demand
gcloud services enable ondemandscanning.googleapis.com
gcloud projects add-iam-policy-binding $projectId \
    --member=serviceAccount:$saId \
    --role=roles/ondemandscanning.admin

# Setup GitHub actions variables
gh auth login --web
gh secret set CONTAINER_REGISTRY_PROJECT_ID -b"${projectId}"
gh secret set CONTAINER_REGISTRY_NAME -b"${artifactRegistryName}"
gh secret set CONTAINER_REGISTRY_HOST_NAME -b"${artifactRegistryLocation}-docker.pkg.dev"
gh secret set CONTAINER_IMAGE_BUILDER_SERVICE_ACCOUNT_ID -b"${saId}"
gh secret set WORKLOAD_IDENTITY_POOL_PROVIDER -b"${providerId}"
```

## Setup Cloud Monitoring

```
gcloud monitoring dashboards create --config-from-file=gcloud/monitoring/dashboard.yaml
```
