# Install AWX on a fresh CentOS 7 installation 

## Install EPEL repos

```
sudo yum install epel-release
```

## Install Ansible

```
sudo yum install ansible
```

### Check the Ansible version 

Make sure the version of Ansible is 2.7+
```
ansible --version

ansible 2.7.10
  config file = /etc/ansible/ansible.cfg
  configured module search path = [u'/home/parallels/.ansible/plugins/modules', u'/usr/share/ansible/plugins/modules']
  ansible python module location = /usr/lib/python2.7/site-packages/ansible
  executable location = /usr/bin/ansible
  python version = 2.7.5 (default, Apr  9 2019, 14:30:50) [GCC 4.8.5 20150623 (Red Hat 4.8.5-36)]
```

## Install required packages

The packages installation could be done using one single command
line, but I prefer specify each components independently. 

### Install Git

```
sudo yum install git
```

### Install Docker

```
sudo yum install docker
```

### Install Docker Compose

```
sudo yum install docker-compose
```

### Install Python Package Manager

```
sudo yum install python-pip
```

## Install required Python libraries

The following requirements may be different if you decide to install AWX on Kubernetes
or Openshift. This guide is for a standard installation (unmodified inventory file)
on a fresh CentOS. 

### Install Python Docker package

```
sudo pip install docker
```

### Install Python Docker-compose package

```
sudo pip install docker-compose
```

### Fix GSSException

Reference:
https://github.com/paramiko/paramiko/issues/1068
```
sudo pip uninstall gssapi 
```

### Set SELinux to Permissive

Reference:
https://github.com/ansible/awx/issues/202

Edit the file /etc/selinux/config 
```
# This file controls the state of SELinux on the system.
# SELINUX= can take one of these three values:
#     enforcing - SELinux security policy is enforced.
#     permissive - SELinux prints warnings instead of enforcing.
#     disabled - No SELinux policy is loaded.
SELINUX=permissive
# SELINUXTYPE= can take one of three two values:
#     targeted - Targeted processes are protected,
#     minimum - Modification of targeted policy. Only selected processes are protected.
#     mls - Multi Level Security protection.
SELINUXTYPE=targeted
```


## Install AWX

### Run Ansible installer

```
cd awx/installer

sudo ansible-playbook -i inventory install.yml
```

### Wait few minutes for processes to start

### Access the UI

```
http://localhost
```

Default username is "admin"
Default password is "password"


# AWX configuration

## Clone an Git repository using self signed certificate

#### Start a bash process in awx_task container

```
sudo docker exec -it awx_task bash
```

#### Edit the tower settings.py file

```
[root@awx awx]# vi /etc/tower/settings.py
```

#### Add GIT_SSL_NO_VERIFY parameter to AWX environment

```
AWX_TASK_ENV['GIT_SSL_NO_VERIFY'] = 'True'
```

#### Restart the Task container

Exit the bash session from container, then restart the container

```
sudo docker restart awx_task
```

