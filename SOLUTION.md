# Full CICD for terraform/ansible using jenkins

## Preconfigs:
- navigate to the install-jenkins.tf
- change the `git-token` from the `variables.tf` and `mykey` change it to YOUR-KEY
- run `terraform init` then apply
- Explore the created resources check the remote bucket for state storage
**Jenkins Plugins needed**:
- Open your Jenkins dashboard and navigate to `Manage Jenkins` >> `Manage Plugins` >> `Available` tab

- Search and select `GitHub Integration, Pipeline: GitHub, Docker, Docker Pipeline, ansible, terraform` plugins, then click to `Install without restart`.

- extras: blueocean for visibility

- Tools configurations: added `terraform` and `ansible` in jenkins from the manage jenkins > tool section
choose from binary you can get the terraform binary from the `/usr/local/bin` dir and ansible check the executable path from ansible `--version`

### Directory Structure 
your directory structure should look something like this feel free to play around but you will need to change the paths in terraform,ansible and the jenkinsfile:
![alt text](image.png)

## Part1:

### 🚀 Starting the Worker Nodes
In this section, we spin up **three EC2 instances** using Terraform. Each instance will later be used to host:

- PostgreSQL (Database)
- Node.js (Backend)
- React (Frontend)

- github setup
echo "# ondia-todo-app" >> README.md
git init
---
git config --global user.email "YOUREMAIL" 
git config --global user.name "YOURNAME"
---
git add README.md
git commit -m "first commit"
--
git branch -M main
git remote add origin https://USERNAME:TOKEN@github.com/USERNAME/ondia-todo-app.git/
git push -u origin main

### Directory Structure 
your directory structure should look something like this feel free to play around but you will need to change the paths in terraform,ansible and the jenkinsfile:
![alt text](image.png)

### terraform code
- create a dir within your newly created repo and call it infra and paste this code block into a `main.tf`
```
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
  backend "s3" {
    bucket = "jenkins-project-backend-serag"
    key = "backend/tf-backend-jenkins.tfstate"
    region = "us-east-1"
  }
}

provider "aws" {
  region = "us-east-1"
}

variable "tags" {
  default = ["postgresql", "nodejs", "react"]
}

resource "aws_instance" "managed_nodes" {
  ami = "ami-0583d8c7a9c35822c"
  count = 3
  instance_type = "t2.micro" 
  key_name = "serag"
  vpc_security_group_ids = [aws_security_group.tf-sec-gr.id]
  iam_instance_profile = aws_iam_instance_profile.ec2-profile.name
  tags = {
    Name = "ansible_${element(var.tags, count.index )}"
    stack = "ansible_project"
    environment = "development"
  }
  user_data = <<-EOF
              #!/bin/bash
              dnf upgrade -y
              EOF
}

resource "aws_iam_role" "aws_access" {
  name = "awsrole-project-rcp"
  assume_role_policy = <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Action": "sts:AssumeRole",
      "Principal": {
        "Service": "ec2.amazonaws.com"
      },
      "Effect": "Allow",
      "Sid": ""
    }
  ]
}
EOF
}

resource "aws_iam_role_policy_attachment" "ecr_full_access" {
  role       = aws_iam_role.aws_access.name
  policy_arn = "arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryFullAccess"
}

resource "aws_iam_instance_profile" "ec2-profile" {
  name = "jenkins-project-profile-rcp"
  role = aws_iam_role.aws_access.name
}

resource "aws_security_group" "tf-sec-gr" {
  name = "project208-sec-gr"
  tags = {
    Name = "project208-sec-gr"
  }

  ingress {
    from_port   = 22
    protocol    = "tcp"
    to_port     = 22
    cidr_blocks = ["0.0.0.0/0"]
  }
  ingress {
    from_port   = 5000
    protocol    = "tcp"
    to_port     = 5000
    cidr_blocks = ["0.0.0.0/0"]
  }
  ingress {
    from_port   = 3000
    protocol    = "tcp"
    to_port     = 3000
    cidr_blocks = ["0.0.0.0/0"]
  }
  ingress {
    from_port   = 5432
    protocol    = "tcp"
    to_port     = 5432
    cidr_blocks = ["0.0.0.0/0"]
  }

  egress {
    from_port   = 0
    protocol    = -1
    to_port     = 0
    cidr_blocks = ["0.0.0.0/0"]
  }
}

output "react_ip" {
  value = "http://${aws_instance.managed_nodes[2].public_ip}:3000"
}

output "node_public_ip" {
  value = aws_instance.managed_nodes[1].public_ip
}

output "postgre_private_ip" {
  value = aws_instance.managed_nodes[0].private_ip
}
```

### Code breakdown
here are code blocks of interest to dive deaper into:
This bucket should match the same exact name you created in the `preconfig` section where we get our state stored
```
backend "s3" {
    bucket = "jenkins-project-backend-serag"
    key = "backend/tf-backend-jenkins.tfstate"
    region = "us-east-1"
  }

```

- Next we use a count block to create three instances based on the tags
```
variable "tags" {
  default = ["postgresql", "nodejs", "react"]
}

resource "aws_instance" "managed_nodes" {
  ami = "ami-0583d8c7a9c35822c"
  count = 3
  instance_type = "t2.micro" 
  key_name = "serag"
  vpc_security_group_ids = [aws_security_group.tf-sec-gr.id]
  iam_instance_profile = aws_iam_instance_profile.ec2-profile.name
  tags = {
    Name = "ansible_${element(var.tags, count.index )}"
    stack = "ansible_project"
    environment = "development"
  }
  user_data = <<-EOF
              #!/bin/bash
              dnf upgrade -y
              EOF
}
```

### Reaching your nodes with ansible
From within your given files you will need to 

- copy your pem key that you used to connect to your nodes typically located in your LOCAL ~/.ssh/ directory

-  copy both `ansible.cfg `and rename it to `.ansible.cfg` and copy the `inventory_aws_ec2.yml` file 

to check your hosts run:
- `ansible-inventory --graph` to check host names you should get something like _ansible_react, _ansible_nodejs are the groups defined based on how we tag our ec2 in our main.tf file: 
![alt text](image-1.png)


- `ansible all ping` to check reachability




## Part2:

Once your EC2 instances are provisioned and reachable, it's time to **configure each one using Ansible**. This is where your infrastructure transforms into a working application stack.

We will:
- Install Docker and AWS CLI on all nodes
- Log in to AWS ECR
- Run Docker containers for PostgreSQL, Node.js, and React

### playbook
We create 4 playbooks:
- one to install docker on each machime (we will be running docker containers to host our apps)

- the other three playbooks ensure we restart a container on its respective port 





```
- name: install docker and config
  hosts: _development
  become: true
  vars:
    aws_region: us-east-1
    ecr_registry: 046402772087.dkr.ecr.us-east-1.amazonaws.com
  tasks:
    - name: update all packages
      ansible.builtin.yum:
        name: '*'
        state: latest

    - name: Remove docker if installed from CentOS repo
      ansible.builtin.yum:
        name:
          - docker
          - docker-client
          - docker-client-latest
          - docker-common
          - docker-latest
          - docker-latest-logrotate
          - docker-logrotate
          - docker-engine
          - python3-pip
        state: removed

    - name: Install yum utils
      ansible.builtin.yum:
        name: 
          - yum-utils
          - unzip
        state: latest

    - name: Add Docker repo
      ansible.builtin.get_url:
        url: https://download.docker.com/linux/centos/docker-ce.repo
        dest: /etc/yum.repos.d/docker-ce.repo

    - name: Install Docker
      ansible.builtin.package:
        name: docker-ce
        state: latest

    - name: Install python3-pip
      ansible.builtin.yum:
        name: python3-pip
        state: latest
    - name: Install Docker SDK and correct requests package
      ansible.builtin.pip:
        name: 
          - docker==6.1.3
          - requests==2.31.0
        state: latest



    - name: Add user ec2-user to docker group
      ansible.builtin.user:
        name: ec2-user
        groups: docker
        append: yes

    - name: Start Docker service
      ansible.builtin.service:
        name: docker
        state: started
        enabled: yes

    - name: install aws cli
      ansible.builtin.get_url:
        url: https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip
        dest: /home/ec2-user/awscliv2.zip

    - name: unzip zip file
      ansible.builtin.unarchive:
        src: /home/ec2-user/awscliv2.zip
        dest: /home/ec2-user
        remote_src: True

    - name: run the installer
      ansible.builtin.command:
      args:
        cmd: "./aws/install"
        creates: /usr/local/bin/aws

    - name: log in to AWS ec2-user
      become: true
      become_user: ec2-user
      ansible.builtin.shell: |
        export PATH=/usr/local/bin:$PATH
        source ~/.bash_profile
        aws ecr get-login-password --region {{ aws_region }} | docker login --username AWS --password-stdin {{ ecr_registry }}

- name: postgre database config
  hosts: _ansible_postgresql
  become: true
  become_user: ec2-user
  vars:
    container_name: postgre
    image_name: 046402772087.dkr.ecr.us-east-1.amazonaws.com/clarusway-repo/cw-todo-app:postgre
  tasks:
    - name: remove postgre container
      community.docker.docker_container:
        name: "{{ container_name }}"
        state: absent
        force_kill: true
        api_version: 1.25

    - name: remove postgre image
      community.docker.docker_image:
        name: "{{ image_name }}"
        state: absent

    - name: Launch postgresql docker container
      community.docker.docker_container:
        name: "{{ container_name }}"
        image: "{{ image_name }}"
        state: started
        ports:
          - "5432:5432"
        env:
          POSTGRES_PASSWORD: "Pp123456789"
        volumes:
          - /db-data:/var/lib/postgresql/data

- name: Nodejs Server configuration
  hosts: _ansible_nodejs
  become: true
  become_user: ec2-user
  vars:
    container_name: nodejs
    image_name: 046402772087.dkr.ecr.us-east-1.amazonaws.com/clarusway-repo/cw-todo-app:nodejs
  tasks:
    - name: remove nodejs container
      community.docker.docker_container:
        name: "{{ container_name }}"
        state: absent
        force_kill: true

    - name: remove nodejs image
      community.docker.docker_image:
        name: "{{ image_name }}"
        state: absent

    - name: Launch nodejs docker container
      community.docker.docker_container:
        name: "{{ container_name }}"
        image: "{{ image_name }}"
        state: started
        ports:
          - "5000:5000"

- name: React UI Server configuration
  hosts: _ansible_react
  become: true
  become_user: ec2-user
  vars:
    container_name: react
    image_name: 046402772087.dkr.ecr.us-east-1.amazonaws.com/clarusway-repo/cw-todo-app:react
  tasks:
    - name: remove react container
      community.docker.docker_container:
        name: "{{ container_name }}"
        state: absent
        force_kill: true

    - name: remove react image
      community.docker.docker_image:
        name: "{{ image_name }}"
        state: absent

    - name: Launch react docker container
      community.docker.docker_container:
        name: "{{ container_name }}"
        image: "{{ image_name }}"
        state: started
        ports:
          - "3000:3000"


```

dont run the playbook just yet !  

## Part3: adding the jenkins file 

```
pipeline{
    agent any
    tools {
        ansible 'ansible'
        terraform 'terraform'
    }
    environment {
        PATH=sh(script:"echo $PATH:/usr/local/bin", returnStdout:true).trim()
        AWS_REGION = "us-east-1"
        AWS_ACCOUNT_ID=sh(script:'export PATH="$PATH:/usr/local/bin" && aws sts get-caller-identity --query Account --output text', returnStdout:true).trim()
        ECR_REGISTRY="${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com"
        APP_REPO_NAME = "clarusway-repo/cw-todo-app"
    }

    stages{ 

        stage('Create Infrastructure for the App') {
            steps {
                echo 'Creating Infrastructure for the App on AWS Cloud'
                sh 'terraform -chdir=./infra init'
                sh 'terraform -chdir=./infra apply --auto-approve'
            }
        }

        stage('Create ECR Repo') {
            steps {
                echo 'Creating ECR Repo for App'
                sh '''
                aws ecr describe-repositories --region ${AWS_REGION} --repository-name ${APP_REPO_NAME} || \
                aws ecr create-repository \
                  --repository-name ${APP_REPO_NAME} \
                  --image-scanning-configuration scanOnPush=false \
                  --image-tag-mutability MUTABLE \
                  --region ${AWS_REGION}
                '''
            }
        }

        stage('Build App Docker Image') {
            steps {
                echo 'Building App Image'
                script {
                    env.NODE_IP = sh(script: 'terraform -chdir=./infra output -raw node_public_ip', returnStdout:true).trim()
                    env.DB_HOST = sh(script: 'terraform -chdir=./infra output -raw postgre_private_ip', returnStdout:true).trim()
                    env.DB_NAME = sh(script: 'aws --region=us-east-1 ssm get-parameters --names "db_name" --query "Parameters[*].{Value:Value}" --output text', returnStdout:true).trim()
                    env.DB_PASSWORD = sh(script: 'aws --region=us-east-1 ssm get-parameters --names "db_password" --query "Parameters[*].{Value:Value}" --output text', returnStdout:true).trim()
                }
                sh 'echo ${DB_HOST}'
                sh 'echo ${NODE_IP}'
                sh 'echo ${DB_NAME}'
                sh 'echo ${DB_PASSWORD}'
                sh 'envsubst < node-env-template > ./nodejs/server/.env'
                sh 'cat ./nodejs/server/.env'
                sh 'envsubst < react-env-template > ./react/client/.env'
                sh 'cat ./react/client/.env'
                sh 'docker build -t "$ECR_REGISTRY/$APP_REPO_NAME:postgre" -f ./postgresql/dockerfile-postgresql .'
                sh 'docker build -t "$ECR_REGISTRY/$APP_REPO_NAME:nodejs" -f ./nodejs/dockerfile-nodejs .'
                sh 'docker build -t "$ECR_REGISTRY/$APP_REPO_NAME:react" -f ./react/dockerfile-react .'
                sh 'docker image ls'
            }
        }

        stage('Push Image to ECR Repo') {
            steps {
                echo 'Pushing App Image to ECR Repo'
                sh 'aws ecr get-login-password --region ${AWS_REGION} | docker login --username AWS --password-stdin "$ECR_REGISTRY"'
                sh 'docker push "$ECR_REGISTRY/$APP_REPO_NAME:postgre"'
                sh 'docker push "$ECR_REGISTRY/$APP_REPO_NAME:nodejs"'
                sh 'docker push "$ECR_REGISTRY/$APP_REPO_NAME:react"'
            }
        }

        stage('wait the instance') {
            steps {
                script {
                    echo 'Waiting for the instance'
                    id = sh(script: 'aws ec2 describe-instances --filters Name=tag-value,Values=ansible_postgresql Name=instance-state-name,Values=running --query Reservations[*].Instances[*].[InstanceId] --output text',  returnStdout:true).trim()
                    sh 'aws ec2 wait instance-status-ok --instance-ids $id'
                }
            }
        }

        stage('Deploy the App') {
            steps {
                echo 'Deploy the App'
                sh 'ls -l'
                sh 'ansible --version'
                sh 'sleep 180'
                sh 'ansible-inventory --graph'
                ansiblePlaybook credentialsId: 'project-208-rcp', disableHostKeyChecking: true, installation: 'ansible', inventory: 'inventory_aws_ec2.yml', playbook: 'docker-project.yml'
             }
        }

        stage('Destroy the infrastructure'){
            steps{
                timeout(time:5, unit:'DAYS'){
                    input message:'Approve terminate'
                }
                sh """
                docker image prune -af
                terraform -chdir=./infra destroy --auto-approve
                aws ecr delete-repository \
                  --repository-name ${APP_REPO_NAME} \
                  --region ${AWS_REGION} \
                  --force
                """
            }
        }

    }

    post {
        always {
            echo 'Deleting all local images'
            sh 'docker image prune -af'
        }

        failure {

            echo 'Delete the Image Repository on ECR due to the Failure'
            sh """
                aws ecr delete-repository \
                  --repository-name ${APP_REPO_NAME} \
                  --region ${AWS_REGION}\
                  --force
                """
            echo 'Deleting Terraform Stack due to the Failure'
                sh 'terraform destroy --auto-approve'
        }
    }  

}   





```

### jenkins file breakdown


#### defining tools used and env variables 
Notes:
- both region and AWS_ACCOUNT_ID you will plug into the ecr_registry format
- export PATH is pointing to /usr/loca/bin where we have our aws cli located  
```
tools {
        ansible 'ansible'
        terraform 'terraform'
    }
    environment {
        PATH=sh(script:"echo $PATH:/usr/local/bin", returnStdout:true).trim()
        AWS_REGION = "us-east-1" 
        AWS_ACCOUNT_ID=sh(script:'export PATH="$PATH:/usr/local/bin" && aws sts get-caller-identity --query Account --output text', returnStdout:true).trim()
        ECR_REGISTRY="${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com"
        APP_REPO_NAME = "clarusway-repo/cw-todo-app"
    }

```
#### Similarly in the build phase
```
echo 'Building App Image'
                script {
                    env.NODE_IP = sh(script: 'terraform -chdir=./infra output -raw node_public_ip', returnStdout:true).trim()
                    env.DB_HOST = sh(script: 'terraform -chdir=./infra output -raw postgre_private_ip', returnStdout:true).trim()
                    env.DB_NAME = sh(script: 'aws --region=us-east-1 ssm get-parameters --names "db_name" --query "Parameters[*].{Value:Value}" --output text', returnStdout:true).trim()
                    env.DB_PASSWORD = sh(script: 'aws --region=us-east-1 ssm get-parameters --names "db_password" --query "Parameters[*].{Value:Value}" --output text', returnStdout:true).trim()
                }
                sh 'echo ${DB_HOST}'
                sh 'echo ${NODE_IP}'
                sh 'echo ${DB_NAME}'
                sh 'echo ${DB_PASSWORD}'
                sh 'envsubst < node-env-template > ./nodejs/server/.env'
                sh 'cat ./nodejs/server/.env'
                sh 'envsubst < react-env-template > ./react/client/.env'
                sh 'cat ./react/client/.env'
                sh 'docker build -t "$ECR_REGISTRY/$APP_REPO_NAME:postgre" -f ./postgresql/
```


#### in deploy app stage
we configure the credentials within the jenkins console make sure the username is ec2-user
```
    sh 'ansible-inventory --graph'
    ansiblePlaybook credentialsId: 'project-208-rcp', disableHostKeyChecking: true, installation: 'ansible', inventory: 'inventory_aws_ec2.yml', playbook: 'docker-project.yml'
```

## Key Optimizations
- This is good for an infra stucture provisioning pipeline but it could be better adding additional separation of concerns 
**consider using:**
- A multi-repo approach (dig into the differences between mutli and monorepo designs and their pros and cons)
- consider using multiple pipelines for each deployment (weigh in the pros and cons aswell from our current approach)
- try different methods to deploying the app maybe try shipping to the vm directly(without containers)/ 
- pipeline is still filled with lots of bash scripting try to clean it up see if you can manage your env variables someother place instead of defining them inline
- update the jenkins versions and explore the vulnerability versions
