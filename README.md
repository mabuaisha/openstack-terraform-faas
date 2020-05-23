# faas-exp
This project is created as part of Thesis in Software Engineering for measuring the performance of Serverless using 3 different container orchestration frameworks (Kubernetes, Docker Swarm, Nomad) where Openfass framework is used as a case study.
The following items are automated by this project:

1. Automate infrastructure provisioning on AWS & Openstack cloud.
2. Automate deployment Serverless functions on Kubernetes, Docker Swarm, Nomad.
3. Automate test cases generation.  

This project is mainly has two parts:
1. Infrastructure provisioning, clusters setup and serverless framework deployment.
2. Experiment runner

## Part1

This part focus on infrastructure provisioning, configuration and serverless framework deployment where two cloud providers are supported (AWS & Openstack).

### Prerequisites 

In order to be able to setup part1 the following items are needed:

1. [Terraform](https://releases.hashicorp.com/terraform/0.12.21/)

2. Setup one of the Public Cloud API credentials:

    - AWS by creating file `.env` file  and source it `source .env`
    
        ```
        export AWS_ACCESS_KEY_ID="XXXXXXXXXXXXXXXXXXX"
        export AWS_SECRET_ACCESS_KEY="XXXXXXXXXXXXXXXXXXX"
        export AWS_DEFAULT_REGION="us-east-1"

        ```
    - Download Openstack API v3 RC file from the Openstack Horizon and source it `source YOUR-OPENSTACK-PROVIDERE-openrc.sh`

2. Linux OS Centos 7.x (with user "centos")

### Terraform files

This project has terraform files for the following resources:

1. [Bastion](/bastion)
2. [Docker Swarm](/docker-swarm)
3. [Nomad](/nomad)
4. [Kubernetes](/k8s)
5. [HAproxy](/haproxy)
6. [FTP](/ftp)


#### Bastion

Bastion is created as jumphost that helps to create all other resources and run the experiment as it is the only resource has a public ip address and all other has private ip addresses. where the following resources get created

1. VPC
2. Public Subnet
3. Private Subnet
4. Internet Gateway
5. NAT Gateway
6. Routing Tables
7. KeyPair


##### AWS

The following inputs are used to create bastion resource on AWS as specified on the [variables](/bastion/aws/variables.tf) file:

- `availability_zone`: Availability zone. The default value is `us-east-1b`.
- `vpc_cidr`: The VPC cidr. The default value is `10.0.0.0/16`.
- `public_subnet_cidr`: The public subnet cidr. The default `10.0.1.0/24`.
- `private_subnet_cidr`: The private subnet cidr. The default `10.0.2.0/24`.
- `public_key`: The public key required to attach for keypair. The default location `~/.ssh/faas_ssh.pub`.
- `private_key`: The private key required to ssh for instances. The default location `~/.ssh/faas_ssh`.
- `env_name`: The environment name. The default value is `serverless-env`.
- `volume_size`: The volume size for bastion instance. The default value is `15` GB.
- `instance_type`: The instance type for AWS. The default value is `t3a.medium`.
- `image_id`: The image id where instance is creating from. The default value is `ami-0affd4508a5d2481b` (Centos 7.6) 


Note: The generation for the both private and public keys can be done using 
```
    ssh-keygen -t rsa -b 4096 -C "youremail@domain.com"
```

To create bastion update your variables file and run the following command inside [aws](/bastion/aws):

```
terraform init
terraform apply

```

Terraform outputs
```
floating_ip = "19.210.11.20"
vpc_id = 'vpc-0a2d1c0a512011a42
subnet_id = "subnet-0a6d2c0a513011a42"
security_group_ids = ["sg-168975527090df7c4"]
```

##### Openstack

The following inputs are used to create bastion resource on Openstack as specified on the [variables](/bastion/openstack/variables.tf) file:

- `external_network_name`: External network name. Required field its based on the Openstack provider could be (ext, ext-net, ..etc).
- `flavor`: The type of the machine need to be created. (Required) Flavor is a custom in Openstack and varies from provider to provider.
- `image`: The image type where instance is going to be created. (Required) This project use Centos and the name varies from from provider to provider.
- `public_key`: The public key required to attach for keypair. The default location `~/.ssh/faas_ssh.pub`.
- `private_key`: The private key required to ssh for instances. The default location `~/.ssh/faas_ssh`.
- `env_name`: The environment name. The default value is `serverless-env`.
- `subnet_cidr`: The subnet cidr. The default `192.168.0.0/24`.
- `dns_nameservers`: An array of DNS name server names used by hosts in this subnet. The default `["8.8.8.8", "8.8.4.4"]`.


Note: The generation for the both private and public keys can be done using 
```
    ssh-keygen -t rsa -b 4096 -C "youremail@domain.com"
```

To create bastion update your variables file and run the following command inside [aws](/bastion/aws):

```
terraform init
terraform apply -var-file=inputs.tfvars

```

`inputs.tfvars` should be created to fill the required values and override any default value.

Terraform outputs
```
floating_ip = "18.222.12.12"
network_id = '2fc090f1-b00a-4ac6-be40-251w4b184471
```

#### Docker Swarm

This module is used to create required resources for docker swarm which includes at least 3 machines: 1 manager + 2 workers.

The current Terraform code support creating only 1 manager and multiple workers and in future we are going to update the code to support creating multiple managers. 

Notes: 
1. Make sure [jq](https://stedolan.github.io/jq/) is installed on the machine that execute terraform.
2. Make sure the ssh agent is up and running `eval "$(ssh-agent -s)"`
3. Make sure to add your private key to the ssh-agent `ssh-add ~/.ssh/faas_ssh`


##### AWS

The following inputs are used to create bastion resource on AWS as specified on the [variables](/docker-swarm/aws/variables.tf) file:

- `subnet_id`: The private subent id. Required.
- `bastion_ip`: The bastion ip. Required.
- `docker_username`: The dockerhub username. Required
- `docker_password`: The dockerhub password. Required
- `security_group_ids`: The list of security group ids. Required
- `private_key`: The private key required to ssh for instances. The default location `~/.ssh/faas_ssh`.
- `worker_name`: The worker name. The default value is `docker-swarm`.
- `env_name`: The environment name. The default value is `serverless-env`.
- `manager_count`: The number of managers. The default value is `1` and should be 1.
- `worker_count`: The number of workers. The default value is `2`.
- `volume_size`: The volume size for bastion instance. The default value is `15` GB.
- `instance_type`: The instance type for AWS. The default value is `t3a.large`.
- `image_id`: The image id where instance is creating from. The default value is `ami-0affd4508a5d2481b` (Centos 7.6)  

To create bastion update your variables file and run the following command inside [aws](/bastion/aws):

```
terraform init
terraform apply -var-file=inputs.tfvars

```

`inputs.tfvars` should be created to fill the required values and override any default value.

Terraform outputs
```
worker-ips = [10.0.2.111, 10.0.2.130]
manager-ips = [10.0.2.112]
worker_token = SWMTKN-1-49nj1cmql0jkz5s954yi3oex3nedyz0fb0xx14ie39trti4wxv-8vxv8rssmk743ojnwacrr2e7c
manager_token =49nj1cmql0jkz5s954yi3oex3nedyz0fb0xx14ie39trti4wxv-8vxv8rssmk743ojnwacrr2e7c 
```

##### Openstack

The following inputs are used to create bastion resource on Openstack as specified on the [variables](/docker-swarm/openstack/variables.tf) file:

- `network_id`:The network id where bastion resourced created in. Required.
- `bastion_ip`: The bastion ip. Required.
- `docker_username`: The dockerhub username. Required
- `docker_password`: The dockerhub password. Required
- `flavor`: The type of the machine need to be created. (Required) Flavor is a custom in Openstack and varies from provider to provider.
- `image`: The image type where instance is going to be created. (Required) This project use Centos and the name varies from from provider to provider.
- `private_key`: The private key required to ssh for instances. The default location `~/.ssh/faas_ssh`.
- `env_name`: The environment name. The default value is `serverless-env`.
- `worker_name`: The worker name. The default value is `docker-swarm`.
- `manager_count`: The number of managers. The default value is `1` and should be 1.
- `worker_count`: The number of workers. The default value is `2`.


#### Nomad
#### Kubernetes
#### HAproxy
#### FTP

## Part2

