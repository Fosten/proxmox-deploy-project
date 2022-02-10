# Ansible Proxmox Deployment

This Ansible playbook utilizes [proxmoxer](https://github.com/proxmoxer/proxmoxer), [cloud-init](https://github.com/canonical/cloud-init), and [Qemu/KVM Virtual Machine Manager](https://pve.proxmox.com/wiki/Qemu-guest-agent) to deploy Virtual Machines.

## Install prereqs

### Setup the Proxmox Node for ansible communication

Login to your proxmox node and run the following commands.

```shell
apt install -y python3-pip build-essential
pip3 install --upgrade pip
pip3 install virtualenv
pip3 install proxmoxer
```

### Download the Debian cloud image and create a VM template with cloud-init

Adjust the VM IDs below to your preference.  You will need to ```su - root``` to set the environment for qm command.

```shell
wget https://cloud.debian.org/images/cloud/bullseye/latest/debian-11-generic-amd64.qcow2
qm create 400 --name "debian-2022-template" --memory 4096 --net0 virtio,bridge=vmbr0 --cores 2
qm importdisk 400 ~/debian-11-generic-amd64.qcow2 local-lvm
qm set 400 --scsihw virtio-scsi-pci --virtio0 local-lvm:vm-400-disk-0
qm set 400 --boot c --bootdisk virtio0
qm set 400 --ide2 local-lvm:cloudinit
qm set 400 --agent 1
qm template 400
```

### Clone template for Ansible Host

Replace the IP addresses below to match your desired environment.

```shell
qm clone 400 401 --name vm-ansible-host
qm set 401 --sshkey ~/.ssh/id_rsa.pub
qm set 401 --ipconfig0 ip=x.x.x.x/x,gw=x.x.x.x
qm resize 401 virtio0 16G
qm start 401
```

Due to [this bug](https://forum.proxmox.com/threads/kernel-panic-after-resizing-a-clone.93738/) Debian cloud-init images will get Kernel Panic upon first boot after using qm resize.  So the VM will need restarted:

```shell
qm stop 401
qm start 401
```

### Setup the Ansible Host

When using cloud-init, there is no password and no root login, but we can establish a connection using the SSH key we created in the last step.

From the proxmox node, ```ssh debian@x.x.x.x``` and do some basic setup, such as:

```shell
sudo adduser user
sudo sed -i 's/#\(PermitRootLogin*\).*/\1 yes/' /etc/ssh/sshd_config
sudo sed -i 's/\(PasswordAuthentication*\).*/\1 yes/' /etc/ssh/sshd_config
sudo service sshd restart
sudo passwd root
```

Then install some programs you'll need.

```shell
sudo apt-get install python3 python3-pip sshpass git -y
```

Change user, install ansible, and grab the repo

```shell
su user
pip3 install ansible
echo 'export PATH=/home/user/.local/bin:$PATH' >>~/.bash_profile
source ~/.bash_profile
git clone https://github.com/Fosten/proxmox-deploy-project.git
```

## Deploy a new VM

Now that everything is installed, to deploy more VMs in the future, you can just repeat the commands below.

### Edit the Hosts File

```shell
cd proxmox-deploy-project
nano hosts
```

Change the following (ssl_verify and ansible_ssh_extra_args can stay the same):

```shell
api_node= (Name of Proxmox Node)
api_user= (User with GUI access)
ansible_host= (IP of Proxmox Node)
ansible_user= (User with SSH access)
ansible_ssh_port= (Port to access SSH)
ansible_ssh_private_key_file= (Path to ssh key)
```

### Run the installer

```shell
make install
```
