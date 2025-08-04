# End-to-End Monitoring Setup through Ansible

- This setup only works for aws servers.

- This repository contains ansible roles to setup monitoring component like prometheus, grafana, alertmanger and deploy the exporters like node, blackbox, process, mysql, mongo, postgres, redis, elasticsearch, kafka.

- This will update prometheus config with the required job/target and rule file of the selected exporters.

- Have the integration with slack, email and squadcast in the alertmanager configuration.

- Deploy the dashboard of the selected exporter in the grafana.


## Initial setup

### Setup the Ansible controller server.
#### Note: python >= 3.6 should be install in the server.

#### 1. Spin-up instance in the cloud, to deploy the ansible controller.

#### 2. Install ansible and related packages:
```sh
sudo apt update
sudo apt install -y software-properties-common python3-pip
sudo add-apt-repository --yes --update ppa:ansible/ansible
sudo apt install -y ansible yamllint
# verify it
ansible --version
sudo pip3 install boto3 netaddr jmespath requests google-auth
# execute the following command only if error encounter while using above command.
sudo pip3 install boto3 netaddr jmespath requests google-auth --break-system-packages
``` 

#### 3. Install ansible modules/collection.
```sh
ansible-galaxy collection install amazon.aws
ansible-galaxy collection install azure.azcollection
ansible-galaxy collection install google.cloud
```

### Clone this repo in the Ansible controller server.
#### Note: Fetch the latest code from the repo.
```sh
cd /opt
git clone https://gitlab.techpartner.in/automation/monitoring.git
# enter your login credentials
git checkout task-id-86cycqywg/update-ansible-roles
# now, follow all the pre-requisite and requirement section. 
```

#### Go through the README properly.

## Pre-requiste: 
Before executing the ansible playbook make sure the following thing should be correct.

### 1. SSH access to all the provision instances for Ansible controller server.
```sh
# create user for ansible, in all the provisioning instances.
useradd ansible
usermod -aG sudo ansible
# copy ansible controller's server SSH key to all the provisioning instances.
```

### 2. Services should be up and running (mysql, mongo, redis, postgres, elasticsearch, kafka, etc.)

### 3. The service's exporter port should be open from the monitoring server(where we going to install prometheus, grafana, and alertmanger.)
```yaml
node: 9100
process: 9256
mysql: 9104
mongodb: 9216
redis: 9121
blackbox: 9115
postgres: 9187
elasticsearch: 9114
kafka: 9308

# Note: create a separate security group for all the exporter related ports, and attached it to all the provisioning instances.
```

### 4. Exporter's user should be created in the mysql, mongo, postgres database for metric scraping.
```sh
# For mysql...
CREATE USER 'mysql_exporter'@'%' IDENTIFIED BY 'exportMe321';
GRANT PROCESS, REPLICATION CLIENT, SELECT ON *.* TO 'mysql_exporter'@'%';
FLUSH PRIVILEGES;

# For mongdb...
use admin
db.createUser({ user: "mongodb_exporter", pwd: "exportMe321", roles: [{ role: "clusterMonitor", db: "admin" }, { role: "read", db: "local" }] })

# For postgres...
CREATE USER postgres_exporter WITH PASSWORD 'exportMe321';
GRANT pg_monitor TO postgres_exporter;
```

### 5. All provisioned instance must be tagged appropriately.
```sh
# To enable the monitoring of the infra, the Tag "Environment" should be present with one of the following value.
Environment: prod/production/preprod/staging
# Tag the instance, based on the service deployed on that particular server/VM.
monitoring_node: yes # where we deploy prometheus, grafana, alertmanager, blackbox service.
process: yes # add this tag if we want to monitor process level resource consumption on the instances.
mysql: yes # add this tag to monitor the mysql service.
mongodb: yes # add this tag to monitor the mongodb service.
redis: yes # add this tag to monitor the redis service.
postgres: yes # add this tag to monitor the postgres service.
elasticsearch: yes # add this tag to monitor the elasticsearch service.
kafka: yes # add this tag to monitor the kafka service.

# Note: Tag's key and value both are case-sensitive, so make sure to use the above key and value appropriately.
```

### 6. IAM policy/role access for ansible and monitoring server, as we are using dynamic inventory and service discovery to get the target info from the respective servers.
```yaml
## For AWS

# 1. Create policy
EC2DescribeInstancesPolicy
---
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": "ec2:DescribeInstances",
            "Resource": "*"
        }
    ]
}
---

# 2. Create role and use above-created policy in it and attached it to EC2 instance where prometheus and ansible is running.
EC2DescribeInstancesRole
```

## Requirements:

Update the following variable files:
- vars/main.yml
- vars/blackbkox_targets_vars.yaml

### Required variables:
The following set of variable is required, so update it before using it.
```yaml
# In vars/main.yml file:

client_name: Techpartner

provider: aws

grafana_url: "http://localhost:3000/"

prometheus_url: "http://localhost:9090/"

alertmanager_url: "http://localhost:9093/"

# smtp variable
smtp_smarthost: "email-smtp.us-east-1.amazonaws.com:587"
smtp_from: "alertmanager@mydomain.com"
smtp_require_tls: True
smtp_auth_username: "AkjfwjkeETS7MQ-dummy"
smtp_auth_password: "XXXXXXX"

# slack variable
slack_url: "https://hooks.slack.com/services/XXXX/XXXX/XXXX"
slack_channel: "#medvarsity-alerts"

# email list variable
email_list:
  - "alerts@company.com"
  - "devops@company.com"

# squadcast variable
squadcast_url: "https://api.squadcast.com/v1/incidents"

# deadmans switch variable
deadmans_url: "https://hc-ping.com/e1f74fc4-46ff-4a30-bc2c-XXXXXXXXXXXX"


# In vars/blackbkox_targets_vars.yaml file:

# update the file and add job name following with target's url, module and name respectively.
# update the url, module and name for each target.
# we can have multiple blackbox jobs in it.
blackbox_targets:
  blackbox_http:
    job_name: "blackbox-http"
    targets:
      - url: "http://api.example.com"
        module: "http_2xx"
        name: "web-api"
      - url: "https://example.com"
        module: "http_2xx"
        name: "web"
  blackbox_icmp:
    job_name: "blackbox-icmp"
    targets:
      - url: "1.1.1.1"
        module: "icmp"
        name: "network"
      - url: "8.8.8.8"
        module: "icmp"
        name: "google-dns"
```

### Optional variables:
The following set of variable is optional, but update the value of it depending upon the respective service's monitoring.
```yaml
# In vars/main.yml file:

mysql_host: ""
mysql_port: 3306

mongodb_host: ""
mongodb_port: 3306

postgres_host: ""
postgres_port: 5432

elastic_url:
elastic_ca_crt_path:

kafka_host:
kafka_port:

redis_host:
redis_port:
```

## Dynamic inventory
For dynamic inventory to work make sure the instances were tagged appropriately.
### AWS AutoDiscovery
```sh
export AWS_REGION="us-east-1" # environment variable to declare the aws region.
ansible-inventory -i inventory_aws_ec2.yaml --graph

# output:
@all:
  |--@ungrouped:
  |--@aws_ec2:
  |  |--ip-10-40-10-224.ec2.internal
  |  |--ip-10-40-20-226.ec2.internal
  |  |--ip-10-40-10-137.ec2.internal
  |--@monitoring_node:
  |  |--ip-10-40-10-224.ec2.internal
  |--@process:
  |  |--ip-10-40-10-224.ec2.internal
  |  |--ip-10-40-20-226.ec2.internal
  |--@mongodb:
  |  |--ip-10-40-10-137.ec2.internal
```

## Ansible configuration

The "ansible.cfg" file was available in the current directory.
```ini
[defaults]
su=True
remote_user=ansible

deprecation_warnings=False
display_skipped_hosts=False
host_key_checking=False

gathering = smart
fact_caching = jsonfile
fact_caching_connection = /tmp/ansible_fact_cache

[privilege_escalation]
become = True
become_method = sudo
become_ask_pass = False


[inventory]
enable_plugins = aws_ec2,gcp_compute,azure_rm,ini,yaml

[ssh_connection]
ssh_args = -o ControlMaster=auto -o ControlPersist=60s -o ServerAliveInterval=30 -o ServerAliveCountMax=5 -o StrictHostKeyChecking=no
pipelining = True
control_path = ~/.ansible/pc/ansible-ssh-%%h-%%p-%%r
```

## Manual Playbook execution

### 1. Excute the checklist playbook
This playbook excute and ensure the following things should be present and working.
```
# check and validate correct tagging
# check and validate blackbkox target var file
# check and validate blackbkox rules var file
```
```sh
export AWS_REGION="us-east-1" # environment variable to declare the aws region.
ansible-playbook -i inventory_aws_ec2.yaml cloud-checklist.yml

# output:
...
ok: [localhost] => {
    "provisioning_instances": [
        {
            "Environment": "prod",
            "Instance_ID": "i-0aac2d679f823e7ec",
            "Matching_Tags": [
                "mysql"
            ],
            "Name": "Neeraj-Ldap-server"
        },
        {
            "Environment": "staging",
            "Instance_ID": "i-0b18c5e96dc634945",
            "Matching_Tags": [
                "process"
            ],
            "Name": "Neeraj-Test-server"
        },
        {
            "Environment": "preprod",
            "Instance_ID": "i-0d0481fda1e73cfd2",
            "Matching_Tags": [
                "mongodb"
            ],
            "Name": "Neeraj-VPN-server"
        },
        {
            "Environment": "prod",
            "Instance_ID": "i-058d560cbc3ac36f3",
            "Matching_Tags": [
                "process",
                "monitoring_node"
            ],
            "Name": "Neeraj-Monitoring-server"
        }
    ]
}
...
```

### 2. Excute the monitoring playbook

#### For AWS
To setup monitoring in the AWS server.
```sh
ansible-playbook -i inventory_aws_ec2.yaml cloud-monitoring.yml
```

## Semaphore Playbook execution

### About

### Pre-requiste

### Requirements


## Author information
Neeraj Gupta, DevOps Engineer
