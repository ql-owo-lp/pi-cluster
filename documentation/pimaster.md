# Preparing Ansible Control Node

A Virtual-BOX VM running on a Windows PC is used as Ansible Control Node for automating the provisioning of the Raspberry PIs cluster.

This server, **pimaster**, is an Ubuntu 20.04 VM that can be automatically created from ubuntu cloud image using the script in this [repository] (https://github.com/ricsanfre/ubuntu-cloud-vbox).

## Installing Docker

Docker is used by Molecule, Ansible's testing tool, for building the testing environment, so it is needed to have a Docker installation on the Control Node for developing and testing the Ansible Playbooks/Roles.

Follow official [installation guide](https://docs.docker.com/engine/install/ubuntu/).

Step 1. Uninstall old versions of docker

    sudo apt-get remove docker docker-engine docker.io containerd runc

Step 2. Install packages to allow apt to use a repository over HTTPS

```
sudo apt-get update

sudo apt-get install \
  apt-transport-https \
  ca-certificates \
  curl \
  gnupg \
  lsb-release
```
	
	
Step 3. Add docker´s official GPG	key
	
	curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
	

Step 4: Add x86_64 repository	

```
echo \
  "deb [arch=amd64 signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

Step 5: Install Docker Engine


    sudo apt-get install docker-ce docker-ce-cli containerd.io
	

Step 6: Enable docker management with non-priviledge user

- Create docker group

    ```
    sudo groupadd docker
    ```
    
- Add user to docker group

    ```
    sudo usermod -aG docker $USER
    ```
    
Step 7: Configure Docker to start on boot

    sudo systemctl enable docker.service
    sudo systemctl enable containerd.service

Step 8: Configure docker daemon.

- Edit file `/etc/docker/daemon.json´
	
   Set storage driver to overlay2 and to use systemd for the management of the container’s cgroups.
   Optionally default directory for storing images/containers can be changed to a different disk partition (example /data).
   Documentation about the possible options can be found [here](https://docs.docker.com/engine/reference/commandline/dockerd/#daemon-configuration-file)
	
	```json
	{
      "exec-opts": ["native.cgroupdriver=systemd"],
      "log-driver": "json-file",
      "log-opts": {
        "max-size": "100m"
       },
      "storage-driver": "overlay2",
	  "data-root": "/data/docker"
	  
    }
    ```	
- Restart docker

    ```
    sudo systemctl restart docker
    ```
    
## Installing Ansible, playboook syntax checking tools and Molecule (Ansible testing environment)

Ansible can be installed in Ubuntu 18.04/20.04 using official package from the ansible repository 'sudo apt install ansible' will install an old ansible verion.

Ansible Molecule is not available as official package, so pip is the only alternative
Instead, install latest version for python3 with python package manager pip.

Python Ansible and Molecule packages and its dependencies installed using Pip might conflict with python3 packages included in the Ubuntu official release.

Since some of those python3 packages installed with apt(like PyYaml) cannot be updated without impacting some of Ubuntu core utils, a python virtual environment for Ansible will be used instead.

Installation of the whole Ansible environment can be done using a python virtual environment.

Step 1. Install python Virtual Env and Pip3 package

    sudo apt-get install python3-venv python3-pip

Step 2. Create Virtual Env for Ansible

    python3 -m venv ansible_env

Step 3. Activate Virtual Environment

    source ansible_env/bin/activate
	
> NOTE: For deactivating the Virtual environment

	deactivate

Step 4. Upgrade setuptools and pip packages

    pip3 install --upgrade pip setuptools
	
Step 5. Install ansible

    pip3 install ansible

Step 6. Install yamllint and ansible-lint

    pip3 install yamllint ansible-lint

Step 7. Install Docker python driver and molecule packages:

    pip3 install molecule[docker]

## Create public/private SSH key for the ansible and remote connection users

`ansible` unix user will be created in all servers with root privileges (sudo permissions) so Ansible can automate the configuration process (use as `ansible_remote_user` when connecting).

For remote connection from my Windows laptop `ubuntu` default UNIX user (with sudo privileges)

ssh private/public keys for both users need to be generated once, and public ssh key can be copied automatically on all servers of the cluster to enable passwordless SSH connection.
Those users and its public keys will be added to cloud-init configuration (`user-data`), when installing Ubuntu OS.

### Create SSH keys

Authentication using SSH keys will be the only mechanism available to login to the server.
We will create SSH keys for two different users:

- **ubuntu** user, used to connect from my home laptop

    For generating SSH private/public key in Windows, Putty Key Generator can be used:

    ![ubuntu-SSH-key-generation](images/ubuntu-user-SSH-key-generation.png "SSH Key Generation")

Public-key string will be used as ssh_authorized_keys of the default user (ubuntu) in cloud-init `user-data`

- **ansible** user, used to automate configuration activities with ansible
 
     For generating ansible SSH keys in Ubuntu server execute command:

        ssh-keygen

    In directory `$HOME/.ssh/` public and private key files can be found for the user

    `id_rsa` contains the private key and `id_rsa.pub` contains the public key.

    Content of the id_rsa.pub file has to be used as ssh_authorized_keys of the ansible user in cloud-init `user-data`
    ```
    cat id_rsa.pub 
    ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQDsVSvxBitgaOiqeX4foCfhIe4yZj+OOaWP+wFuoUOBCZMWQ3cW188nSyXhXKfwYK50oo44O6UVEb2GZiU9bLOoy1fjfiGMOnmp3AUVG+e6Vh5aXOeLCEKKxV3I8LjMXr4ack6vtOqOVFBGFSN0ThaRTZwKpoxQ+pEzh+Q4cMJTXBHXYH0eP7WEuQlPIM/hmhGa4kIw/A92Rm0ZlF2H6L2QzxdLV/2LmnLAkt9C+6tH62hepcMCIQFPvHVUqj93hpmNm9MQI4hM7uK5qyH8wGi3nmPuX311km3hkd5O6XT5KNZq9Nk1HTC2GHqYzwha/cAka5pRUfZmWkJrEuV3sNAl ansible@pimaster
    ```


## Installing KVM and Vagrant

In order to automate the testing of some of the roles that requires a VM and not a docker image (example: Storage roles), KVM and Vagrant will be installed

### Enable nested virtualization within the VM

Need to be changed with the command line. Not supported in GUI

    vboxmanage modifyvm <pimaster-VM> --nested-hw-virt on


### KVM installation in Ubuntu 20.04

Step 2. Install KVM packages and its dependencies

    sudo apt install qemu qemu-kvm libvirt-clients libvirt-daemon-system virtinst bridge-utils

Step 3. Enable on boot and start libvirtd service (If it is not enabled already):

    sudo systemctl enable libvirtd
    sudo systemctl start libvirtd

Step 4. Add the user to libvirt group

    sudo usermod -a -G libvirtd $USER 

### Vagrant installation in Ubuntu 20.04

Step 1.  Add hashicorp apt repository

    curl -fsSL https://apt.releases.hashicorp.com/gpg | sudo apt-key add -
    sudo apt-add-repository "deb [arch=amd64] https://apt.releases.hashicorp.com $(lsb_release -cs) main"
    sudo apt-get update
Step 2. Install vagrant
   
    sudo apt install vagrant

### Install vagrant-libvirt plugin in Linux

In order to run Vagrant virtual machines on KVM, you need to install the vagrant-libvirt plugin. This plugin adds the Libvirt provider to Vagrant and allows Vagrant to control and provision machines via Libvirt

Step 1. Install dependencies

    sudo apt install build-essential qemu libvirt-daemon-system libvirt-clients libxslt-dev libxml2-dev libvirt-dev zlib1g-dev ruby-dev ruby-libvirt ebtables dnsmasq-base libguestfs-tools

Step 2. Install vagrant-libvirt plugin:

    vagrant plugin install vagrant-libvirt

Step 3. Install mutate plugin which converts vagrant boxes to work with different providers.

    vagrant plugin install vagrant-mutate

### Install molecule vagrant driver

   pip3 install vagrant-python molecule-python molecule-vagrant