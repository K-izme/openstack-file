
# Deploy Keystone Before Nova

## 1. Prerequisites

### 1.1 Set Up the Database for Keystone
```bash
sudo mysql -u root -p
```
```sql
CREATE DATABASE keystone;
GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'localhost' IDENTIFIED BY 'password';
GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'%' IDENTIFIED BY 'password';
FLUSH PRIVILEGES;
EXIT;
```

### 1.2 Install Required Packages
```bash
sudo apt update
sudo apt install keystone python3-openstackclient apache2 libapache2-mod-wsgi-py3 -y
```

---

## 2. Configure Keystone

### 2.1 Configure Keystone Database
1. Open the Keystone configuration file:
   ```bash
   sudo nano /etc/keystone/keystone.conf
   ```

2. Update the following sections:

   **[database]**
   ```ini
   [database]
   connection = mysql+pymysql://keystone:password@localhost/keystone
   ```

   **[token]**
   ```ini
   [token]
   provider = fernet
   ```

3. Save and exit.

4. Sync the Keystone database:
   ```bash
   sudo keystone-manage db_sync
   ```

---

### 2.2 Configure Fernet Keys
1. Initialize Fernet keys:
   ```bash
   sudo keystone-manage fernet_setup --keystone-user keystone --keystone-group keystone
   sudo keystone-manage credential_setup --keystone-user keystone --keystone-group keystone
   ```

2. Bootstrap Keystone:
   ```bash
   sudo keystone-manage bootstrap \
     --bootstrap-password <admin_password> \
     --bootstrap-admin-url http://<controller_ip>:5000/v3/ \
     --bootstrap-internal-url http://<controller_ip>:5000/v3/ \
     --bootstrap-public-url http://<controller_ip>:5000/v3/ \
     --bootstrap-region-id RegionOne
   ```

---

## 3. Configure Apache for Keystone

### 3.1 Enable WSGI Modules
```bash
sudo a2enmod wsgi
```

### 3.2 Create Keystone WSGI Configuration
1. Create a new file:
   ```bash
   sudo nano /etc/apache2/sites-available/keystone.conf
   ```

2. Add the following content:
   ```apache
   Listen 5000

   <VirtualHost *:5000>
       ServerName controller
       WSGIDaemonProcess keystone group=keystone processes=5 threads=1 user=keystone display-name=%{GROUP}
       WSGIProcessGroup keystone
       WSGIScriptAlias / /usr/bin/keystone-wsgi-public
       WSGIApplicationGroup %{GLOBAL}
       <Directory /usr/bin>
           Require all granted
       </Directory>
   </VirtualHost>
   ```

3. Save and exit.

4. Enable the Keystone site and restart Apache:
   ```bash
   sudo a2ensite keystone
   sudo systemctl restart apache2
   ```

---

## 4. Verify Keystone Deployment

### 4.1 Test Keystone API Locally
```bash
curl -i http://localhost:5000/v3/
```

### 4.2 Test Keystone API Remotely
```bash
curl -i http://<controller_ip>:5000/v3/
```

---

## 5. Set Up the OpenRC File

### 5.1 Create OpenRC File
```bash
sudo nano /etc/openstack/openrc
```

Add the following content:
```bash
export OS_AUTH_URL=http://<controller_ip>:5000/v3
export OS_PROJECT_NAME=admin
export OS_USERNAME=admin
export OS_PASSWORD=<admin_password>
export OS_REGION_NAME=RegionOne
export OS_INTERFACE=public
export OS_IDENTITY_API_VERSION=3
```

### 5.2 Source the File
```bash
source /etc/openstack/openrc
```

### 5.3 Test OpenStack CLI
```bash
openstack token issue
```

---

## 6. Proceed with Nova Deployment

Once Keystone is deployed and operational, revisit the Nova playbook or deployment steps and ensure Nova services are correctly registered in Keystone.

---
