this is nowhere near done yet - if you're reading this, check back in a year


# AWS Notes
### Github -> ECR setup
1. Create IAM user for github actions
    - iam -> users -> create user
        - name: "github-actions"
        - Add permission "AmazonEC2ContainerRegistryFullAccess"
2. Create an access key for the user
3. Copy the key id and secret into github repository -> settings -> secrets -> actions
4. create a repository in ECR, either through the web interface or via the CLI like so
    ```
    aws ecr create-repository --repository-name nginx --region us-east-2
    ```
5. push something to main to test it out! you should see the new image showing up in ECR

### ECR -> ECS setup
1. Create a VPC
    - VPC -> create vpc and more
    - default settings, disable NAT
2. Create security group
    - EC2 -> security groups -> create
    - name: "permissive-internet"
    - inbound rules:
        - TCP, 80, any ipv4
        - TCP, 443, any ipv4
        - TCP, 80, any ipv6
        - TCP, 443, any ipv6
        - ICMP, any ipv4
        - ICMP, any ipv6
    - outbound rules:
        - all traffic, any ipv4
        - all traffic, any ipv6
3. Create IAM role `ecsTaskExecutionRole`
    - [guide in docs](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/task_execution_IAM_role.html>
    - IAM -> Create Role
        - entity type: AWS service
        - service or use case: Elastic Container Service
        - use case: Elastic Container Service Task
        - add permission "AmazonECSTaskExecutionRolePolicy"
        - role name: "ecsTaskExecutionRole"
4. Create a task definition
    - ECS -> Task Definitions -> Create
    - name: "production-task-definition"
    - launch type: fargate
    - task role: none
    - task execution role: ecsTaskExecutionRole (from step 2)
    - container 1
        - name: "nginx"
        - image uri: copy it from the ECR page ("381492078014.dkr.ecr.us-east-2.amazonaws.com/nginx:latest")
        - port mappings: 80, 443
5. Create an ECS cluster
    - ECS -> Clusters -> Create
    - name: "production-cluster"
    - choose Fargate
6. Create a service
    - "production-cluster" -> services -> create
    - capacity provider strategy
    - use fargate
    - application type: service
    - task definition: "production-task-definition"
    - service name: production-service
    - type: replica
    - networking
        - vpc: from step 1
        - enable all subnets
        - security group: "permissive_internet"
7. Fix auto-deployment
    - edit "github-actions" role
        - add inline custom policy
        - ARN: reference the production-service
        - permission: "ecr:UpdateService"
8. EC2 -> Create Load balancer
    - network load balancer
    - 




# ATTEMPT 2
just going to write down everything I do
1. Create IAM user for github-actions
    - don't give console access
    - add permissions manually
    - add AmazonEC2ContainerRegistryFullAccess
2. Give GitHub access to this user
    - create access key for the user
    - select 'third party service' and ignore warning - create anyway (just cuz I don't wanna deal with more complicated options yet)
    - description: Used by GitHub Actions to push images during CI.
    - repository -> settings -> Secrets and variables -> Actions -> create secrets
        - AWS_ECR_SECRET_ACCESS_KEY: copy/paste the "secret access key" from aws
        - AWS_ECR_ACCESS_KEY_ID: copy/paste the "access key" from aws
3. Create a (private) ECR repository on AWS for each of the images that you want to launch in ECS
4. Create or update a GitHub Actions workflow to build your images and push them to ECR.
    - see example: [build.yml](.github/workflows/build.yml)
5. Push some code and verify that the build and push succeeds. Check that a new `latest` image is available in the AWS console.
