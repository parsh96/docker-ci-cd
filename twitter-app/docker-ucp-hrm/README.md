# Using UCP HRM for rolling upgrade
## Build and push

Build the image using below command

docker image build -t $dtr-url/$dtr-user/tweet-to-us:docker-ucp-hrm-b1 .
docker push $dtr-url/$dtr-user/tweet-to-us:docker-ucp-hrm-b1
e.g.

(*)Login if needed

docker login dtr.example.com:12443/myrepo --username user --password MyComplexPa$$w0rd

docker image build --build-arg 'constraint:ostype==linux' -t dtr.example.com:12443/myrepo/tweet-to-us-hrm:latest .
     
docker push dtr.example.com:12443/myrepo/tweet-to-us-hrm:latest

build-arg will help to run this only on a Linux machine if your UCP cluster has Windows and Linux worker nodes

## Pull and run a service with HRM 

docker service create --network ucp-hrm --name tweet-to-us --mode replicated --replicas 2 --label com.docker.ucp.mesh.http.80="external_route=http://tweet-app.example.com/,internal_port=80" --constraint 'node.platform.os==linux' dtr.example.com:12443/myrepo/tweet-to-us-hrm:latest

You can also do this from the UCP

Make sure that you go to Environment Tab and enter the Label
Key = com.docker.ucp.mesh.http.80
Value = external_route=http://tweet-app.example.com/,internal_port=80

## Make a change and roll to production - Rolling Update

Make changes and rebuild
```
vi index.html
```
Once the changes are done then build again and push to DTR repo
```
docker image build --build-arg 'constraint:ostype==linux' -t dtr.example.com:12443/myrepo/tweet-to-us-hrm:$gitCheckinHash .
docker push dtr.example.com:12443/myrepo/tweet-to-us-hrm:$gitCheckinHash
```
## Update the service
```
docker service update --image dtr.example.com:12443/myrepo/tweet-to-us-hrm:$gitCheckinHash2 tweet-to-us
```

This time you would see a different message.


## Rollback

```
docker service update --rollback   tweet-to-us
```

# Continuous Integration
Refer to the pdf - [GitHub-Jenkins-Integration.pdf](GitHub-Jenkins-Integration.pdf) which gives details of how you can create a Webhook in GitHub repo which can send a notification to your CI Server (Jenkins in this case) to trigger a build on every new change in the repo.


# Continuous Deployment
Refer to the diagram [continuous-deployment-jenkins-and-docker-trusted-reg.png](continuous-deployment-jenkins-and-docker-trusted-reg.png) to get an idea of how you can setup Jenkins to do an automated build and deploy the newly built image.

# Jenkins Build Configuration
Use ssh method to do the build. Use below commands

### First we need to let our docker client talk to UCP (our orchestrator)
** Download the Client Bundle from your UCP

```
cd /home/centos/ucp-bundle
eval $(<env.sh)
cd $WORKSPACE
```

** Now perform a build and push to the dtr
```
docker build --pull=true -t dtr.example.com:12443/myrepo/twitter-app-ci-cd:$GIT_COMMIT ./twitter-app/docker-ucp-hrm/code

docker login --username user --password myComplexPa$$w0rd dtr.example.com:12443/myrepo

docker push dtr.example.com:12443/myrepo/twitter-app-ci-cd:$GIT_COMMIT
```
** Upate the service
```
docker service update --force --update-parallelism 1 --update-delay 30s --image dtr.example.com:12443/myrepo/twitter-app-ci-cd:$GIT_COMMIT tweet-to-us 
```
