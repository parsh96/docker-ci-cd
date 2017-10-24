## Running with docker run command

docker container run --rm -d --name tweet-to-us -p 80 -p 443 -v $(pwd):/usr/share/nginx/html nginx

## Build and Run as service
Optionally you can also build using this Dockerfile

docker image build -t tweet-to-us:docker-service . 

docker container run -p 80 -p 443 -d tweet-to-us:docker-service


Credits: This is built using the code from - https://github.com/mikegcoleman/hybrid-workshop
