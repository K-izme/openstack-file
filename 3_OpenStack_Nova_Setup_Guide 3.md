
# **OpenStack Nova Setup with OpenStack-Ansible**

# Prerequisites for OpenStack Compute (Nova) Deployment

## **System Requirements**

- **CPU Compatibility**:
  - Supported architectures: `x86_64`, `ppc64le` (only for Compute nodes).
  - At least one `repo_build` node per CPU type is required.
- **Storage**:
  - If using shared storage (e.g., NFS, Ceph), ensure UID and GID synchronization across hosts.
- **Networking**:
  - Adequate network bridges (management, tenant, provider).
  - Configure `multipath` for iSCSI-based storage, if needed.

## **BIOS Configuration**

- **For Intel Processors**: Enable VT-d/VT-x for virtualization features.
- **For AMD Processors**: Enable AMD-V/AMD-Vi in BIOS.

## **Setup Dependencies**

- Install `pip >= 7.1` on the target host.
```bash
pip3 install --upgrade pip
```
- RabbitMQ

```bash
sudo apt install rabbitmq-server -y
sudo systemctl enable rabbitmq-server
sudo systemctl start rabbitmq-server
sudo systemctl start rabbitmq-server
# Check status
sudo systemctl status rabbitmq-server

# Set Up Nova RabbitMQ User
sudo rabbitmqctl add_user nova <your secure_password>

# Set Permition for user
sudo rabbitmqctl set_permissions -p / nova ".*" ".*" ".*"

# Verify the users
sudo rabbitmqctl list_users
```
- Setup memcached
```bash
sudo apt install memcached
sudo systemctl start memcached
sudo systemctl enable memcached
sudo ufw allow 11211
```


## **Default Variables** (Optional Customization)

- Nova configurations can be customized in `/etc/openstack_deploy/user_variables.yml`:

  ```yaml
  nova_virt_type: "kvm"  # Default virtualization type
  nova_system_user_uid: 1001  # Sync UID for shared storage (checking: id <username>)
  nova_system_group_gid: 1001  # Sync GID for shared storage
  ```

## **Optional Features**

1. **Availability Zones**:
   - Define zones for specific hypervisor or hardware needs.
   - Example: Assign racks to different zones for fault tolerance.

2. **Config Drive**:
   - Force config drives for instance metadata if cloud-init is unavailable:

     ```yaml
     nova_nova_conf_overrides:
       DEFAULT:
         force_config_drive: True
     ```

3. **Ceph Integration**:
   - Use RBD for ephemeral storage by defining:

     ```yaml
     nova_libvirt_images_rbd_pool: "ephemeral-vms"
     ceph_mons:
       - "172.29.244.151"
       - "172.29.244.152"
     ```

4. **TLS for Live Migration**:
   - Enable secure live migrations by configuring libvirt TLS options.

---

## **Setup Overview**

Nova, OpenStack's compute service, is responsible for managing and deploying virtual machines (VMs). The setup involves six key steps:

1. **Revise `openstack_user_config.yml`**: Define the infrastructure, network topology, and resource allocation.
2. **Revise `user_variables.yml`**: Customizes OpenStack-specific configurations for Services (Nova) and Ansible role variables.
3. **Update Inventory, local DNS and other Configurations**: Revise inventory, Local DNS `/etc/hosts`, and network files to include Nova-related resources and local VM configurations.
4. **Define configurations specific to the Nova compute service**: Configure `group_vars/nova_all.yml`.
5. **Create a ```yaml file for the Nova setup**: Define tasks for setting up Nova in a playbook.
6. **Run the playbook and verify the setup**.
7. **Debug and troubleshooting**.

---

## **Step 1: Revise `openstack_user_config.yml` to Define the infrastructure, network topology, and resource allocation**

The `openstack_user_config.yml` file defines the infrastructure layout, network topology, and resource allocation.

### **Update the `openstack_user_config.yml`File to detemine where Ansible runs (OpenStack roles)**

```bash
sudo nano /etc/openstack_deploy/openstack_user_config.yml
```

-**Example Configuration (for one physical host)**

```yaml
---
control_plane:
  hosts:
    aio:  # name of the host
      ansible_host: 192.168.1.102  # The single host for the control plane
      ansible_user: deployer
  internal_lb_vip_address: 192.168.1.102  # No external load balancer required
compute_hosts:
  hosts:
    aio:
      ansible_host: 192.168.1.102  # Where Ansible run to config compute service
      ansible_user: deployer
storage_hosts:
  hosts:
    aio:
      ansible_host: 192.168.1.102  # On the same host to config storage services
      ansible_user: deployer
network_hosts:
  hosts:
    aio:
      ansible_host: 192.168.1.102  # On the same host to config network services
      ansible_user: deployer
```

-**Test Configuration (for syntax errors)**:

```bash
sudo apt install yamllint
yamllint /etc/openstack_deploy/openstack_user_config.yml
```

## **Step 2: Revise `user_variables.yml` to customizes deployment-specific settings and service variables**

The `user_variables.yml` file is used to override default Ansible role variables and define deployment-wide settings.

```bash
sudo nano /etc/openstack_deploy/user_variables.yml
```

-**Example Configuration for Nova**

```yaml
---
# Internal load balancer's virtual IP
internal_lb_vip_address: 192.168.1.102

# Keystone Service Configuration
keystone_service_internaluri_proto: http
keystone_service_port: 5000
keystone_service_internaluri: >-
  {{ keystone_service_internaluri_proto }}://
  {{ internal_lb_vip_address }}:{{ keystone_service_port }}
  
# Nova Service Configuration
nova_cpu_allocation_ratio: 10.0
nova_ram_allocation_ratio: 1.2
nova_virt_type: "kvm"
nova_enable_live_migration: true

# Nova acess control via UID, GUI (check: id username)
nova_system_user_uid: 1001  # Sync UID for shared storage
nova_system_group_gid: 1001  # Sync GID for shared storage

# Container networking (Inter-Domain Routing Configuration)
cidr_networks:
  container: "192.168.1.102/24"

```


-**Test Configuration (Validate the syntax):**

```bash
yamllint /etc/openstack_deploy/user_variables.yml
```

## **Step 3: Update Inventory, local DNS and other Configurations**

This step includes updating the inventory file `/etc/ansible/hosts`, local DNS `/etc/hosts` and network files.

1. **Gnerate SSH account (no passphrase) for authentication**

```bash
# checking: id username to name private key (my case is 1001)
ssh-keygen -t rsa -b 3072

# Copy to target host and set permitsion
ssh-copy-id -i 1001_rsa.pub -f deployer@192.168.1.102
sudo mkdir -p /etc/ansible/keys
sudo mv ~/1001_rsa /etc/ansible/keys/
sudo chmod 600 /etc/ansible/keys/1001_rsa
sudo chown root:root /etc/ansible/keys/1001_rsa

# Verify conection
sudo ssh -i /etc/ansible/keys/1001_rsa deployer@192.168.1.102

```

2. **Update the Inventory File to defines all hosts managed by Ansible**

```bash
sudo nano /etc/ansible/hosts
```

-**Example Inventory (only one physical host for all Openstack services)**

```ini
[all]
aio ansible_host=192.168.1.102 ansible_user=deployer ansible_ssh_private_key_file=/etc/ansible/keys/1001_rsa

[control_plane]
aio

[compute]
aio

[storage]
aio

[network]
aio

[nova_compute]
aio 
```
--**Verify the setup**

```bash
sudo ansible-inventory --list -i /etc/ansible/hosts
```

-**Example Inventory (more than one physical hosts for Openstack services)**
```ini
[all]
control1 ansible_host=192.168.1.101 ansible_user=deployer ansible_ssh_pass=your_password
compute1 ansible_host=192.168.1.102 ansible_user=deployer ansible_ssh_pass=your_password
storage1 ansible_host=192.168.1.103 ansible_user=deployer ansible_ssh_pass=your_password
network1 ansible_host=192.168.1.104 ansible_user=deployer ansible_ssh_pass=your_password

[control_plane]
control1

[compute]
compute1

[storage]
storage1

[network]
network1
```

3. **Update local DNS file `/etc/hosts`**

```bash
sudo nano /etc/hosts
```

-**Example addding line for /etc/hosts**

```plaintext
192.168.1.102 aio all control_plane compute storage network

```
- **Test Connectivity (Verify DNS resolution and connectivity):**

```bash
ping aio
ping controller
sudo ansible all -m ping -i /etc/ansible/hosts
```
Checking the return should be "aio | SUCCESS ..."

## **Step 4: Create Server Digital Certificates
# Step 4: Create Server Digital Certificates

This step includes creating digital certificates for Nova to authenticate and secure communication, and updating the necessary Ansible YAML files.

---

## 1. Generate Digital Certificates using ECC Ed25519

1. **Create a Directory for Certificates:**
   ```bash
   sudo mkdir -p /etc/openstack/certs
   sudo chmod 700 /etc/openstack/certs
   ```

2. **Generate a Private Key:**
   ```bash
   sudo openssl genpkey -algorithm Ed25519 -out /etc/openstack/certs/nova.key
   ```

3. **Generate a Certificate Signing Request (CSR):**
   ```bash
   sudo openssl req -new -key /etc/openstack/certs/nova.key -out /etc/openstack/certs/nova.csr -subj "/C=VN/ST=HCM/L=ThuDuc/O=UIT/OU=MMT/CN=nova.local"
   ```

4. **Self-Sign the Certificate (or use a CA):**
   ```bash
   sudo openssl x509 -req -in /etc/openstack/certs/nova.csr -signkey /etc/openstack/certs/nova.key -out /etc/openstack/certs/nova.crt -days 365

   ```
   - Replace the self-signed certificate with a CA-signed certificate if needed for production environments.

5. **Set Proper Permissions:**
   ```bash
	sudo chmod 600 /etc/openstack/certs/nova.key
	sudo chmod 644 /etc/openstack/certs/nova.crt
   ```
## 2. Generate SSH Signing Key

```bash
sudo ssh-keygen -t rsa -b 4096 -f /etc/openstack_deploy/ssh_signing_key -N ""
# Verify (should return two key)
sudo ls /etc/openstack_deploy/
# Set Permistion
sudo chmod 600 /etc/openstack_deploy/ssh_signing_key
sudo chmod 644 /etc/openstack_deploy/ssh_signing_key.pub
sudo chown root:root /etc/openstack_deploy/ssh_signing_key*
```
- Revise `create_keypair.yml`
```bash
sudo yamllint /etc/ansible/ansible_collections/openstack/osa/roles/ssh_keypairs/tasks/standalone/create_keypair.yml
- Revise the line: 
_ca_file: "{{ ssh_keypairs_dir ~ '/' ~ kp.cert.signed_by }}"
to 
_ca_file: "{{ kp.cert.signed_by }}"
```

## **Step 5: Define Configurations for Nova in group_vars/nova_all.yml**

The `group_vars/nova_all.yml` file contains configurations specific to the Nova compute service.

- **Edit Configuration File**/

```bash
sudo mkdir -p /etc/ansible/group_vars
sudo nano /etc/ansible/group_vars/nova_all.yml
```

-**Example Configuration**

```yaml
# SSH signing key configuration
openstack_ssh_signing_key: "/etc/openstack_deploy/ssh_signing_key"
nova_ssh_key_principals: "nova"
nova_ssh_key_valid_from: "always"
nova_ssh_key_valid_to: "forever"

# Keypair directory
ssh_keypairs_dir: "/etc/openstack_deploy/ssh_keypairs"

# Password setting
nova_container_mysql_password: "your-secure-password"
nova_api_database_password: "your-secure-api-password"
nova_cell0_database_password: "your-secure-cell0-password"
nova_cell1_database_password: "your-secure-cell1-password"
nova_metadata_database_password: "your-secure-metadata-password"
nova_api_container_mysql_password: "your-secure-metadata-password"

# PKI settings
pki_dir: "/etc/pki"
openstack_pki_dir: "{{ pki_dir }}/openstack"
nova_pki_dir: "/etc/openstack/certs"  # Path to existing certificates
nova_pki_generate_certificates: false
nova_pki_certificates: []  # Empty list for API/console certificates
nova_pki_console_certificates: []  # Empty list for console certificates
nova_pki_compute_certificates: []  # Empty list for compute certificates
nova_pki_console_install_certificates: []
nova_pki_compute_install_certificates: []
nova_pki_regen_cert: false  # Prevent certificate regeneration
nova_user_ssl_cert: "/etc/openstack/certs/nova.crt"  # Pre-existing cert path
nova_user_ssl_key: "/etc/openstack/certs/nova.key"  # Pre-existing key path

# Nova-specific TLS settings for existing certificates
nova_tls:
  enabled: true
  key_file: "/etc/openstack/certs/nova.key"  # Path to existing private key
  cert_file: "/etc/openstack/certs/nova.crt"  # Path to existing certificate

# Network and Virtualization settings
nova_network_type: "vxlan"
nova_virt_type: "kvm"

# Resource allocation ratios
nova_cpu_allocation_ratio: 10.0
nova_ram_allocation_ratio: 1.2

# OpenStack Messaging (RabbitMQ) credentials
nova_oslomsg_rpc_user: "nova"
nova_oslomsg_rpc_password: "29112001Tu"

# Nova-specific service credentials
nova_service_user: "nova"
nova_service_password: "29112001Tu"

# Scheduler configuration
nova_scheduler_default_filters:
  - "RetryFilter"
  - "AvailabilityZoneFilter"
  - "RamFilter"
  - "DiskFilter"
  - "ComputeFilter"

# Memcached servers configuration
memcached_servers:
  - "127.0.0.1:11211"
memcached_encryption_key: "YourSecureEncryptionKeyHere"
external_lb_vip_address: "192.168.1.102"

# Keystone configuration
keystone_service_adminurl: "http://192.168.1.102:5000/v3"
keystone_service_internalurl: "http://192.168.1.102:5000/v3"
keystone_service_publicurl: "http://192.168.1.102:5000/v3"
keystone_service_adminuri: "http://192.168.1.102:5000/v3"
keystone_service_internaluri: "http://192.168.1.102:5000/v3"
keystone_service_publicuri: "http://192.168.1.102:5000/v3"

# Insecure settings (adjust based on deployment security)
keystone_service_adminuri_insecure: false
keystone_service_internaluri_insecure: false
keystone_service_publicuri_insecure: false

# Keystone service region
keystone_service_region: "RegionOne"  # Specify the region name

# Nova metadata proxy configuration
nova_metadata_proxy_secret: "29112001TuProxy"
```

**- Test Configuration (Check the syntax)**:

```bash
yamllint /etc/ansible/group_vars/nova_all.yml
```

## **Step6: Create a `yaml File for Nova Setup` to Install Nova packages, Configure Nova and Start the Nova services.**

- **Create a playbook for deploying Nova at `/opt/openstack-ansible/playbooks/nova.yml`**

```bash
sudo nano /opt/openstack-ansible/playbooks/nova.yml
```

- **Example Playbook**

```yaml
---
# Nova Deployment Playbook
# This playbook configures and deploys the Nova service.
- name: Deploy Nova
  hosts: aio
  gather_facts: true
  vars:
    pki_enabled: true  # Enable PKI for this deployment
  become: true  # Apply privilege escalation globally for all tasks
  tasks:
    # Debug section
    - name: Debug all loaded variables
      debug:
        var: hostvars[inventory_hostname]
      tags:
        - debug

    - name: Debug nova_oslomsg_rpc_user variable
      debug:
        var: nova_oslomsg_rpc_user
      tags:
        - debug

    - name: Debug nova_service_user variable
      debug:
        var: nova_service_user
      tags:
        - debug

    # Ensure rootwrap.d directory exists
    - name: Ensure /etc/nova/rootwrap.d exists
      file:
        path: /etc/nova/rootwrap.d
        state: directory
        owner: root
        group: root
        mode: '0755'

    # Copy nova rootwrap filter config
    - name: Copy nova rootwrap filter config
      copy:
        src: "/etc/nova/rootwrap.d/compute.filters"
        dest: "/etc/nova/rootwrap.d/compute.filters"
        owner: root
        group: root
        mode: '0644'
      when: >-
        'compute.filters' in
        lookup('fileglob', '/etc/nova/rootwrap.d/*', errors='ignore')
        
    # Package installation
    - name: Ensure nova packages are installed
      include_role:
        name: "os_nova"
      tags:
        - packages

    # Nova configuration
    - name: Configure Nova
      include_role:
        name: "os_nova"
      vars:
	nova_pki_generate_certificates: false  # Disable certificate generation
        nova_tls_key_file: "/etc/openstack/certs/nova.key"  # Private key
        nova_tls_cert_file: "/etc/openstack/certs/nova.crt"  # Certificate
        nova_tls_ca_file: null  # No CA file as the certificate is self-signed
      tags:
        - configure

    # Start Nova services
    - name: Start Nova services
      include_role:
        name: "os_nova"
      tags:
        - start
```

-Validate Playbook (syntax check):

```bash
yamllint /opt/openstack-ansible/playbooks/nova.yml

sudo ansible-playbook -i /etc/ansible/hosts  /opt/openstack-ansible/playbooks/nova.yml --syntax-check
```

- The result should be:

playbook: /opt/openstack-ansible/playbooks/nova.yml

EXIT NOTICE [Playbook execution success] **************************************
===============================================================================


## **Step 7: Setup SSH accounts and run Playbook to install Nova and Verify the Setup**

## **Step 7.1 : Generate SSH account

### Generate SSH host keys

```bash
sudo ssh-keygen -t rsa -b 4096 -f /etc/ssh/ssh_host_rsa_key -N ""
sudo ssh-keygen -t ed25519 -f /etc/ssh/ssh_host_ed25519_key -N ""
sudo chmod 600 /etc/ssh/ssh_host_*
sudo chown root:root /etc/ssh/ssh_host_*
```
### Revise `/etc/ssh/sshd_config` for SSH key-based authentication

```bash
sudo nano /etc/ssh/sshd_config
```
### Enable these lines:

```ini
HostKey /etc/ssh/ssh_host_ed25519_key
HostKey /etc/ssh/ssh_host_rsa_key
PermitRootLogin prohibit-password
PubkeyAuthentication yes
AuthorizedKeysFile      .ssh/authorized_keys
PasswordAuthentication yes  # This allows password fallback
```

#### Restart `sshd`

```bash
sudo systemctl restart sshd
```

# Test SSH conection

```bash
ssh-keygen -f "/home/deployer/.ssh/known_hosts" -R "192.168.1.102"
ssh -i /etc/ansible/keys/1001_rsa deployer@192.168.1.102
```

## **Step 7.2 : Run ansible-playbook to deploy Nova


### Verify dependeces

```bash
sudo ansible-galaxy collection list
```

### Revise `ansible.cfg`

```bash
sudo nano /etc/ansible/ansible.cfg
```
### Example `ansible.cfg`

```ini
[defaults]
inventory = /etc/ansible/hosts
roles_path = /etc/ansible/roles:/usr/share/ansible/roles
collections_paths = /etc/ansible/ansible_collections:/usr/share/ansible/collections:~/.ansible/collections
host_key_checking = False
remote_user = deployer
```

### Disable Certificate Creation Tasks

```bash
sudo nano /etc/ansible/roles/os_nova/tasks/main.yml
```
Replace old block:
```ini
# Create certs after nova groups have been created but before handlers
- name: Create and install SSL certificates for API and Consoles
  include_role:
    name: pki
    tasks_from: main_certs.yml
    apply:
      tags:
        - nova-config
        - pki
  vars:
    pki_setup_host: "{{ nova_pki_setup_host }}"
    pki_dir: "{{ nova_pki_dir }}"
    pki_create_certificates: "{{ nova_user_ssl_cert is not defined and nova_user_ssl_key is not defined }}"
    pki_regen_cert: "{{ nova_pki_regen_cert }}"
    pki_certificates: "{{ nova_pki_certificates + nova_pki_console_certificates }}"
    pki_install_certificates: "{{ nova_pki_install_certificates + nova_pki_console_install_certificates }}"
  when:
    - nova_pki_certificates_condition | bool or nova_pki_console_condition | bool
  tags:
    - always

- name: Create and install SSL certificates for compute hosts
  include_role:
    name: pki
    tasks_from: main_certs.yml
    apply:
      tags:
        - nova-config
        - pki
  vars:
    pki_setup_host: "{{ nova_pki_setup_host }}"
    pki_dir: "{{ nova_pki_dir }}"
    pki_create_certificates: "{{ nova_user_ssl_cert is not defined and nova_user_ssl_key is not defined }}"
    pki_regen_cert: "{{ nova_pki_regen_cert }}"
    pki_certificates: "{{ nova_pki_compute_certificates }}"
    pki_install_certificates: "{{ nova_pki_compute_install_certificates }}"
  when:
    - nova_libvirtd_listen_tls == 1
    - "'nova_compute' in group_names"
    - nova_virt_type != 'ironic'
  tags:
    - always
```
To the new one
```ini
# Create certs after nova groups have been created but before handlers
- name: Create and install SSL certificates for API and Consoles
  include_role:
    name: pki
    tasks_from: main_certs.yml
    apply:
      tags:
        - nova-config
        - pki
  vars:
    pki_setup_host: "{{ nova_pki_setup_host }}"
    pki_dir: "{{ nova_pki_dir }}"
    pki_create_certificates: false  # Disable certificate generation
    pki_regen_cert: false
    pki_certificates: "{{ nova_pki_certificates + nova_pki_console_certificates }}"
    pki_install_certificates: "{{ nova_pki_install_certificates + nova_pki_console_install_certificates }}"
  when:
    - nova_pki_certificates_condition | bool or nova_pki_console_condition | bool
  tags:
    - always

- name: Create and install SSL certificates for compute hosts
  include_role:
    name: pki
    tasks_from: main_certs.yml
    apply:
      tags:
        - nova-config
        - pki
  vars:
    pki_setup_host: "{{ nova_pki_setup_host }}"
    pki_dir: "{{ nova_pki_dir }}"
    pki_create_certificates: false  # Disable certificate generation
    pki_regen_cert: false
    pki_certificates: "{{ nova_pki_compute_certificates }}"
    pki_install_certificates: "{{ nova_pki_compute_install_certificates }}"
  when:
    - nova_libvirtd_listen_tls == 1
    - "'nova_compute' in group_names"
    - nova_virt_type != 'ironic'
    - nova_pki_compute_certificates | length > 0  # Ensure list is defined
  tags:
    - always
```


### Run ansible-playbook to setup Nova

```bash
sudo chmod 644 /etc/nova/rootwrap.d/compute.filters

sudo ansible-playbook -i /etc/ansible/hosts \
  /opt/openstack-ansible/playbooks/nova.yml \
  -e "@/etc/ansible/group_vars/nova_all.yml" \
  --ask-become-pass

```

- **Verify the status of Nova services**

```bash
sudo openstack compute service list
```

## **Step 7.3: Create a test Virtual Machine using Nova**

```bash
sudo openstack server create --flavor m1.small --image cirros --network private my-server
```

- **Verify the status of the test Virtual Machine instance**:

```bash
sudo apt update
sudo apt install python3-openstackclient -y
sudo nano /etc/openstack/openrc
# add these lines to openrc
```
export OS_AUTH_URL=http://192.168.1.100:5000/v3  # Replace 192.168.1.100 with your OpenStack controller's IP
export OS_PROJECT_ID=1234567890abcdef1234567890abcdef  # Replace with your actual project ID
export OS_PROJECT_NAME=admin  # Replace with your project name (e.g., 'admin' or 'demo')
export OS_USER_DOMAIN_NAME=Default
export OS_PROJECT_DOMAIN_NAME=Default
export OS_USERNAME=admin  # Replace with your OpenStack admin username
export OS_PASSWORD=your_password  # Replace with your OpenStack admin password
export OS_REGION_NAME=RegionOne
export OS_INTERFACE=public
export OS_IDENTITY_API_VERSION=3

```
# source the file
source /etc/openstack/openrc
# Check
openstack --version
openstack server list
```

## **Step 7: Debugging (If Necessary)**

- Logs: Check /var/log/nova/ for errors.

- Run Verbose Mode: Re-run the playbook with increased verbosity:

```bash
ansible-playbook -i /etc/ansible/hosts /opt/openstack-ansible/playbooks/nova.yml -vvv
```

**Summary**
This guide provides a structured process for setting up Nova, ensuring consistency and proper configurations across all related files. Adjust configurations as needed to match your specific environment.

**References**

<https://opendev.org/openstack/openstack-ansible-os_Nova>
<https://docs.openstack.org/openstack-ansible-os_nova/latest/>
