# Deploy Kubernetes Applications on Google Cloud: Challenge Lab 

<img width="1903" height="881" alt="Screenshot From 2026-10-23 17-28-37" src="https://github.com/user-attachments/assets/c1dc285d-d3e9-4454-bdc5-a6af7cedbec0" />


### Set the Variables 

```
export REPO=valkyrie-repository
export DOCKER_IMAGE=valkyrie-app
export TAG=v0.0.2
export ZONE=us-east1-c
export REGION=us-east1

```

### Task 1. Create a Docker image and store the Dockerfile ###
- Open Cloud Shell and run the following command. This will install the script you will use to help check your progress.

```
source <(gcloud storage cat gs://spls/gsp318/script.sh)
```
```
gcloud auth list
```

```
gcloud storage cp gs://spls/gsp318/valkyrie-app.tgz .
tar -xzf valkyrie-app.tgz
cd valkyrie-app
```

#### Create valkyrie-app/Dockerfile and add the configuration below ####
```
cat > Dockerfile <<EOF
FROM golang:1.10
WORKDIR /go/src/app
COPY source .
RUN go install -v
ENTRYPOINT ["app","-single=true","-port=8080"]
EOF
```
#### Build the docker image ####
```
docker build -t $DOCKER_IMAGE:$TAG .
```
#### Verify that the Docker image was created
```
docker images
```
#### Launch a container using the image ####
```
docker run -p 8080:8080 $DOCKER_IMAGE:$TAG &
```
#### Create a repository named Repository Name in Artifact Registry ####
```
gcloud artifacts repositories create $REPO \
    --repository-format=docker \
    --location=$REGION \
    --description="awesome lab" \
    --async 
```
#### Configure Docker to use the Google Cloud CLI to authenticate requests to Artifact Registry ####
```
gcloud auth configure-docker $REGION-docker.pkg.dev --quiet
```
```
Image_ID=$(docker images --format='{{.ID}}')
```
#### Re-tag the container to be able push it to the repository. ####
```
docker tag $Image_ID $REGION-docker.pkg.dev/$DEVSHELL_PROJECT_ID/$REPO/$DOCKER_IMAGE:$TAG
```
```
docker push $REGION-docker.pkg.dev/$DEVSHELL_PROJECT_ID/$REPO/$DOCKER_IMAGE:$TAG
```
### Task 4. Create and expose a deployment in Kubernetes ###
```
sed -i s#IMAGE_HERE#$REGION-docker.pkg.dev/$DEVSHELL_PROJECT_ID/$REPO/$DOCKER_IMAGE:$TAG#g k8s/deployment.yaml
```
```
gcloud container clusters get-credentials valkyrie-dev --zone $ZONE
```
```
kubectl create -f k8s/deployment.yaml
```
```
kubectl create -f k8s/service.yaml
```
