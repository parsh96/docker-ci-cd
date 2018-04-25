# Using UCP HRM for rolling upgrade
This post demonstrates how you can use Docker Enterprise Edition for CI/CD and rolling upgrade. The post uses a simple html file and Nginx image to explain and demonstrate how you can
   
  *	Use HTTP Routing Mesh (HRM) feature of UCP
  *	Use HRM for rolling upgrade
  *	Use Jenkins and Github along with Docker for continuous deployments

The commands provided here are for example purpose but most of them can also be used in a sandbox environment for testing purpose. Pre-requisite for the examples to work fine- 

  *  You have a full-fledged Docker Enterprise Edition deployment with Universal Control Plane (UCP) cluster setup.
  *  You have a Docker Trusted Registry (DTR) setup for the UCP cluster.
  *  You have HRM enabled in the UCP cluster with neccessary setup of application load-balancer and DNS configuration.

# Swarm Routing Mesh
Before starting let's take a while to understand the basics of networking in a Docker Swarm Cluster. Before we get into HRM, let's take a look at Swarm Routing Mesh which uses an ingress network to expose a container port on all the members of the swarm cluster. A schematic diagram of Swarm Routing Mesh is depicted in the below diagram
</br> 
<img src="https://github.com/sameerkasi200x/docker-ci-cd/blob/master/swarm-routing-mesh.png?raw=true" alt="Swarm Routing Mesh" width="600" />
</br>
Using Swarm routing mesh can makes it easier for you to configure new services as you can expore the port on all the nodes and traffic will be routed by the ingress overlay network. You don't need to configure a load balancer for each and every service.

But Swarm Routing Mesh suffers one limitation - still each service is listening on a different port and this would pose some challenges in enterprise environment where you will have to roll out new services rapidly. This is where value of HTTP Routing Mesh comes up.

# HTTP Routing Mesh
A schematic explanation of HTTP Routing Mesh is as described in the below diagram

</br>
<img src="https://github.com/sameerkasi200x/docker-ci-cd/blob/master/http-routing-mesh.png?raw=true" alt="HTTP Routing Mesh" width="600" />
</br>
As you can see in the diagram the you don't need to expose different ports and all traffic is routed via http ports.


# Docker Build and Deployment

Now that we have understood the concept of Swarm Routing Mesh and HTT Routing Mesh, let's look the docker commands which will be useful for performing a Docker based build and deployment for a given application.

### Get Client Bundle from UCP
UCP offers secure Role Based Access Control and user management on top of a Docker Swarm Cluster. In my example I will be using secure login for accessing UCP and DTR. 

You can follow the online Docker documentation to understand [how to download Client Bundle](https://docs.docker.com/datacenter/ucp/2.2/guides/user/access-ucp/cli-based-access/#download-client-certificates).

### Build and push

  * Use ```docker image build``` command to build an image using the latest code from code repoistory

		docker image build -t $dtr-url/$dtr-RepoOwner/tweet-to-us:docker-ucp-hrm-b1 .
		
  * Use ```docker push``` command to push an image to DTR

    		docker push $dtr-url/$dtr-user/tweet-to-us:docker-ucp-hrm-b1
	
	> + The ```twitter-app-ci-cd``` repository should exist in advance
	> + The ```$dtr-url``` is the URL which points to DTR
	> + ```$dtr-RepoOwner``` under whom the repostiory twitter-app-ci-cd been placed  


### Login if needed

Ideally you should not be required to login when you are using UCP Client Bundle. The username and certificates will be used for single signon against UCP and DTR. But you would be required to login if you want to first perform a local build and then push the image to DTR.

	docker login dtr.example.com:12443/RepoOwner --username user --password MyComplexPa$$w0rd

	docker image build --build-arg 'constraint:ostype==linux' -t dtr.example.com:12443/RepoOwner/tweet-to-us-hrm:b1 .

	docker push dtr.example.com:12443/RepoOwner/tweet-to-us-hrm:b1

>```build-arg``` will help to run this only on a Linux machine if your UCP cluster has Windows and Linux worker nodes

### Pull and run a service with HRM 

Once you have pushed the image to the repository you can use the image for deployment on a UCP cluster.

```
	docker service create --network ucp-hrm --name tweet-to-us --mode replicated --replicas 2 --label com.docker.ucp.mesh.http.80="external_route=http://tweet.app.example.com/,internal_port=80" --constraint 'node.platform.os==linux' dtr.example.com:12443/RepoOwner/tweet-to-us-hrm:b1
```

You can also do deployments from from the UCP portal/GUI. When you do a deployment from GUI, you can still provide the HRM configuration label. IN order to do so, make sure that you go to Environment Tab and enter the Label

>Key = ```com.docker.ucp.mesh.http.80```
>
>Value = ```external_route=http://tweet.app.example.com/,internal_port=80```


To deploy on UCP cluster, you will require UCP Client Bundle and the user in the Client Bundle should have access to the repository where you have pushed the image.


### Running with Docker EE 2.0 (Interlock)
If you are running latest version of UCP, you would be using interlock. Here is a command that might help you get start
```
docker service create --network ucp-interlock --name tweet-to-us --mode replicated \
        --replicas 2 \
        --label com.docker.lb.hosts="tweet.apps.example.com" \
        --label com.docker.lb.network="ucp-interlock" \
        --label com.docker.lb.port="80" \
        --constraint 'node.platform.os==linux' \
        --detach=true \
        $dtr_url/development/tweet-to-us:b1
```

Note that the labels have changed. So if you are using UCP portal, you need to use below label:
```com.docker.lb.hosts``` for providing the hostname by which the service will be accesed. ```com.docker.lb.network``` for the network on which is used by interlock proxy service. ```com.docker.lb.port``` is the port on which the containers that belong to your service would be listening.



### Make a change and roll to production 

Make changes to ```index.html``` and rebuild. In reallife scenario this could be your Java or Python file containing application logic.  

	docker image build --build-arg 'constraint:ostype==linux' -t dtr.example.com:12443/RepoOwner/tweet-to-us-hrm:b2 .

Optionally, you can also include unit test cases in your code and spin up a container using ```docker container run ``` command to verify the unit cases.

Once the changes are done then build again and push to DTR repo

	docker push dtr.example.com:12443/RepoOwner/tweet-to-us-hrm:b2

### Update the service - Rolling Update

Now you can use ```docker service update``` command to update the service with newly built image.

	docker service update --image dtr.example.com:12443/RepoOwner/tweet-to-us-hrm:b2 tweet-to-us

This time you would see a different message as per your edit in ```index.html```.

### Rollback

There could be a case when the changes you have made to code does not work very well after publishing to production, testing or integration environment. Docker service provides an easy way of rollbacking the previous change made by using ```docker service update``` comamnd

	docker service update --rollback   tweet-to-us

# Continuous Integration

We discussed how UCP, HRM, docker service and DTR can be used for performing rolling upgrade. You can go a step further and combine your Jenkins pipeline with Docker build. Jenkins pipeline can automate the build your application code. For better understanding of the concept and how the setup could be done, you can refer to the GitHub & Jenkins integration guide on Docker sucess portal - [GitHub-Jenkins-Integration.pdf](https://www.docker.com/sites/default/files/UseCase/RA_CI%20with%20Docker_08.25.2015.pdf). It gives details of how you can create a Webhook in GitHub repo which can send a notification to your CI Server (Jenkins in this case) to trigger a Docker based build on every new change in the repo. The diagram below depicts the flow of events. 

</br>
<img src="https://github.com/sameerkasi200x/docker-ci-cd/blob/master/continuous-integration-jenkins-and-docker-trusted-reg.png?raw=true"  alt="Jenkins - GitHub - Docker integration" width="800" />

</br>
You can extend this diagram to do a continuous integration of multiple services or introduce unit test cases into the flow after the image build.


# Continuous Deployment
We can go a step further from our idea of continuous builds and do continuous deployments if the code is promoted to the production repository. Refer to the diagram below to get an idea of how you can setup Jenkins to do an automated build and deploy the newly built image.

</br>
<img src="https://github.com/sameerkasi200x/docker-ci-cd/blob/master/continuous-deployment-jenkins-and-docker-trusted-reg.png?raw=true"  alt="Continuous Deployments" width="800" />
</br>

# Jenkins Build Configuration
The commands we discussed above can be put into an atuomated build of a CI/CD tool for continuous testing and deployment. 

### Jenkins Task

  * Choose a slave which can access the UCP via docker client and fire docker command (i.e. has docker client installed).
  * Setup a new Jenkins task/project and use tags to run it on specific Jenkins Slaves.
  * Use ssh method for executing the Task. We will discuss the commands to be used in further sections.

### Setup Client Bundle

  * First we need to setup docker UCP client  bundle so that the docker client on the Jenkins Slave can talk to UCP (our orchestrator).
  * Download the Client Bundle from your UCP and set it up on the Jenkins Slave(s) which will be used for executing the build. Now we will be using the UCP Client Bundle for setting up the build environment in Jenkins Pipeline.

		cd /home/centos/ucp-bundle
		eval $(<env.sh)
		cd $WORKSPACE
	
	> + In my Example the client bundle is placed under ```/home/centos/ucp-bundle```. Make sure that this directory is acceesible by the user which is being used by Jenkins Master to connect to your Jenkins slave
	> 
	> + ```WORKSPACE``` will translate into Jenkins workspace
 

### Build and Push 
  * Now we would need to perform a build to build a docker image using the code.
  * If the build is successful, the image built could be pushed to the DTR.
  * You can have optional steps to run test cases using test cases written using frameworks (for example JUnit etc), before you push to the DTR.
  
		docker build --pull=true -t dtr.example.com:12443/RepoOwner/twitter-app-ci-cd:$GIT_COMMIT ./code

		docker login --username user --password myComplexPa$$w0rd dtr.example.com:12443/RepoOwner

		docker push dtr.example.com:12443/RepoOwner/twitter-app-ci-cd:$GIT_COMMIT
	> + ```GIT_COMMIT``` is the commit tag returned by git while pulling the code base
	> 
	> + Make sure that the ```user``` being used for ```docker login``` and the user in Client Bundle are same.
	>  
	> + Ensure that the user ```user``` has access to the repositories of user ```RepoOwner```.
	> 
	> + The ```twitter-app-ci-cd``` repository should exist in advance for the user ```RepoOwner``` 
	>
	> + ```dtr.example.com:12443``` is the url for the Docker Trusted Registery(DTR)

### Service Update
  * You can use docker service update command to update the service
  * You can skip this step if you don't want to do continuous deployments.

		docker service update --force --update-parallelism 1 --update-delay 30s --image dtr.example.com:12443/RepoOwner/twitter-app-ci-cd:$GIT_COMMIT tweet-to-us 

	> + The option ```--update-parallelism 1``` helps to update one instance of the container at a time. This is specially helpful when you are running your service in replicated mode with replicas of the same container. That way you won't notice a downtime for your service as upgrade will be rolled to all containers one by one. 
	>
	> + The option ```--update-delay``` helps you to have a delay in between the update of containers. This would phase-wise rollout of features wherein one of the underlying containers/tasks could be using old code and another one would be using new code. This (if you keep the delay long enough) can help in doing blue-green deployment to test the new code without impacting all the users.


  * Refer online doc for mode option - [docker service update](https://docs.docker.com/engine/reference/commandline/service_update/)

# Docker and DevOps

This is just one of the use cases of how pivotal a role Docker could play in your DevOps culture. It has lot of features which could overlap with the automation or configuration management tool you are using for putting up infrastructure-as-code or with the application integration and deployment tools you are using for automated build and deployments. But at the same time Docker offers great level of extensibility by means of its APIs, commands and seamless expreience across different environments. The real power as we discovered in this example is in combining these tools together. 

</br>
<img src="https://github.com/sameerkasi200x/docker-ci-cd/blob/master/1280px-Devops-toolchain.svg%5B1%5D.png?raw=true"  alt="DevOps" width="600" />
</br>

The example we saw here could be implemented for test environments to roll out new features for testing on continuous basis. The CI/CD pipeline discussed here could also be used for continuously updating microservices in a large enterprise deployment without affecting the user experience and with minimal downtime.  

# Alternatives

  * For the purpose of demonstration I have used GitHub, you can host your code practically anywhere else as long your CI/CD tools allows receiving notification or polling.
  * For the purpose of demonstration I have used Docker Trusted Registry. Docker Trusted Registery is a fully secured Docker Image Repository which can be hosted on-prem or private cloud. You can use any other repository as well e.g. dockerhub
  * For the purpose of demonstration I have used Jenkins but you can also use any other CI tool or build and deployment tool.
  * For the purpose of demo I have used HRM (and also becuase I found it easy to use) but you can also use other proxy feature/tool on top of Docker setup e.g. interlock. Alternatively you can also publish your service directly on a port using the port mapping. In that case you need to ensure firewall setting if the service is going to be consumed externally (outside the Docker network).
  * Or hey, build something of your own e.g. You can also use S3 bucket to host the code and use a Lambda function to invoke CI pipeline using rest service offered by your CI tool to perform a build and push the image to your public repository on DockerHub which can be consumed by your end users when they need it.
