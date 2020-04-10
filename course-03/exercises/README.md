# Refactor Udagram App into Microservices and Deploy
[![Build Status](https://travis-ci.com/msdtim/cloud-developer.svg?branch=master)](https://travis-ci.com/msdtim/cloud-developer)

My docker repo: [Docker Hub](https://hub.docker.com/u/msdtim)
## Get Started
For setting up the local develop environment and run the code locally, please refer to [Get Started.md](https://github.com/msdtim/cloud-developer/blob/master/course-03/exercises/Get%20Started.md)
## Environment Variables
Make sure all the following environment variables are properly set.
```
export POSTGRESS_USERNAME=udagrammodev
export POSTGRESS_PASSWORD=XXXXXXXX
export POSTGRESS_DB=udagrammodev
export POSTGRESS_HOST=udagrammodev.cwgjvyl1afii.us-east-1.rds.amazonaws.com
export AWS_REGION=us-east-1
export AWS_PROFILE=default
export AWS_MEDIA_BUCKET=udagram-msdtim-dev2
export AWS_BUCKET=udagram-msdtim-dev2
export JWT_SECRET=helloworld
export URL=http://localhost:8100
```
## Divide the application into smaller services
The starter code has already done the splitting. The applicate is divided to three microserivces and one proxy.
1. The Simple Frontend, A basic Ionic client web application which consumes the RestAPI Backend. `./udacity-c3-frontend/`
2. The RestAPI Feed Backend, a Node-Express feed microservice. `./udacity-c3-restapi-feed/`
3. The RestAPI User Backend, a Node-Express user microservice. `./udacity-c3-restapi-user/`
4. Reverse proxy, a Nginx proxy to guild traffic to the two backend services. `./udacity-c3-deployment/docker/`

## Containerize Application
### Create and edit Dockerfile for each service.
### Create docker images
For each microservice, run the following commands within its directory.

Frontend:
```
## cd to frontend dir
docker build -t msdtim/udacity-frontend .
## cd ../udacity-restapi-feed
docker build -t msdtim/udacity-restapi-feed .
## cd ../udacity-restapi-user
docker build -t msdtim/udacity-restapi-user .
## cd ../udacity-c3-deployment/docker
docker build -t msdtim/reverseproxy .
```
We can run the newly created images by,
```
docker run --rm --publish 8080:8080 -v $HOME/.aws:/root/.aws --env POSTGRESS_HOST=$POSTGRESS_HOST --env POSTGRESS_USERNAME=$POSTGRESS_USERNAME --env POSTGRESS_PASSWORD=$POSTGRESS_PASSWORD --env POSTGRESS_DB=$POSTGRESS_DB --env AWS_REGION=$AWS_REGION --env AWS_PROFILE=$AWS_PROFILE --env AWS_BUCKET=$AWS_BUCKET --env JWT_SECRET=$JWT_SECRET --env URL=$URL --name feed msdtim/udacity-restapi-feed
```
We can interact with the backend with Postman.

To interact using a front end, open another shall window and run:
```
docker run --rm --publish 8100:80 --name frontend msdtim/udacity-frontend
```
**Troubleshoot:**
> Http failure response for http://localhost:8080/api/v0/feed: 0 Unknown Error  

Make sure the `--env URL=$URL`  is in backend run command.
And the `--publish 8100:80` is set correctly for the frontend

### Docker compose and publish
Go to `./udacity-c3-deployment/docker/`

The services are defined in `docker-compose.yaml`

nd the build information is defined in `docker-compose-build.yaml`

Build and publish the docker images by
```
docker-compose -f docker-compose-build.yaml build --parallel
docker-compose -f docker-compose-build.yaml push
```
Run all the image together without the need to open multiple	shell.
```
docker-compose up
```
## Deploy to Kubernetes cluster
### Provision infrastructure using Terraform and install Kubernetes  on AWS using KubeOne 
https://github.com/kubermatic/kubeone/blob/master/docs/quickstart-aws.md

View related files are in: `./udacity-c3-deployment/aws/`

**Troubleshoot:** When `terraform apply`
> Error: Error launching source instance: UnauthorizedOperation: You are not authorized to perform this operation.  

1. If you are using AWS Educate/Student account, there are certain restrictions on what kinds of EC2 instance you can provision, the default `t3.medium` is not available. I tried different kinds of machines but was not able to get it stood up. So I go back to the regular Free Tier AWS account. **Note: `t3.medium` dose not count towards in the Free Tier 750hr quota. It costs real $$. Don’t forget to turn it down.**
2. Temporarily add AdministratorAccess to your IAM user.
### Deploy service to Kubernetes
Update all the files in `./udacity-c3-deployment/k8s/`

Run the following commands in the exact order,
```
kubectl apply -f env-configmap.yaml
kubectl apply -f env-secret.yaml
kubectl apply -f aws-secret.yaml
kubectl apply -f backend-feed-deployment.yaml
kubectl apply -f backend-feed-service.yaml
kubectl apply -f backend-user-deployment.yaml
kubectl apply -f backend-user-service.yaml
kubectl apply -f frontend-deployment.yaml
kubectl apply -f frontend-service.yaml
kubectl apply -f reverseproxy-deployment.yaml
kubectl apply -f reverseproxy-service.yaml
```
You can run `kubectl get pods` in between to check if the service is successfully spin up.
**Troubleshoot:**  
> Pod status: CrashLoopBackOff  

Use `kubectl logs <your_pod_name>` to check what is causing the crash.

For me, I forgot the `-n` when converting the credential to base64.

The correct usage is 
`echo -n <your_credential> | base64`
### Port-forwarding to connect to the service.
In two shall windows (your pod postfix will be different):
```
kubectl port-forward reverseproxy-5976c55bf5-w8wcp 8080:8080
```
```
kubectl port-forward frontend-64d4978694-nh5fm 8100:80 
```
## Travis CI/CD
Add `.travis.yml` in the repo root directory.

The build status page: https://travis-ci.com/msdtim/cloud-developer

Build badge:[![Build Status](https://travis-ci.com/msdtim/cloud-developer.svg?branch=master)](https://travis-ci.com/msdtim/cloud-developer)

