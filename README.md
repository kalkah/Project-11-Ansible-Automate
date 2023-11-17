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

The permission of the file was changed with the command below to make it executable: `sudo chmod +x nginx-install.sh`

The script was run using this command: `./nginx-install.sh 51.20.79.124 13.53.214.97:8000 51.20.133.184:8000`  (./nginx-install.sh PUBLIC_IP nginxloadbalancer Webserver-1 Webserver-2)

The script run successfully

<img width="659" alt="image" src="https://github.com/kalkah/Project-11-Ansible-Automate/assets/95209274/74e4e07a-58e6-4c42-9cc6-01bdc03e16d5">

 open the Nginx configuration file with the following command:    `sudo nano /etc/nginx/conf.d/loadbalancer.conf'

We copy and paste in the configuration file below to configure nginx to act as a load balancer. As can be seen in the file, necessary information like Public IP and Port Number for our three web servers are provided. We also need to provide the Public IP address of our Nginx Load Balancer.
```
        upstream backend_servers {

            # your are to replace the public IP and Port to that of your webservers
            server 51.20.82.142:80; # public IP and port for webserver 1
            server 51.20.131.231:80; # public IP and port for webserver 2
            server 51.20.122.38:80; # public IP and port for webserver 3

        }

        server {
            listen 8080;
            server_name 51.20.190.36; # provide your load balancers public IP address

            location / {
                proxy_pass http://backend_servers;
                proxy_set_header Host $host;
                proxy_set_header X-Real-IP $remote_addr;
                proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            }
        }
```

<img width="662" alt="image" src="https://github.com/kalkah/Project-11-Ansible-Automate/assets/95209274/dbdbc8f4-3f37-416a-acda-ea360c842a72">

The following serves to break down the configuration file above and explain in more detail:

    upstream backend_servers defines a group of back end servers (our two web servers).

    The server lines inside the upstream block lists the ports and public IP addresses of both of our backend webservers.

    proxy_pass inside the location block sets up the load balancing, passing the request to the back end servers.

    The proxy_set_header lines pass necessary headers to the backend servers to correctly haandle the requests.

The command `sudo nginx -t` was used to test our configuration:

<img width="401" alt="image" src="https://github.com/kalkah/Project-11-Ansible-Automate/assets/95209274/a9b6814d-d209-4dea-956e-a718b74bd074">

The `/etc/hosts` file was updated  for local DNS with Web Servers names (e.g. Webserver1 and Webserver2) and their local IP addresses.

<img width="669" alt="image" src="https://github.com/kalkah/Project-11-Ansible-Automate/assets/95209274/798035d5-3780-455c-bb84-6ee39675474f">

we execute the following command `sudo systemctl restart nginx`to restart the Nginx service and load our new configuration.

Paste in the public IP address of our Nginx loadbalancer (syntax is: http://<Public-IP-Address>:8080) into the browser: **http://51.20.79.124**
<img width="958" alt="image" src="https://github.com/kalkah/Project-11-Ansible-Automate/assets/95209274/dc415e6e-4a31-41f3-814e-517bfcd3b964">

As shown in the image above, our load balancer is able to serve content from the tooling website on our web servers.

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

In Jenkins, on the `ansible` project page, click on configure, under "Build Triggers", we check the box for "GitHub hook trigger for GITScm polling"

<img width="880" alt="image" src="https://github.com/kalkah/Project-11-Ansible-Automate/assets/95209274/3efd23fc-6d21-4623-bd53-8b5a495b6ad2">

Next to ensure Jenkins saves all files also known as "Build Artifacts", we go under Post-build Actions, we click on Add post-build Action and we select "Archive the artifacts"

<img width="947" alt="image" src="https://github.com/kalkah/Project-11-Ansible-Automate/assets/95209274/3472ab9b-57bb-4175-9cb5-a22144d6a908">

In the dialogue box under "Files to archive", we simply enter ** which refers to all available paths in the working space. Then we click on "Apply" and "Save" at the bottom of the page.

<img width="950" alt="image" src="https://github.com/kalkah/Project-11-Ansible-Automate/assets/95209274/99fbc1d6-db8b-485f-ad19-2ff42769b149">

To test the above set up, some changes were made to the README.md file in our ansible-config-mgt GitHub repository.

<img width="958" alt="image" src="https://github.com/kalkah/Project-11-Ansible-Automate/assets/95209274/af50c5f6-b366-4e93-b5c4-bd7dcef1e634">

The build started automatically and Jenkins saved the files (build artifacts).The following commands were used to further confirm this:

`ls /var/lib/jenkins/jobs/ansible/builds/3/archive/`        `sudo cat /var/lib/jenkins/jobs/ansible/builds/3/archive/README.md`

<img width="635" alt="image" src="https://github.com/kalkah/Project-11-Ansible-Automate/assets/95209274/ee2eb3c3-cd12-49f1-a7e0-b91a504f82e8">

**Configure Jenkins to Connect and Copy Files to NFS Server**

In this step, we will proceed to configure Jenkins to connect via SSH and copy files to the `/mnt/opt` directory in the NFS server we deployed for the Tooling Website Solution. To begin this implementation, we will require a plugin called "Publish Over SSH". We install this by clicking on the "Manage Jenkins", under the System configuration page, click on "Plugins" and search for "Publish Over SSH". ckick on "Install"

<img width="960" alt="image" src="https://github.com/kalkah/Project-11-Ansible-Automate/assets/95209274/9d23bafa-fb41-44c4-a7fd-5af1307e63a9">

We configure the "Publish Over SSH" plugin and our ansible job/project to copy our build artifacts to the NFS server. On the Jenkins Dashboard click on the "Manage Jenkins", under the System configuration page, click on "System". scroll down to "Publish Over SSH" configuration section

<img width="935" alt="image" src="https://github.com/kalkah/Project-11-Ansible-Automate/assets/95209274/a86eafea-59f5-4679-878e-44ecf7bad000">

Under "Key", we put in the content of .pem file that we used in connecting to to the NFS server via SSH.

<img width="865" alt="image" src="https://github.com/kalkah/Project-11-Ansible-Automate/assets/95209274/37b251d9-e0c2-4246-86c4-8073e9c7c3a2">

We need to input configuration more configuration to enable connection to the NFS Server. Under "SSH Servers", we click on the "Add" button and then we input the following:

Name: This can be any arbitrary name.

Hostname: This will be the private IP address of the NFS server (172.31.28.109).

Username: For this we will use `ec2-user` which is the username used for Red Hat Enterprise Linux based servers such as our NFS Server.

Remote directory: Here, we will use the `/mnt/opt` directory which we specified we will be using for Jenkins in our Tooling Website Solution project.

Next is to test the configuration and ensure the connection returns "Success". We should note that TCP Port 22 on our NFS server must be open to receive SSH connections. Then clikc on "Apply" and "Save"

<img width="794" alt="image" src="https://github.com/kalkah/Project-11-Ansible-Automate/assets/95209274/68448699-839f-4e33-9a57-1b583b589fb8">

### Prepare Development Enviroment

The first part of "DevOps" is "Dev", which means we will be required to write some codes and to make coding and debugging comfortable, we will need an Integrated development environment (IDE) or Source-code Editor. There is a plethora of different IDEs and Source-code Editors for different languages with their own advantages and drawbacks. We however decided to use one free and universal editor that will fully satisfy our needs – Visual Studio Code (VSC).

 VSC was opened and configured to connect to our newly created GitHub repository that we named ansible-config-mgt using the following steps:

On the left hand pane we click on "Source Control", click on the "Clone Repository" button and under the search bar, we click on "Clone from Github". This takes us to our browser and we get a prompt to log into our GitHub account so we click on "Sign in"

<img width="960" alt="image" src="https://github.com/kalkah/Project-11-Ansible-Automate/assets/95209274/f392e6a4-40de-4177-8bc2-973c325560b9">

<img width="947" alt="image" src="https://github.com/kalkah/Project-11-Ansible-Automate/assets/95209274/2cefb565-282f-4db9-9dfe-cd4a52c56441">

This prompts us to select a folder to use as the repository destination. So we create a folder for our repo and select it.
<img width="467" alt="image" src="https://github.com/kalkah/Project-11-Ansible-Automate/assets/95209274/37c07925-b1a7-4abe-b539-9beda7880746">

Then we get a dialogue box to open the cloned repository.
<img width="960" alt="image" src="https://github.com/kalkah/Project-11-Ansible-Automate/assets/95209274/ad540190-aa92-4bcf-bb50-aaf764e0f914">

As we can see in the image below, the ansible-config-mgt repository has been cloned to VS Code
<img width="959" alt="image" src="https://github.com/kalkah/Project-11-Ansible-Automate/assets/95209274/be0a8509-4a46-4900-b830-8e3adf3e4a71">

Next, we clone down the ansible-config-mgt repository to our Jenkins-Ansible instance with the following command:    `git clone <ansible-config-mgt repo link>`

`git clone https://github.com/kalkah/Ansible-config-mgt.git`
<img width="554" alt="image" src="https://github.com/kalkah/Project-11-Ansible-Automate/assets/95209274/234b3d0e-271b-4272-b65c-cdae5b830bff">

### Begin Ansible Development, Set up an Inventory and Create a Simple Ansible Playbook to Automate Server Configuration

**Step 1: Begin Ansible Development**
i. To begin Ansible development we go to our ansible-config-mgt GitHub repository in VS Code and we create a new branch new-feature that will be used for development of a new feature. We do this by following the steps below:

From the VS Code environment we go to the bottom of the page and we click on main. Next, under the dialogue box, we select "Create new branch from.." and we choose main. Then we subsequently click on the "Publish Branch" button. Afterwards, we enter the new branch name new-feature and we press Enter.
<img width="959" alt="image" src="https://github.com/kalkah/Project-11-Ansible-Automate/assets/95209274/70cee24d-ebf6-4fe2-b614-8a6a8c97b941">

ii. we enter the command below to checkout the newly created branch new-feature to our local machine and start building our code and directory structure:

`git checkout new-feature` 
<img width="721" alt="image" src="https://github.com/kalkah/Project-11-Ansible-Automate/assets/95209274/f2df6b95-6e83-404e-bc88-8da92497ad01">

iiI. We use the following command to create a directory that will be used to store all our playbooks files and we name it playbooks. `mkdir playbooks`

iv. We also create a directory that will be used to keep our hosts organised and we name this inventory.    `mkdir inventory`
<img width="722" alt="image" src="https://github.com/kalkah/Project-11-Ansible-Automate/assets/95209274/78374dd5-ca63-4981-98b8-1285152bd229">

v. Within the playbooks folder, we create our first playbook and name it common.yml: `touch common.yml`
<img width="722" alt="image" src="https://github.com/kalkah/Project-11-Ansible-Automate/assets/95209274/7d66dfa4-37a5-4917-b35a-7bda81d1ab46">

vi. And within the inventory folder, we create dev.yml, prod.yml, staging.yml and uat.yml for development, production, staging and user acceptance testing environments respectively. `cd inventory`    `touch dev.yml`    `touch prod.yml`    `touch staging.yml`    `touch uat.yml`
<img width="713" alt="image" src="https://github.com/kalkah/Project-11-Ansible-Automate/assets/95209274/7a8672cd-4294-4c1a-b22d-1104fd6d96c4">

**Step 2: Setting up Ansible Inventory**

An Ansible inventory file defines the hosts and groups of hosts upon which commands, modules, and tasks in a playbook operate. Since our intention is to execute Linux commands on remote hosts, and ensure that it is the intended configuration on a particular server that occurs, it is important to have a way to organize our hosts in such an inventory. We shall save the below inventory structure in the inventory/dev file to start configuring our development servers. We will also ensure to replace the IP addresses according to our own setup.

i. Ansible uses TCP port 22 by default, which means it needs to ssh into target servers from the Jenkins-Ansible host. For this, we implement the concept of ssh-agent. To install and configure Openssh server and agent for Windows 11, we follow these [instructions.](https://windowsloop.com/install-openssh-server-windows-11/).

ii. Next, we need to enable the ssh agent for the current session: `eval `ssh-agent -s``
<img width="727" alt="image" src="https://github.com/kalkah/Project-11-Ansible-Automate/assets/95209274/965d6358-31ae-4bb7-b294-fec9639cb4d4">

iii. Then we import our key into ssh-agent by executing the following command: `ssh-add <path-to-private-key>`  (ssh -add C:\Users\Kaamil\Downloads\server.pem)

iv. Afterwards, we confirm the key has been added with the command below:    `ssh-add -l`
<img width="729" alt="image" src="https://github.com/kalkah/Project-11-Ansible-Automate/assets/95209274/39435032-20fd-4379-b90e-ccacf18aa934">

v. Now we proceed to ssh into our Jenkins-Ansible server using the ssh-agent:    'ssh -A ubuntu@public-ip' (ssh -A ubuntu@13.49.9.69)
<img width="720" alt="image" src="https://github.com/kalkah/Project-11-Ansible-Automate/assets/95209274/4a12415e-043a-4743-bcf3-90685cdef6a7">

vi. Then we update our inventory/dev.yml file with the following lines of code:

```
[nfs]
<NFS-Server-Private-IP-Address> ansible_ssh_user=ec2-user

[webservers]
<Web-Server1-Private-IP-Address> ansible_ssh_user=ec2-user
<Web-Server2-Private-IP-Address> ansible_ssh_user=ec2-user

[db]
<Database-Private-IP-Address> ansible_ssh_user=ec2-user 

[lb]
<Load-Balancer-Private-IP-Address> ansible_ssh_user=ubuntu

```
<img width="710" alt="image" src="https://github.com/kalkah/Project-11-Ansible-Automate/assets/95209274/4c22890d-0e89-44ca-b8ed-a9b509ab47fe">


