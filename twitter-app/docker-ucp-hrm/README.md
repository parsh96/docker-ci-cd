## Build and push

Build the image using below command

docker image build -t $dtr-url/$dtr-user/tweet-to-us:docker-ucp-hrm-b1 .
docker push $dtr-url/$dtr-user/tweet-to-us:docker-ucp-hrm-b1
e.g.

(*)Login if needed

docker login dtr.ashnikdemo.com:12443/ashnik --username ashnik --password password

docker image build --build-arg 'constraint:ostype==linux' -t dtr.ashnikdemo.com:12443/ashnik/tweet-to-us:docker-ucp-hrm-b1 .
     
docker push dtr.ashnikdemo.com:12443/ashnik/tweet-to-us:docker-ucp-hrm-b1

build-arg will help to run this only on a Linux machine if your UCP cluster has Windows and Linux worker nodes

## Pull and run a service with HRM 

docker service create --network ucp-hrm --name tweet-to-us --mode replicated --replicas 2 --label com.docker.ucp.mesh.http.80="external_route=http://tweet-app.demoapps.ashnikdemo.com/,internal_port=80" --constraint 'node.platform.os==linux' dtr.ashnikdemo.com:12443/ashnik/tweet-to-us:docker-ucp-hrm-b1 

You can also do this from the UCP

Make sure that you go to Environment Tab and enter the Label
Key = com.docker.ucp.mesh.http.80
Value = external_route=http://tweet-app.demoapps.ashnikdemo.com/,internal_port=80

## Make a change and roll to production - Rolling Update

Make changes and rebuild

rm -f index.html

cp  index-with-mistake.html index.html

docker image build --build-arg 'constraint:ostype==linux' -t dtr.ashnikdemo.com:12443/ashnik/tweet-to-us:docker-ucp-hrm-b2 .

docker push dtr.ashnikdemo.com:12443/ashnik/tweet-to-us:docker-ucp-hrm-b2


Update the service

docker service update --image dtr.ashnikdemo.com:12443/ashnik/tweet-to-us:docker-ucp-hrm-b2 tweet-to-us


This time you would see a different message.


## Rollback

 docker service update --rollback   tweet-to-us
