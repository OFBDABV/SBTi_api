# SBTi Temperature Alignment tool API
This package helps companies and financial institutions to assess the temperature alignment of current
targets, commitments, and investment and lending portfolios, and to use this information to develop 
targets for official validation by the SBTi.

Under the hood, this API uses the SBTi Python module. The complete structure that consists of a Python module, API and a UI looks as follows:

    +-------------------------------------------------+
    |   UI     : Simple user interface on top of API  |
    |   Source : github.com/OFBDABV/SBTi_ui           |
    |   Install: via source or dockerhub              |
    |            docker.io/sbti/ui:latest             |
    |                                                 |
    | +-----------------------------------------+     |
    | | REST API: Dockerized Flask/NGINX        |     |
    | | Source : github.com/OFBDABV/SBTi_api    |     |
    | | Install: via source or dockerhub        |     |
    | |          dcoker.io/sbti/sbti/api:latest |     |
    | |                                         |     |
    | | +---------------------------------+     |     |
    | | |                                 |     |     |
    | | |Core   : Python Module           |     |     |
    | | |Source : github.com/OFBDABV/SBTi |     |     |
    | | |Install: via source or PyPi      |     |     |
    | | |                                 |     |     |
    | | +---------------------------------+     |     |
    | +-----------------------------------------+     |
    +-------------------------------------------------+


## Structure
The folder structure for this project is as follows:

    .
    ├── .github                 # Github specific files (Github Actions workflows)
    ├── app                     # Flask app files for the API endpoints
    └── config                  # Config files for the Docker container

## Deployment
This service can be deployed in two ways, either as a standalone API or in conjunction with a no-frills UI.
For both of these options a docker configuration has been set up. 

In order to run the docker container locally on non linux machines one needs to install [Docker Desktop](https://www.docker.com/products/docker-desktop) available for Mac and Windows

### API-only
Both the master and DEV branch have public images at docker hub. To run them, use the following commands: 

```bash
docker run -d -p 5000:8080 sbti/sbti_tool:latest # to run  the latest stable release
```
or 
```bash
docker run -d -p 5000:8080 sbti/sbti_tool:DEV # to run the latest DEV version
```
The API documentation should now be available at [http://localhost:5000/docs/](http://localhost:5000/docs/).

### API and UI
To launch both the API and the UI, you need to use the provided docker-compose files.
This will spin up two containers that work in conjunction with one another.

To launch the latest release:
```bash
docker-compose -f docker-compose-ui.yml -d --build
``` 

To use your local code-base:
```bash
docker-compose -f docker-compose-ui-dev.yml -d --build
``` 

The UI should now be available at [http://localhost:5000/](http://localhost:5000/) and check [http://localhost:5001/docs/](http://localhost:5001/docs/) for the API documentation

To build an run the docker container locally use the following command:
```bash
docker-compose up -d --build
```

## Deploy on Amazon Web Services
These instructions assume that you've installed and configured the Amazon [AWS CLI tools](https://docs.aws.amazon.com/cli/latest/userguide/install-cliv2.html) and the [ECS CLI tools](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/ECS_CLI_Configuration.html) with an IAM account that has at least write access to ECS and EC2 and the capability of creating AIM roles.

1. Create a repository. 
```bash
aws ecr create-repository --repository-name sbti-ecs
```

*Output:*

```
{
    "repository": {
        "registryId": "aws_account_id",
        "repositoryName": "sbti-ecs",
        "repositoryArn": "arn:aws:ecr:region:aws_account_id:repository/sbti-ecs",
        "createdAt": 1505337806.0,
        "repositoryUri": "aws_account_id.dkr.ecr.region.amazonaws.com/sbti-ecs"
    }
}
```

2. Create your own AWS docker compose launch file
```bash
cp docker-compose_aws_example.yml docker-compose_aws.yml
```
2. Update the docker-compose_aws.yml file with the repository URI.
3. Build the docker image
```bash
docker-compose -f docker-compose_aws.yml build
```
4. Check if the image has indeed been build
```bash
docker image ls
```
5. Push the image to the repository
```bash
docker-compose -f docker-compose_aws.yml push
```
6. Configure the cluster. You can update the region and names as you see fit
```bash
ecs-cli configure --cluster sbti-ecs-cluster --region eu-central-1 --config-name sbti-ecs-conf --cfn-stack-name sbti-ecs-stack --default-launch-type ec2
```
7. Create a new key pair. The result of this command is a key. Store this safely as you can later use it to access your instance through SSH.
```bash
aws ec2 create-key-pair --key-name sbti
```
8. Create the instance that'll run the image. Here we used 1 server of type t2.medium. Change this as you see fit.
```bash
ecs-cli up --keypair sbti --capability-iam --size 1 --instance-type t2.medium --cluster-config sbti-ecs-conf
```
9. Update the server and make it run the docker image.
```bash
ecs-cli compose -f docker-compose_aws.yml up --cluster-config sbti-ecs-conf
```
10. Now that the instance is running we can't access it yet. That's because NGINX only listens to localhost. We need to change this to make sure it's accessible on the WWW.
11. Login to the Amazon AWS console
12. Go to the EC2 service
13. In the instance list find the instance running the Docker image
14. Copy the public IP address of the instance
15. In ```config/flask-site-nginx.conf``` update the server name to the public IP.
16. Now we need to rebuild and re-upload the image.

```bash
docker-compose -f docker-compose_aws.yml build --no-cache
docker-compose -f docker-compose_aws.yml push
ecs-cli compose -f docker-compose_aws.yml up --cluster-config sbti-ecs-conf --force-update
```
17. You should now be able to access the API.

> :warning: This will make the API publicly available on the world wide web! Please note that this API is not protected in any way. Therefore it's recommended to run your instance in a private subnet and only access it through there. Alternatively you can change the security group settings to only allow incoming connections from your local IP or company VPN.  
