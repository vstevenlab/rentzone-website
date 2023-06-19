# Deployment Pipeline GitHub Actions Workflow

This GitHub Actions workflow automates the deployment pipeline for this project. It consists of several jobs that execute based on specific conditions and dependencies. Below is a description of each job and its purpose:

## Job 1: `configure_aws_credentials`

This job configures the AWS credentials required for subsequent steps. It uses the `aws-actions/configure-aws-credentials` action to set up the AWS access key, secret access key, and region.

## Job 2: `deploy_aws_infrastructure`

This job builds the AWS infrastructure using Terraform. It checks out the repository, sets up Terraform, initializes it, and then apply or destroy the infrastructure based on the value of the `TERRAFORM_ACTION` environment variable. After applying the action, it retrieves various output values from Terraform and stores them as environment variables using `echo` and `grep`. These output values can be used in subsequent steps.

## Job 3: `create_ecr_repository`

This job creates an Elastic Container Registry (ECR) repository. It checks if the repository already exists using the `aws ecr describe-repositories` command and stores the result in the `repo_name` environment variable. If the repository doesn't exist, it creates a new one using `aws ecr create-repository`.

## Job 4: `start_runner`

This job starts a self-hosted EC2 runner for GitHub Actions. It checks if there is already a running EC2 runner by querying the running instances using `aws ec2 describe-instances`. If no instances are found, it starts a new EC2 runner using the `machulav/ec2-github-runner` action. The runner is started with specific configurations, such as the EC2 image ID, instance type, subnet ID, and security group ID.

## Job 5: `build_and_push_image`

This job builds a Docker image using the checked-out repository and pushes it to the ECR repository created earlier. It logs in to Amazon ECR using the `aws-actions/amazon-ecr-login` action, builds the Docker image with specific build arguments and tags, and pushes it to the ECR repository.

## Job 6: `export_env_variables`

This job creates an environment file containing required environment variable values and exports it to an S3 bucket. It creates the environment file with various environment variable values and their corresponding secrets. Then, it uploads the file to an S3 bucket using the `aws s3 cp` command.

## Job 7: `migrate_data`

This job migrates data into an RDS database using Flyway. It depends on the successful execution of the `deploy_aws_infrastructure`, `start_runner`, and `build_and_push_image` jobs.

## Job 8: `stop_runner`

This job stops the self-hosted EC2 runner that was started earlier. It has dependencies on multiple jobs, including `configure_aws_credentials`, `deploy_aws_infrastructure`, `start_runner`, `build_and_push_image`, `export_env_variables`, and `migrate_data`. It only runs if the `terraform_action` output from the `deploy_aws_infrastructure` job is not set to 'destroy'. It uses the `machulav/ec2-github-runner` action to stop the EC2 runner by specifying the runner's label and EC2 instance ID.

## Job 9: `create_td_revision`

This job creates a new revision of the ECS task definition. It depends on several jobs, including `configure_aws_credentials`, `deploy_aws_infrastructure`, `create_ecr_repository`, `start_runner`, `build_and_push_image`, `export_env_variables`,

## Job 10: `restart_ecs_service`

This job restarts the ECS Fargate service. It depends on multiple jobs, including `configure_aws_credentials`, `deploy_aws_infrastructure`, `create_ecr_repository`, `start_runner`, `build_and_push_image`, `export_env_variables`, `migrate_data`, `stop_runner`, and `create_td_revision`. It runs if the `terraform_action` output from the `deploy_aws_infrastructure` job is not set to 'destroy'. 

Steps:
1. Update ECS Service: This step updates the ECS service using the `aws ecs update-service` command. It specifies the ECS cluster name, service name, and the new task definition revision obtained from the `create_td_revision` job.
2. Wait for ECS service to become stable: This step waits for the ECS service to become stable using the `aws ecs wait services-stable` command. It ensures that the service is fully deployed and running.


