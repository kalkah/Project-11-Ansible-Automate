# Project-11-Ansible-Automate
## Ansible Configuration Management

The project will show how deploying Ansible will help to simplify complex tasks and streamline IT infastructure. We will also work with Jenkins to configure and execute build jobs whilst writing code using declarative language such as YAML. This project is 3-in-1 Project, further automation is built on the NFS server with installation of Jenkins, Ansible, and Load balancer.

## Introduction to Ansible Configuration Management

The goal of this project is to demonstrate Ansible's powerful automation capabilities. Ansible is an open source, command line software application that is used for automating IT operations such as deploying applications, managing configurations scaling infrastructure and other activities involving many repetitive tasks. Ansible's main strengths are simplicity and ease of use. It lets IT professionals quickly and easily deploy multi tier apps. Rather than needing to write lengthy code to automate our systems, we simply list the tasks that require automation by writing a Playbook and Ansible figures out how to get our systems to the state we need them to be in. It is also important to note Ansible has a strong focus on security and reliability and as such it has very minimal moving parts. A great exapmle of this is that Ansible is agentless. This means that the devices or infrastructure it monitors do not require any proprietary software agent to be installed on them beforehand. Ansible has two major types of files:

1. The Ansible inventory file defines the hosts and groups of hosts upon which commands, modules, and tasks in a playbook operate.

2. Ansible Playbooks are lists of tasks that are automatically executed for the specified inventory or groups of hosts. One or more Ansible tasks can be combined to make a play — an ordered grouping of tasks mapped to specific hosts—and tasks are executed in the order in which they are written.

![image](https://github.com/kalkah/Project-11-Ansible-Automate/assets/95209274/88c661ea-d5dc-46b4-aa38-d91fb1d98f2d)

## Ansible as Jump Server

A Jump Server (sometimes also referred as Bastion Host) is an intermediary server through which access to internal network can be provided. If you think about the current architecture we are working on, ideally, the webservers would be inside a secured network which cannot be reached directly from the Internet. That means, even DevOps engineers cannot SSH into the Web servers directly and can only access it through a Jump Server – it provides better security and reduces attack surface.

On the diagram below, the Virtual Private Network (VPC) is divided into two subnets – Public subnet has public IP addresses and Private subnet is only reachable by private IP addresses.

![image](https://github.com/kalkah/Project-11-Ansible-Automate/assets/95209274/d3ba010c-3420-4f9e-b34f-ef9f5095795a)

Further on in our future projects, we will implement a proper Bastion Host. But for now, we will make do with developing Ansible scripts to simulate the use of a Jump Box/Bastion Host to access our Web Servers. Therefore, based on the infrastructure to be used and the goal of this project, it shall consist of three parts:

1. Install and Set-up Jenkins on EC2 Instance.

2. Install and Configure Ansible to act as a Jump Server/Bastion Host.

3. Begin Ansible Development, Set up an Inventory and Create a Simple Ansible Playbook to Automate Server Configuration.

The final architecture is shown below:

![image](https://github.com/kalkah/Project-11-Ansible-Automate/assets/95209274/a58fa0d8-6d60-4775-a2c9-61907b0c4ac6)

## Nginix Load Balancer Installation

nginix-install.sh was opened with the command below

`sudo nano nginx-install.sh`

The code below was inserted into the nginx-install.sh

```
#!/bin/bash

######################################################################################################################
##### This automates the configuration of Nginx to act as a load balancer
##### Usage: The script is called with 3 command line arguments. The public IP of the EC2 instance where Nginx is installed
##### the webserver urls for which the load balancer distributes traffic. An example of how to call the script is shown below:
##### ./configure_nginx_loadbalancer.sh PUBLIC_IP Webserver-1 Webserver-2
#####  ./configure_nginx_loadbalancer.sh 127.0.0.1 192.2.4.6:8000  192.32.5.8:8000
############################################################################################################# 

PUBLIC_IP=$1
firstWebserver=$2
secondWebserver=$3

[ -z "${PUBLIC_IP}" ] && echo "Please pass the Public IP of your EC2 instance as the argument to the script" && exit 1

[ -z "${firstWebserver}" ] && echo "Please pass the Public IP together with its port number in this format: 127.0.0.1:8000 as the second argument to the script" && exit 1

[ -z "${secondWebserver}" ] && echo "Please pass the Public IP together with its port number in this format: 127.0.0.1:8000 as the third argument to the script" && exit 1

set -x # debug mode
set -e # exit the script if there is an error
set -o pipefail # exit the script when there is a pipe failure


sudo apt update -y && sudo apt install nginx -y
sudo systemctl status nginx

if [[ $? -eq 0 ]]; then
    sudo touch /etc/nginx/conf.d/loadbalancer.conf

    sudo chmod 777 /etc/nginx/conf.d/loadbalancer.conf
    sudo chmod 777 -R /etc/nginx/

    
    echo " upstream backend_servers {

            # your are to replace the public IP and Port to that of your webservers
            server  "${firstWebserver}"; # public IP and port for webserser 1
            server "${secondWebserver}"; # public IP and port for webserver 2

            }

           server {
            listen 80;
            server_name "${PUBLIC_IP}";

            location / {
                proxy_pass http://backend_servers;   
            }
    } " > /etc/nginx/conf.d/loadbalancer.conf
fi

sudo nginx -t

sudo systemctl restart nginx
```

The permission of the file was chnaged with the command below to make it executable: `sudo chmod +x nginx-install.sh`

The script was run using this command: `./nginx-install.sh 51.20.79.124 13.53.214.97:8000 51.20.133.184:8000`  (./nginx-install.sh PUBLIC_IP nginxloadbalancer Webserver-1 Webserver-2)

The script run successfully

<img width="659" alt="image" src="https://github.com/kalkah/Project-11-Ansible-Automate/assets/95209274/74e4e07a-58e6-4c42-9cc6-01bdc03e16d5">

The `/etc/hosts` file was updated  for local DNS with Web Servers names (e.g. Webserver1 and Webserver2) and their local IP addresses.

<img width="669" alt="image" src="https://github.com/kalkah/Project-11-Ansible-Automate/assets/95209274/798035d5-3780-455c-bb84-6ee39675474f">

### Jenkins Installation

Jenkins is an open-source automation tool written in Java with plugins built for continuous integration. Jenkins is used to build and test software projects continuously making it easier for developers to integrate changes to the project, and making it easier for users to obtain a fresh build. It also allows developers to continuously deliver software by integrating with a large number of testing and deployment technologies. We deploy Jenkins by implementing the following steps:

An EC2 instance was launched using the free tier eligible version of Ubuntu Linux Server 22.04 LTS (HVM). The server was named Jenkins-Ansible server

Port 8080 was opened in the security group inbounds rules to allow traffic from anywhere. Jenkins uses TCP Port 8080 by default.

Elastic IP address was created and allocated to the Jenkins-Ansible server: Considering that we'll be using Jenkins with Github and configuring Web Hooks in this project, it will make our job easier to create and allocate an elastic IP adress to our Jenkins-Ansible Server. This is beacuse everytime we stop/start the server, there will be a need to keep reconfiguring Github Web Hooks to a new IP address. Having an elastic IP address (which will not change when we stop/start the server) is the ideal way to overcome this issue.

