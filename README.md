# **NGINX LOAD BALANCER**

**Description and Goal:** Our goal is to create a load balancer with Nginx and distribute the traffic load among several applications by using Docker containers.

There are two ways to achieve the goal.
 1. Running the Nginx in Docker and balancing the load by distributing traffic among others Docker containers where our applications are deployed.

2. Running Nginx on a server as a host without Dockerizing and balancing the load by distributing traffic among other Docker containers where our applications are deployed.

Here we are going to use process 1.

**Advantage and Disadvantage:** Advantage is we can deploy anytime anywhere by launching the containers. Disadvantage is every time when we need to update and deploy the Nginx container it should be stopped first and then run again which will create downtime.

**Step 1: Creating a sample application and push it in GIT Repository**

For example, here we are going to create a flask application.

Start with creating a new directory; let&#39;s call it service
```sh
mkdir service
```
Go inside the folder and create basic app.py file for your basic application:

```sh
# flask_web/app.py
from flask import Flask
app = Flask(__name__)
 
@app.route('/')
def hello_world():
       return 'Welcome to the Docker container of Service'
 
if __name__ == '__main__':
   app.run(debug=True, host='0.0.0.0')
```
Now, we need to include Flask in our requirements.txt file:
```sh
Flask==0.10.1
```
Create docker file with the name Dockerfile
```sh
FROM ubuntu:latest
 
RUN apt-get update -y && \
   apt-get install -y python3-pip python-dev
 
# We copy just the requirements.txt first to leverage Docker cache
COPY ./requirements.txt /app/requirements.txt
 
WORKDIR /app
 
RUN pip3 install -r requirements.txt
 
COPY . /app
 
ENTRYPOINT [ "python3", "app.py" ]
# CMD [ "app.py" ]

```
Now we need to push the code in git
```
git init
git add .
git push -u origin master

```

**Step 2: Deploy Docker applications in server**
Connect your server through SSH and login. Update server and install Docker Engine
```sh
ssh -i my_server.pem ubuntu@serverip

```
```sh
sudo apt update
sudo apt  install docker.io
```
**Step**

**Create Three folders for three projects** where one is our Nginx Load Balancer name &quot;lb&quot; and the other two is service1 and service2. We can create service1 and service2 when cloning the project service from git.
 For example,
```sh
git clone https://github.com/tanvir4hmed/service.git service1
```
```sh
git clone https://github.com/tanvir4hmed/service.git service2
```
```sh
mkdir lb
```
Output will look like

![](https://github.com/tanvir4hmed/service/blob/main/readme_images/image_1.png)

Now go to the service1 folder and modify the app.py by replacing the word &#39;Service&#39; with &#39;Service 1&#39; in return so we can recognize the service1 project.

Build Docker image for service1 and run the image
and
then go to the service2 folder and modify the app.py by replacing the word &#39;Service&#39; with &#39;Service 2&#39; in return so we can recognize the service2 project.
Build Docker image for service2 and run the image

App.py changed output:

![](https://github.com/tanvir4hmed/service/blob/main/readme_images/image_2.png)


To do so execute these commands:
```sh
cd service1
nano app.py
# Change the word service with service 1, save and exit from editor
sudo docker build -t service1 .
cd ..
cd service2
nano app.py
# Change the word service with service 1, save and exit from editor
sudo docker build -t service2 .
sudo docker run -d -p 8000:5000 service1
sudo docker run -d -p 8001:5000 service2
```
Docker run output will look like:

![](https://github.com/tanvir4hmed/service/blob/main/readme_images/image_3.png)


**Step 3: Deploy NGINX Load Balancer with Docker in server**
To do that we need to know the ip of service1 and service2.
Lets inspect our docker container and note the ip from the bottom of the output.

![](https://github.com/tanvir4hmed/service/blob/main/readme_images/image_4.png)

Do the same for the service2 container inspecting the container id from the output of docker ps.
Write it down for using it in nginx.conf. For example,
Service1 ip 172.17.0.2 which is a private ip of the container
Service2 ip 172.17.0.3 which is a private ip of the container

Now go to the folder lb and create two files, one is Dockerfile and other is nginx.conf

![](https://github.com/tanvir4hmed/service/blob/main/readme_images/image_5.png)

 In Dockerfile write these
```sh
FROM nginx
COPY nginx.conf  /etc/nginx/
```
In nginx.conf use the code below and replace with your container ip that was noted
```sh
user www-data;
worker_processes auto;
pid /run/nginx.pid;
include /etc/nginx/modules-enabled/*.conf;
 
events {
}


http {
 upstream app {
   server 172.17.0.2:8000;
   server 172.17.0.3.:8001;
}
 server{
   listen 80;
   location /{
 
     proxy_pass http://app/;
   }
}
}
```

Now build image and run the container of lb
```sh
sudo docker build -t lb .
sudo docker run -d -p 80:80
```
**Step 4: Checking the traffic flow from inside of lb container using tcpdump**
To do that we need container id of lb to go inside the lb container by using docker exec and then install some tools in that lb container
```sh
docker ps
sudo docker exec -it ld_container_id bash
# Now we are inside of the container and will run the commands below
apt update
apt install net-tools
ifconfig #To check the interface name, for me its eth0
apt install tcpdump
tcpdump -i eth0

```

Now open a new terminal, login to the server and curl with lb container ip 2 times
```sh
sudo docker inspect lb_container_id
#Note the ip from the bottom
curl http://172.17.0.5:80 # for me lb container ip is 127.17.0.5
# Hit again
curl http://172.17.0.5:80
```

**Final Output** can be found if we check the other terminal where tcpdump was running.
 To make it easy you can copy the output and search by the prefix of our ip such as &#39;127.17.0&#39;
 We can see two different response from service1 and service2 by finding the ip 127.17.0.2 and 127.17.0.3
```sh
15:41:48.715573 IP 02dede94c403.80 > 
15:41:48.715659 IP ip-172-17-0-3.ap-south-1.compute.internal.8001 > 02dede94c403.45274: Flags [R.], seq 0, ack 802058568, win 0, length 0
15:41:48.715704 IP 02dede94c403.47666 > ip-172-17-0-2.ap-south-1.compute.internal.8000: Flags [S], seq 4164453549, win 64240, options [mss 1460,sackOK,TS val 1141177490 ecr 0,nop,wscale 7], length 0
```
