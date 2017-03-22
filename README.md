Ansible Docker Development
===
Project uses docker, docker-compose, and ansible to create an ansible development
environment that provides a web and db tier. The provided ansible playbook creates a local administrator account as well as an application administrator account based on which tier the node belongs to. This is a proof of concept and should be used as a starting point. 

For more information on each of these tools check out the corresponding documentation:
 - [Docker](https://docs.docker.com/):   
   - Docker containers wrap up a piece of software in a complete filesystem that contains everything it needs to run: code, runtime, system tools, system libraries – anything you can install on a server. This guarantees that it will always run the same, regardless of the environment it is running in.
 - [Docker-Compose](https://docs.docker.com/compose/)  
   - Compose is a tool for defining and running multi-container Docker applications
 - [Ansible](http://docs.ansible.com/)
   - Ansible is an open-source automation engine that automates software provisioning, configuration management, and application deployment.  

# Quick Start  
```shell
git clone https://inarush0@gitlab.com/inarush0/ansible-docker-dev.git
cd ansible-docker-dev
docker-compose up -d
ansible-playbook playbooks/users.yml -k
```

# Docker Details
## Dockerfile
A Dockerfile is a text document that contains all the commands a user could call on the command line to assemble an image.
```Dockerfile
FROM centos:6
ENV container docker

# Update all base packages to keep them fresh
RUN yum -y update; yum clean all

# Install initscripts, but turn off all services by default
RUN yum -y install initscripts; yum clean all; rm /etc/rc.d/rc*.d/*

# Disable ttys
RUN mv /etc/init/serial.conf /etc/init/serial.conf.disabled; \
mv /etc/init/tty.conf /etc/init/tty.conf.disabled; \
mv /etc/init/start-ttys.conf /etc/init/start-ttys.conf.disabled

RUN yum -y install openssh-server openssh-clients
RUN sed -i 's/#PermitRootLogin no/PermitRootLogin yes/' /etc/ssh/sshd_config
#echo "GSSAPIAuthentication no" >> /etc/ssh/sshd_config
#sed -i "s/#UsePrivilegeSeparation.*/UsePrivilegeSeparation no/g" /etc/ssh/sshd_config && \
#sed -i "s/UsePAM.*/UsePAM no/g" /etc/ssh/sshd_config
RUN echo 'root:root' | chpasswd
RUN /etc/init.d/sshd start

EXPOSE 22

CMD ["/sbin/init"]
```  

## docker-compose.yml
The Docker Compose file defines the services that will be created from the docker image created from the Dockerfile.  
```yml
version: '2'
services:
  db:
    container_name: db01
    build: .
    networks:
      mynet:
        ipv4_address: 10.0.0.11
  web:
    container_name: web01
    build: .
    networks:
      mynet:
        ipv4_address: 10.0.0.21

networks:
  mynet:
    driver: bridge
    ipam: 
      config:
      - subnet: 10.0.0.0/24
```

# Ansible Details
## ansible.cfg
Ansible configuration is portable. The ansible.cfg file defines the configuration for this project.
```ini
[defaults]
inventory = hosts
host_key_checking = False

[paramiko_connection]
record_host_keys = False

[ssh_connection]
ssh_args = -o ControlMaster=auto -o ControlPersist=60s -o UserKnownHostsFile=/dev/null
``` 
## hosts
The hosts file defines the hosts and groups that ansible will operate on
```ini
[db]
db01 ansible_host=10.0.0.11

[web]
web01 ansible_host=10.0.0.21
```

## playbooks/users.yml
Playbooks define the instructions that ansible will use on each host. Tasks can be run on all hosts, or a specified subset.
```yaml
---
- hosts: all
  remote_user: root
  tasks:
    - name: Create Local Administrator
      user:
        name: ladmin
        comment: "Local Admin"
        uid: 1000
        password: $1$salt$R/cwcKyTA1TUFFnFfsgjD.

- hosts: web
  remote_user: root
  tasks:
    - name: Create Web Administrator
      user:
        name: webadmin
        comment: "Web Administrator"
        uid: 2000
        password: $1$salt$R/cwcKyTA1TUFFnFfsgjD.

- hosts: db
  remote_user: root
  tasks:
    - name: Create DB Administrator
      user:
        name: dbadmin
        comment: "DB Administrator"
        uid: 2001
        password: $1$salt$R/cwcKyTA1TUFFnFfsgjD.
```