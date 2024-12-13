
# Deploy Ansible and OpenStack-Ansible
**Overview**
1. **Setup Prerequtesites**
2. **Initialize SSH: Start the SSH service and verify access using passwords**.
3. **Setup Ansible with the correct inventory and SSH keys**
4. **Ditribute Ansible SSH passwordless publick Keys to related hosts using Initialize SSH**.
5. **Deploy OpenStack-Ansible: Proceed with OpenStack-Ansible installation and playbooks**.

This enhanced guide provides a structured approach to deploying OpenStack core services with OpenStack-Ansible on Ubuntu 22.04. It includes detailed deployment host preparation, installation, and testing instructions.

## Deployment Host Requirements

To ensure a successful OpenStack-Ansible deployment, your deployment host must meet specific hardware, software, and network requirements.

### Hardware Requirements

- Minimum **8 GB of RAM**
- At least **2 CPU cores**
- **50 GB** of available disk space

### Software Requirements

1. **Operating System**: Ubuntu 22.04 LTS
2. **Python 3.x** and **pip**: Install these on all nodes. You can verify installation using:
   ```bash
   sudo apt dist-upgrade
   sudo apt-get install bridge-utils debootstrap openssh-server tcpdump vlan python3 python3-pip yamllint
   python3 --version
   pip3 --version
   
   ```

### Preparation for Deployment Host

1. **Create an Administrative User and Password based SSH Keys foreach**:
   - Set up a user with sudo privileges for running OpenStack-Ansible:
   ```bash
   sudo adduser --disabled-password --gecos "" deployer
   sudo usermod -aG sudo deployer
   sudo passwd deployer
   sudo su - deployer
   ```
   - Configure password-based SSH for this user on all nodes.
   ssh-keygen -t ecdsa -b 521
   
   - Distributed public key to all nodes
   ssh-copy-id -i ~/.ssh/id_ecdsa.pub -f deployer@<node_ip>

2. **Disable Conflicting Services**:
   Ensure that services like **AppArmor** or **SELinux** are disabled to avoid interference with OpenStack services.

   ```bash
   sudo systemctl disable apparmor
   sudo systemctl stop apparmor
   ```

## 1. **Setup Networking for your physical hosts**

This section ensures network connectivity, hostname configuration, and internal DNS setup for your physical hosts.

### Setting Static IPs for your hosts

1. **Check Existing Netplan Configuration**:
   - List all Netplan YAML files:
     ```bash
     sudo ls /etc/netplan/
     ```

2. **Remove Existing Network Configurations**:
   - Move existing Netplan configuration files to prevent conflicts:
     ```bash
     sudo mv -f /etc/netplan/<file_name> /etc/netplan/backup_<file_name>
     ```

3. **Identify Active Network Interface**:
   - Check the active network interfaces to confirm the device name (inet or wifi):
     ```bash
     ip a
     ```

4. **Creat new network configurations**:
   - install openvswitch-switch for advanced network configurations
```bash
   sudo apt-get install openvswitch-switch
   sudo systemctl start systemd-networkd
```
 - Start systemd-networkd for advanced network configurations
```bash
   sudo systemctl start systemd-networkd
```
   - Creat a new Netplan file:
```bash
sudo nano /etc/netplan/01-netcfg-all.yaml
```
   - Example for `01-netcfg-all.yaml`:

```yaml
---
network:
  version: 2
  wifis:
    wlp4s0:  # Replace with your NIC name
      dhcp4: false
      addresses:
        - 192.168.1.200/24   # Replace with your static IP
      routes:
        - to: default
          via: 192.168.1.1   # Replace with your gateway IP
      nameservers:
        addresses: [8.8.8.8, 8.8.4.4]  # DNS servers
      access-points:
        "B408":       # Replace with your WiFi SSID
          password: "B408123456"   # Replace with your WiFi password

```
   - Save and close the file (Ctrl + S, then Enter to save, and Ctrl + X to exit).

5. **Apply the New Configuration**:
   - Apply the configuration to activate the new network settings:
     
```bash
# Set Permition for `01-netcfg-all.yaml` to only ownner can read and write 
sudo chmod 600 /etc/netplan/*
# Run netplan
sudo netplan apply
```

6. **Verify Connectivity**:
   - Test network connectivity with a simple ping command:
```bash
     ping 8.8.8.8
```

### Configure Internal DNS (Optional)

If you need a more scalable setup, configure an internal DNS server (e.g., Bind9 on Ubuntu) to manage name resolution for all nodes:
sudo apt update
sudo apt install bind9 bind9utils bind9-doc

3. **System Update**:
   Update and upgrade all packages on each node:
   ```bash
   sudo apt update
   ```

## 2. **Setup Asible**
### Setup Asible
```bash
sudo apt-get install python3-pip
pip3 install --upgrade pip
pip3 install netaddr jinja2

sudo apt install software-properties-common
sudo add-apt-repository --yes --update ppa:ansible/ansible
sudo apt install ansible
# checking version
sudo ansible --version
```
### Generate SSH key and distributed to related host for Openstack-Ansible (no passphrase)
1.**Gnerate the keys to connect to related host (controller, compute, storage, network, etc)**
```bash
ssh-keygen -t ecdsa -b 521 -f ~/.ssh/controller_key
ssh-keygen -t ecdsa -b 521 -f ~/.ssh/compute_key
ssh-keygen -t ecdsa -b 521 -f ~/.ssh/storage_key
ssh-keygen -t ecdsa -b 521 -f ~/.ssh/network_key
```
2.**Distribution to the related hosts uisng initile SSH conection**
- Prepare the .ssh Directory on the Target Hosts


```bash
ssh-copy-id -i ~/.ssh/controller_key.pub deployer@192.168.1.102
ssh-copy-id -i ~/.ssh/compute_key.pub deployer@192.168.1.102
ssh-copy-id -i ~/.ssh/storage_key.pub deployer@192.168.1.102
ssh-copy-id -i ~/.ssh/network_key.pub deployer@192.168.1.102
```

3.**Secure Save Private Keys**

```bash
sudo mkdir -p /etc/ansible/keys
sudo mv ~/.ssh/controller_key /etc/ansible/keys/controller_key
sudo mv ~/.ssh/compute_key /etc/ansible/keys/compute_key
sudo mv ~/.ssh/storage_key /etc/ansible/keys/storage_key
sudo mv ~/.ssh/network_key /etc/ansible/keys/network_key
sudo chmod 600 /etc/ansible/keys/*
sudo chown deployer:deployer /etc/ansible/keys/*
```

4. **Testing connections**
```bash
ssh -i /etc/ansible/keys/controller_key deployer@192.168.1.102
ssh -i /etc/ansible/keys/compute_key deployer@192.168.1.102
ssh -i /etc/ansible/keys/storage_key deployer@192.168.1.102
ssh -i /etc/ansible/keys/network_key deployer@192.168.1.102
```
### Setting Ansible Inventory for Openstack-Ansible

-**Defines how Ansible connects to other hosts to deploy OpenStack-Ansible.**

```bash
sudo nano /etc/ansible/hosts
```
- **Example for the `hosts` file**

```ini
[all]
controller ansible_host=192.168.1.102 ansible_user=deployer ansible_ssh_private_key_file=/etc/ansible/keys/controller_key
compute ansible_host=192.168.1.102 ansible_user=deployer ansible_ssh_private_key_file=/etc/ansible/keys/compute_key
storage ansible_host=192.168.1.102 ansible_user=deployer ansible_ssh_private_key_file=/etc/ansible/keys/storage_key
network ansible_host=192.168.1.102 ansible_user=deployer ansible_ssh_private_key_file=/etc/ansible/keys/network_key

# Groups
[controller]
controller

[compute]
compute

[storage]
storage

[network]
network

[nova_compute]
compute

```
-**Checking the setting**
```bash
ansible all -m ping
```

## 3: Install OpenStack-Ansible

### On the Deployment Host

1. **Clone the OpenStack-Ansible Repository**:

   ```bash
   sudo apt-get install git
   sudo git clone https://opendev.org/openstack/openstack-ansible /opt/openstack-ansible
   cd /opt/openstack-ansible
   ```

2. **Run the Bootstrap Script to setup Ansible**:

```bash
   sudo /opt/openstack-ansible/scripts/bootstrap-ansible.sh

```
Verify your results:
/opt/openstack-ansible
+ unset ANSIBLE_LIBRARY
+ unset ANSIBLE_LOOKUP_PLUGINS
+ unset ANSIBLE_FILTER_PLUGINS
+ unset ANSIBLE_ACTION_PLUGINS
+ unset ANSIBLE_CALLBACK_PLUGINS
+ unset ANSIBLE_CALLBACKS_ENABLED
+ unset ANSIBLE_TEST_PLUGINS
+ unset ANSIBLE_VARS_PLUGINS
+ unset ANSIBLE_STRATEGY_PLUGINS
+ unset ANSIBLE_CONFIG
+ unset ANSIBLE_COLLECTIONS_PATH
+ unset ANSIBLE_TRANSPORT
+ unset ANSIBLE_STRATEGY
+ echo 'System is bootstrapped and ready for use.'
System is bootstrapped and ready for use.

3. **Validation`**

```bash
ansible-config dump --only-changed
```


