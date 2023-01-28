### Instructions to properly set up a production-ready NextJS App with AWS
This is a summary of how to start your own NextJS app, create the repo on Github, upload later in an AWS EC2 Instance and automate the process with AWS Codebuild, CodeDeploy, & CodePipeline.

After following these instructions you should be able to:
- Create a NextJS App
- Create an AWS EC2 instance
- Be able to SSH to an EC2 instance
- Prepare the instance as a Node environment
- Configure Nginx service for proper routing
- Configure domain and add SSL
- Automate the source push, build, deploy process using AWS CodeBuild, CodeDeploy, & CodePipeline

## Step 1 - Create a NextJS App with AWS Scripts and Configs
```
npx create-next-app my-app
cd my-app

// instructions for AWS CodeBuild, file content below
touch buildspec.yml // creates a file

// instructions for AWS CodeDeploy, file content below
touch appspec.yml // creates a file

mkdir scripts
cd scripts
touch before_install.sh
touch after_install.sh
touch application_start.sh
```
Upload this to Github

**buildspec.yml** - used by AWS CodeBuild
```
version: 0.2

phases:
  install:
    runtime-versions:
      nodejs: 12

    commands:
      # install npm
      - npm install

  build:
    commands:
      # run build script
      - npm build

artifacts:
  # include all files required to run application
  # notably excluded is node_modules, as this will cause overwrite error on deploy
  files:
    - assets/**/*
    - components/**/*
    - containers/**/*
    - pages/**/*
    - public/**/*
    - scripts/**/*
    - settings/**/*
    - styles/**/*
    - jsconfig.json
    - package.json
    - appspec.yml
    - postcss.config.js
    - tailwind.config.js

```

**appspec.yml** - used by AWS CodeDeploy
```
version: 0.0
os: linux
files:
  - source: /
    destination: /home/ec2-user/app-frontend
    overwrite: true
permissions:
  - object: /
    pattern: "**"
    owner: ec2-user
    group: ec2-user

hooks:
  BeforeInstall:
    - location: scripts/before_install.sh
      timeout: 300
      runas: root
  AfterInstall:
    - location: scripts/after_install.sh
      timeout: 300
      runas: root
  ApplicationStart:
    - location: scripts/application_start.sh
      timeout: 300
      runas: root
```

**before_install.sh**
```
#!/bin/bash
cd /home/ec2-user/server
curl -sL https://rpm.nodesource.com/setup_14.x | sudo -E bash -
yum -y install nodejs npm
```

**after_install.sh**
```
#!/bin/bash
cd /home/ec2-user/app-frontend
npm install
npm install pm2 -g
```
**application_start.sh**
```
#!/bin/bash
cd /home/ec2-user/app-frontend
npm run build
pm2 start npm --name "sweetcollective" -- start
pm2 startup
pm2 save
pm2 restart all
```
## Step 2 - Create and test SSH to an AWS EC2 instance

**Create an IAM Role : AWS IAM**
*AWS service roles are used to grant permissions to an AWS service so it can access AWS resources.*
1. Create Role
Role name: EC2RoleForS3
Description: Allows EC2 instances to access AWS S3 Bucket
Policies: AWSS3ReadOnlyAccess

2. Create Role
Role name: EC2RoleForCodeDeploy
Description: Allows EC2 instances to access AWS CodeDeploy
Policies: AWSCodeDeployRole

**Create an EC2 Instance : AWS EC2**
In AWS Console, head over to EC2 and create an instance, select t2.micro
Add Tag key:"Name", Value: "nextjs-app"
Security group, create one or use an existing
Before launching, you will be prompted to create an RSA key pair
Save this securely in your local machine (don't lose this)

```
// go to the keypair folder location
sudo chmod 400 keypair.pem (or keypair.cer)
ssh -i keypair.pem ec2-user@<ec2-instance-public-ip>
```

Once you're in SSH terminal,

```
// install nvm
curl https://raw.githubusercontent.com/creationix/nvm/master/install.sh | bash

// install LTS (Gallium version)
nvm install 16.13.0 

// install node
nvm install node

// install pm2 (Process Manager)
npm i -g pm2

// install nginx
sudo amazon-linux-extras install nginx1
```

## Step 4 - Set and Configure Nginx 
in etc/nginx/nginx.conf

```
sudo vim nginx.conf

...
location / {
        # reverse proxy for next server
        	proxy_pass http://127.0.0.1:3005; # your nextJs service and port
        	proxy_http_version 1.1;
        	proxy_set_header X-Real-IP $remote_addr;
       		proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
       		proxy_set_header X-Forwarded-Proto $scheme;
        	# we need to remove this 404 handling
      		# because next's _next folder and own handling
        	# try_files $uri $uri/ =404;
    	}
...

// restart service
sudo systemcl restart nginx

```

### Step 5 - AWS CodeBuild, CodeDeploy, CodePipeline

**Create a new CodeDeploy Application - AWS CodeDeploy**
Create New Application
Add Name then finish

Create Deployment Group
Add name
Select ServiceRole - EC2RoleForCodeDeploy we created above
Deployment Type - leave default

- Environment Configuration
Check Amazon EC2 Instance
Add Key: Name, Value: nextjs-app // refer above
- Agent Configuration
Install AWS CodeDeploy Agent - Only Once
- Deployment settings - leave default
- Load Balancing - uncheck box
Create Deployment Group

**Create a new Pipeline - AWS CodePipeline**
1) Choose Pipeline Settings
Add a Name
Create New Service Role and check Allow AWS CodePipeline to create a service role
Next

2) Choose Source
Select Github (v2)
Connect your Github
Select the app repository
Select branch
Check start pipeline on source code change
Leave CodePipeline default selected
Next

4) **Add Build Stage - AWS CodeBuild**
Select AWS CodeBuild
Create Project

// CodeBuild tab opens
Fill in CodeBuild Details
Environment
- Managed Image
- Select OS Amazon Linux 2
- Runtime: Standard
- Create ServiceRole, any name

Buildspec (default)
Continue to pipeline
// CodeBuild tab closes

Dont refresh the page
Select Previous, then Next
Select Project name you just created on a different tab
Next

5) Add Deploy Stage
Select AWS CodeDeploy
Select Application Name
Select Deployment Group
Review then Create

Congratulations! Pipeline should run right away after creation, if all steps were followed, everything should go smoothly.

**Known Errors:**
- Connection refused when visiting ec2 public ip address
  solution 1: check if 12.12.12.12 is http or https in browser url
  solution 2: check EC2 security group attached and see inbound rules, add HTTP, TCP, Port 80, 0.0.0.0/0
  
  It's also possible that you've turned off/on the instance without having an Elastic IP address attached to it
  In this case then attach an elastic ip so that it doesn't change every on/off
- Connection refused when trying to SSH
  solution 1: check EC2 security group attached and see inbound rules, add SSH, TCP, Port 22, My IP Address
- Port 3000 already in use
  solution 1: I've so far edited package.json and changed the npm run start command to
  ```
  ...
  "start": "next start -p 3005",
  ....
  ```
  
references:
https://docs.aws.amazon.com/codedeploy/latest/userguide/codedeploy-agent-operations-install-linux.html
https://blog.devgenius.io/deploy-a-reactjs-application-to-aws-ec2-instance-using-aws-codepipeline-3df5e4157028
