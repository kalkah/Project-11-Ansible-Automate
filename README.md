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

<img width="739" alt="image" src="https://github.com/kalkah/Project-11-Ansible-Automate/assets/95209274/4fd0724c-37e4-4a44-aff6-fc9e4f4f9529">

<img width="751" alt="image" src="https://github.com/kalkah/Project-11-Ansible-Automate/assets/95209274/698c71c9-d6da-4af7-98af-d68d90b01e66">

Connect to the Jenkins-Ansible Server via the Terminal using the SSH Client. The script below was used for the jenkins installation by creating jenkins.sh file

`sudo nano jenkins.sh`

```
#!/bin/bash

#jenkins installation script
#update the server repository
sudo apt update -y

#This is the Debian package repository of Jenkins to automate installation and upgrade. To use this repository, first add the key to your system:
  curl -fsSL https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key | sudo tee \
    /usr/share/keyrings/jenkins-keyring.asc > /dev/null

# Then add a Jenkins apt repository entry:
  echo deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] \
    https://pkg.jenkins.io/debian-stable binary/ | sudo tee \
    /etc/apt/sources.list.d/jenkins.list > /dev/null

# Update your local package index, then finally install Jenkins:
sudo apt-get update -y
sudo apt-get install fontconfig openjdk-11-jre -y
sudo apt-get install jenkins -y

echo "Installation is successful"
sudo systemctl status jenkins
```

The permission of the file was chnaged with the command below to make it executable: `sudo chmod +x jenkins.sh`

The script was run with the command `./jenkins.sh'

The script run successfully

<img width="662" alt="image" src="https://github.com/kalkah/Project-11-Ansible-Automate/assets/95209274/1757c0c7-6af4-45cd-ad2c-db1328db53b7">

we begin setting up Jenkins by accessing it via our browser using the following syntax:

`http://<Jenkins-Server-Public-IP-Address-or-Public-DNS-Name>:8080`

<img width="738" alt="image" src="https://github.com/kalkah/Project-11-Ansible-Automate/assets/95209274/96e00e67-bfb1-430b-9f41-21f09eae70a6">

As shown in the output image above, we are prompted to provide an Administrator password to unlock Jenkins. The password from can be retrievied from the Jenkins Server, using the following command: `sudo cat /var/lib/jenkins/secrets/initialAdminPassword`

The password was inputed and the suggested plugins were installed
<img width="742" alt="image" src="https://github.com/kalkah/Project-11-Ansible-Automate/assets/95209274/6bb15e64-e6e8-4bb4-b4f8-5dd9c2dda4f2">

<img width="746" alt="image" src="https://github.com/kalkah/Project-11-Ansible-Automate/assets/95209274/256c3cd1-e138-41b5-a0f5-71b2d6526869">

After completion plugins installation, Jenkins prompts to either  "Create a First Admin User" (we can either fill in our details to do this) or  "Skip and continue as admin". I decide to go with the latter option.

<img width="739" alt="image" src="https://github.com/kalkah/Project-11-Ansible-Automate/assets/95209274/eaae1ec8-f4b3-4ec4-a626-f94bca1692bc">

The elastic ip address was inserted in the instance configuration page,  and click on "Save and Finish".

<img width="742" alt="image" src="https://github.com/kalkah/Project-11-Ansible-Automate/assets/95209274/c0c87dc8-6065-4acd-ba35-e9608e7f1ac3">

The set-up is complete and Jenkins is ready to be used. We click on "Start using Jenkins" to move into the main Jenkins Environment.

<img width="744" alt="image" src="https://github.com/kalkah/Project-11-Ansible-Automate/assets/95209274/14d183d2-bb4b-4c63-a886-84d6300b7eef">

### Install and Configure Ansible to act as a Jump Server/Bastion Host

The next step involves the installation and configuration of Ansible as a Jump Server/Bastion Host. An SSH Jump Server/Bastion Host is a regular Linux server, accessible from the Internet, which is used as a gateway to access other Linux machines on a private network using the SSH protocol. The purpose of an SSH jump server is to be the only gateway for access to our infrastructure reducing the size of any potential attack surface. We shall proceed to implement this with the following steps:

**Creation of a new github repository** named **ansible-config-mgt** for ansible automation and configuration management. 

<img width="943" alt="image" src="https://github.com/kalkah/Project-11-Ansible-Automate/assets/95209274/d43be4b3-1826-4222-b406-2a1fa822a049">

**Ansible installation** on the Jenkins-Ansible server using the following commands:

`sudo apt update -y`     `sudo apt install ansible -y`

<img width="661" alt="image" src="https://github.com/kalkah/Project-11-Ansible-Automate/assets/95209274/6448e6d7-d85a-4496-a1b1-1ab38d72cd98">

Check for ansible version using this command: `ansible --version`

<img width="642" alt="image" src="https://github.com/kalkah/Project-11-Ansible-Automate/assets/95209274/d15fd5f5-1dfb-4187-b951-8a47493b465f">

**Create and Configure Jenkins Freestyle Build Job**
In the Jenkins web console, we create a new freestyle project that we will name ansible and point it to our GitHub ansible-config-mgt repository. We do this with the following steps:

From the Jenkins web console, we click on "New item"
<img width="951" alt="image" src="https://github.com/kalkah/Project-11-Ansible-Automate/assets/95209274/9e5a0b3a-e020-44cf-bac1-874c947669fd">

The project was named ansible and freestyle project was selected.
<img width="946" alt="image" src="https://github.com/kalkah/Project-11-Ansible-Automate/assets/95209274/95aa5ae5-0c3d-45bc-b007-e130c243d68e">

The project url was retrieved from github: (`https://github.com/kalkah/Ansible-config-mgt`) and inserted in the  in the Jenkins configuration page, we click on the "GitHub project" check box and we paste in the "project URL".

<img width="659" alt="image" src="https://github.com/kalkah/Project-11-Ansible-Automate/assets/95209274/1d93147b-0cb1-4929-87b9-99c5eeba14cc">

The repository URL was also retrieved from the github (https://github.com/kalkah/Ansible-config-mgt.git) and inserted in the n the Jenkins configuration page, under "Soure Code Management" we click on "Repository URL" and we paste in the remote link for our GitHub repository.

<img width="947" alt="image" src="https://github.com/kalkah/Project-11-Ansible-Automate/assets/95209274/e996b812-bd9c-429e-a7e0-27e538734169">

Under "Branch Specifier", we change */master to */main. Click apply and save.

<img width="943" alt="image" src="https://github.com/kalkah/Project-11-Ansible-Automate/assets/95209274/23d1a9a3-3234-419c-96f3-18d15417d852">

In the Jenkins web console, we go to the left pane and click on "Build Now" and if the build is successful, we will see it under "Build History" as seen in the image below:

<img width="944" alt="image" src="https://github.com/kalkah/Project-11-Ansible-Automate/assets/95209274/7fb5e122-9e38-4997-b3ea-4666396a5cea">

To view more details about the successful build, we click on the drop down icon beside the build and we select "Console Output".

<img width="928" alt="image" src="https://github.com/kalkah/Project-11-Ansible-Automate/assets/95209274/49874bdd-5a30-41fa-9142-d9c63ac665a1">

**Configure Jenkins Build Job to Archive Repository Content everytime there are changes**
After creating and configuring the ansible freestyle project, we had to trigger it manually for it to run. But we can go a step further. To enable our build run automatically whenever there is a change in our Git repository, we need to enable Webhooks in our GitHub repository settings and configure it to trigger ansible build:

We go to the settings on the **ansible-config-mgt** repository and configure webhooks. On the Webhooks/Add webhook page, under "Payload URL", we input our elastic IP address using the following syntax:  `http://<Jenkins server IP address>/github-webhook/` (http://13.49.9.69:8080/github-webhook)

Under "Content type", we select "application/json"

<img width="944" alt="image" src="https://github.com/kalkah/Project-11-Ansible-Automate/assets/95209274/b9254339-bdc3-4fab-b00d-958b49912cfd">


