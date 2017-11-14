# Using UCP HRM for rolling upgrade
This example uses a simple html file and nginx image to demonstrate how you can 
  * Use the HTTP Routing Mesh (HRM) feature of UCP
  * Use HRM for rolling upgrade
  * Use Jenkins and Github along with Docker for continuous deployments

# Swarm Routing Mesh
Before understanding HRM let's take a look at Swarm Routing Mesh which uses an ingress network to expose a container port on all the members of the swarm cluster. A schematic diagram of Swarm Routing Mesh is depicted in the diagram - [swarm-routing-mesh.png](swarm-routing-mesh.png)

![Swarm Routing Mesh](https://github.com/sameerkasi200x/docker-ci-cd/blob/master/swarm-routing-mesh.png?raw=true)

Using Swarm routing mesh can makes it easier for you to configure new services as you can expore the port on all the nodes and traffic will be routed by the ingress overlay network. You don't need to configure a load balancer for each and every service.

But Swarm Routing Mesh suffers one limitation - still each service is listening on a different port and this would pose some challenges in enterprise environment where you will have to roll out new services rapidly. This is where value of HTTP Routing Mesh comes up.

# HTTP Routing Mesh
A schematic explanation of HTTP Routing Mesh is as described in the diagram - [http-routing-mesh.png](http-routing-mesh.png).

![HTTP Routing Mesh](https://github.com/sameerkasi200x/docker-ci-cd/blob/master/http-routing-mesh.png?raw=true)

As you can see in the diagram the you don't need to expose different ports and all traffic is routed via http ports.

## Build and push

Build the image using below command

```
docker image build -t $dtr-url/$dtr-RepoOwner/tweet-to-us:docker-ucp-hrm-b1 .
docker push $dtr-url/$dtr-user/tweet-to-us:docker-ucp-hrm-b1
```
e.g.

```
##Login if needed
docker login dtr.example.com:12443/RepoOwner --username user --password MyComplexPa$$w0rd
```
```
docker image build --build-arg 'constraint:ostype==linux' -t dtr.example.com:12443/RepoOwner/tweet-to-us-hrm:b1 .
docker push dtr.example.com:12443/RepoOwner/tweet-to-us-hrm:b1
```
build-arg will help to run this only on a Linux machine if your UCP cluster has Windows and Linux worker nodes

## Pull and run a service with HRM 

```
docker service create --network ucp-hrm --name tweet-to-us --mode replicated --replicas 2 --label com.docker.ucp.mesh.http.80="external_route=http://tweet.app.example.com/,internal_port=80" --constraint 'node.platform.os==linux' dtr.example.com:12443/RepoOwner/tweet-to-us-hrm:b1
```

You can also do this from the UCP

Make sure that you go to Environment Tab and enter the Label
Key = ```com.docker.ucp.mesh.http.80```
Value = ```external_route=http://tweet.app.example.com/,internal_port=80```

## Make a change and roll to production - Rolling Update

Make changes to ```index.html``` and rebuild.

Once the changes are done then build again and push to DTR repo
```
docker image build --build-arg 'constraint:ostype==linux' -t dtr.example.com:12443/RepoOwner/tweet-to-us-hrm:b2 .
docker push dtr.example.com:12443/RepoOwner/tweet-to-us-hrm:b2
```
## Update the service
```
docker service update --image dtr.example.com:12443/RepoOwner/tweet-to-us-hrm:b2 tweet-to-us
```

This time you would see a different message as per your edit.


## Rollback

```
docker service update --rollback   tweet-to-us
```

# Continuous Integration
Refer to the pdf - [GitHub-Jenkins-Integration.pdf](GitHub-Jenkins-Integration.pdf) which gives details of how you can create a Webhook in GitHub repo which can send a notification to your CI Server (Jenkins in this case) to trigger a build on every new change in the repo.
The diagram [continuous-integration-jenkins-and-docker-trusted-reg.png](continuous-integration-jenkins-and-docker-trusted-reg.png) to get an idea of the flow of events. 

![Jenkins - GitHub - Docker integration](https://github.com/sameerkasi200x/docker-ci-cd/blob/master/continuous-integration-jenkins-and-docker-trusted-reg.png?raw=true)

You can extend this diagram to do a continuous integration of multiple services or introduce unit test cases into the flow after the image build.

# Continuous Deployment
We can go a step further from our idea of continuous builds and do continuous deployments if the code is promoted to the production repository. Refer to the diagram [continuous-deployment-jenkins-and-docker-trusted-reg.png](continuous-deployment-jenkins-and-docker-trusted-reg.png) to get an idea of how you can setup Jenkins to do an automated build and deploy the newly built image.

![Continuous Deployments](https://github.com/sameerkasi200x/docker-ci-cd/blob/master/continuous-deployment-jenkins-and-docker-trusted-reg.png?raw=true)


# Jenkins Build Configuration
Use ssh method to do the build. Use below commands

### First we need to let our docker client talk to UCP (our orchestrator)
  * Download the Client Bundle from your UCP

```
cd /home/centos/ucp-bundle
eval $(<env.sh)
cd $WORKSPACE
```

  * Now perform a build and push to the dtr
```
docker build --pull=true -t dtr.example.com:12443/RepoOwner/twitter-app-ci-cd:$GIT_COMMIT ./code

docker login --username user --password myComplexPa$$w0rd dtr.example.com:12443/RepoOwner

docker push dtr.example.com:12443/RepoOwner/twitter-app-ci-cd:$GIT_COMMIT
```
  * Update the service
  
  You can skip this step if you don't want to do continuous deployments.
  
```
docker service update --force --update-parallelism 1 --update-delay 30s --image dtr.example.com:12443/RepoOwner/twitter-app-ci-cd:$GIT_COMMIT tweet-to-us 
```
The option ```--update-parallelism 1``` helps to update one instance of the container at a time. This is specially helpful when you are running your service in replicated mode with replicas of the same container. That way you won't notice a downtime for your service as upgrade will be rolled to all containers one by one. 

The option ```--update-delay``` helps you to have a delay in between the update of containers that way one of the underlying containers/tasks could be using old code and another one would be using new code. This (if you keep the delay long enough) can help in doing blue-green deployment to test the new code without impacting all the users.


Refer online doc for mode option - [docker service update](https://docs.docker.com/engine/reference/commandline/service_update/)


# Docker and DevOps
This is a small and just one of the examples of how powerful Docker could be in your DevOps culture

![DevOps](https://github.com/sameerkasi200x/docker-ci-cd/blob/master/1280px-Devops-toolchain.svg%5B1%5D.png?raw=true)

# Alternatives
  * For the purpose of demonstration I have used GitHub, you can host your code practically anywhere else as long your CI/CD tools allows receiving notification or polling. 
  * For the purpose of Demonstration I have used Docker Trusted Registry but you can use any other repository as well e.g. dockerhub
  * For the purpose of demonstration I have used Jenkins but you can also use any other CI tool or build and deployment tool
  * For the purpose of demo I have used HRM (and also becuase I found it easy to use) but you can also use other proxy feature/tool on top of Docker setup e.g. interlock. 
  * Or hey, build something of your own ;-)  e.g. You can also use S3 bucket to host the code and use a Lambda function to invoke CI pipeline using rest service offered by your CI tool to perform a build and push the image to your public repository on DockerHub which can be consumed by your end users when they need it.
