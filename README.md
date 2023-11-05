# Project-11-Ansible-Automate
## Ansible Configuration Management

This project is 3-in-1 Project, further automation is built on the NFS server with installation of Jenkins, Ansible, and Load balancer.

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

