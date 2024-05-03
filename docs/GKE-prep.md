### GKE

Specific requirements for deploying to Google Kubernetes Engine (GKE)


1. A working connection to your GKE cluster setup with Artifact Registry

```shell
# Log into gcloud
gcloud auth login

# configure docker (make sure to specify appropriate region)
gcloud auth configure-docker europe-west1-docker.pkg.dev 
```

2. Build and push the image to Google Artifact Registry

```shell
export TAG=latest \
  REPOSITORY="leeram-docker" PROJECT=leeram
export GCR=europe-west1-docker.pkg.dev/$PROJECT/$REPOSITORY/moodle:$TAG
```

```shell
docker build . -t ceduth/moodle:$TAG \
  && docker tag ceduth/moodle:$TAG $GCR \
  && docker push $GCR
```

3. Resume with [next step](../README.md#kubernetes) Kubernetes deployment guide.