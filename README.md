# Provisioning AWS Infrastructure Using GitHub Actions and CloudFormation – A GitOps Approach

In the evolving world of DevOps and infrastructure automation, GitOps has emerged as a powerful paradigm for managing infrastructure through code stored in Git repositories. We've seen GitOps workflows using tools like GitSync, AWS CodePipeline, Jenkins, and GitLab CI/CD. Now, let’s explore how we can achieve the same with GitHub Actions — GitHub’s native CI/CD tool.

In this post, we will walk through provisioning AWS infrastructure (a simple web server) across three environments — development, staging, and production — using AWS CloudFormation templates managed and deployed through GitHub Actions.

### Why GitHub Actions?
GitHub Actions offers a seamless way to automate tasks within the software development lifecycle. If your source code already lives in GitHub, you get these key benefits:

- Tight integration: CI/CD pipelines exist alongside your code and issues.
- Event-driven: Automatically trigger deployments on push, PR, or schedule.
- Secure secrets management: Store AWS credentials securely.
- Cost-effective: GitHub Actions comes with free minutes for public repos and reasonable limits for private ones.
- Scalable and Extensible: You can define complex workflows with conditional steps, environments, and matrix builds.

GitHub Actions acts as the bridge between your Git repository and AWS, enabling true GitOps workflows.


### Architecture Overview
Our architecture consists of:

![alt text](/images/architecture.png)

A nested CloudFormation stack setup with two layers:
- Network Stack: Provisions a VPC, subnets, and related networking components.
    - Compute Stack: Provisions EC2 instances (a web server) inside the VPC.
    - Infrastructure code (templates) for three environments — development, staging, and production.
- A GitHub Actions workflow that:
    - Creates an S3 bucket (if not exists)
    - Uploads CloudFormation templates
    - Lints and validates templates
    - Deploys the stacks based on environment


### Step 1: Prerequisites
To allow GitHub Actions to deploy resources on AWS, you'll need to:

Create a programmatic IAM user in AWS (for simplicity, we’ll use an admin user, but least privilege is recommended for production).

Add the following secrets in your GitHub repository under Settings → Secrets and variables → Actions:

- `AWS_ACCESS_KEY_ID` - Secret
- `AWS_SECRET_ACCESS_KEY` - Secret
- `AWS_REGION` - Variable

Ensure that secrets are protected (only accessible to protected branches) and masked to avoid printing them in logs.

![alt text](/images/secrets.png)

![alt text](/images/variables.png)

### Step 2: CloudFormation Template Structure
Organize your repo with three environment folders:

```
├───.github
│   └───workflows
│           aws-cfn-deploy.yml
└───infrastructure
    │   create-cfn-template-bucket.yaml
    ├───development
    │       compute.yaml
    │       network.yaml
    │       root.yaml
    ├───production
    │       compute.yaml
    │       network.yaml
    │       root.yaml
    └───staging
            compute.yaml
            network.yaml
            root.yaml
```

Each folder contains:

- `network.yaml`: Defines VPC, subnets, Internet gateway, route tables, etc.
- `compute.yaml`: Defines EC2 instance(s), security groups, and any web server bootstrap logic (like user data to install Apache/HTTPD).
- `root.yaml`: Defines template includes network.yaml and compute.yaml as nested stacks.

These templates are modular and reusable using nested stacks.

### Step 3: GitHub Actions Workflow (.github/workflows/aws-cfn-deploy.yml)
Here’s a sample GitHub Actions pipeline:

- `create_bucket` - Creates an S3 bucket to store templates
- `copy_templates` - Uploads CFN templates to S3
- `lint_templates` - Lints CFN templates using cfn-lint
- `validate_templates`	Validates S3-hosted templates via aws cloudformation validate-template
- `deploy` - Deploys stacks using a matrix across dev, staging, and prod environments. 
- `deploy_production` - Conditional logic allows deploying production only via manual trigger (workflow_dispatch) to avoid accidental deployments on push.

```yml
name: AWS CloudFormation CI/CD using GitHub Actions

on:
  push:
    branches:
      - main
  workflow_dispatch:

env:
  AWS_REGION: us-east-1
  BUCKET_NAME: ct-cfn-files-for-stack-github-actions

jobs:

  create_bucket:
    name: Create S3 Bucket
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v3

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v2
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ${{ env.AWS_REGION }}

      - name: Deploy bucket stack
        run: |
          aws cloudformation deploy \
            --stack-name cfn-template-bucket \
            --template-file infrastructure/create-cfn-template-bucket.yaml \
            --region $AWS_REGION \
            --capabilities CAPABILITY_NAMED_IAM

  copy_templates:
    name: Upload Templates to S3
    needs: create_bucket
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v2
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ${{ env.AWS_REGION }}

      - name: Copy templates to S3
        run: |
          aws s3 cp infrastructure/ s3://${BUCKET_NAME}/infrastructure/ --recursive

  lint_templates:
    name: Lint CloudFormation Templates
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3

      - name: Set up Python
        uses: actions/setup-python@v4
        with:
          python-version: '3.x'

      - name: Install cfn-lint
        run: pip install cfn-lint

      - name: Run cfn-lint
        run: |
          ERR=0
          for file in $(find ./infrastructure -type f \( -iname "*.yaml" -o -iname "*.yml" \)); do
            cfn-lint "$file" || ERR=1
          done
          if [ "$ERR" -eq "1" ]; then
            exit 1
          fi

  validate_templates:
    name: Validate Templates
    needs: [copy_templates]
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v2
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ${{ env.AWS_REGION }}

      - name: Validate all templates
        run: |
          for env in development staging production; do
            aws cloudformation validate-template \
              --template-url https://${BUCKET_NAME}.s3.${AWS_REGION}.amazonaws.com/infrastructure/$env/root.yaml
          done

  deploy:
    name: Deploy Stacks
    needs: [validate_templates]
    runs-on: ubuntu-latest
    strategy:
      matrix:
        env: [development, staging]
    environment:
      name: ${{ matrix.env }}
      url: https://console.aws.amazon.com/cloudformation/home?region=us-east-1
    steps:
      - uses: actions/checkout@v3

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v2
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ${{ env.AWS_REGION }}

      - name: Deploy CloudFormation Stack
        run: |
          echo "Deploying to ${{ matrix.env }}..."
          aws cloudformation create-stack \
            --template-url https://${BUCKET_NAME}.s3.${AWS_REGION}.amazonaws.com/infrastructure/${{ matrix.env }}/root.yaml \
            --stack-name Deploy${{ matrix.env }}Stack \
            --parameters ParameterKey=Environment,ParameterValue=${{ matrix.env }} \
            --capabilities CAPABILITY_NAMED_IAM

      - name: Wait for Stack Completion
        run: |
          aws cloudformation wait stack-create-complete --stack-name Deploy${{ matrix.env }}Stack

  deploy_production:
    name: Deploy Production Stack
    needs: [validate_templates]
    runs-on: ubuntu-latest
    environment:
      name: production
      url: https://console.aws.amazon.com/cloudformation/home?region=us-east-1
    steps:
      - uses: actions/checkout@v3

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v2
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: us-east-1

      - name: Deploy Production Stack
        run: |
          aws cloudformation create-stack \
            --template-url https://${{ env.BUCKET_NAME }}.s3.${{ env.AWS_REGION }}.amazonaws.com/infrastructure/production/root.yaml \
            --stack-name DeployProductionStack \
            --parameters ParameterKey=Environment,ParameterValue=production \
            --capabilities CAPABILITY_NAMED_IAM

      - name: Wait for Stack Completion
        run: |
          aws cloudformation wait stack-create-complete --stack-name DeployProductionStack
```

In this yml file, `push` to `main`: Automatically triggers the workflow whenever someone pushes code to the main branch.

`workflow_dispatch`: Enables manual trigger via GitHub UI (useful for production environments or reruns).

![alt text](/images/environments.png)

### Step 4: Git Push to repo to see automated GitHub Actions Pipeline running
Git push the components to GitHub Repo
![alt text](/images/pipeline_1.png)

![alt text](/images/pipeline_2.png)

![alt text](/images/pipeline_3_create_bucket_stack.png)

Development and Staging environments getting provisioned whereas production environment stage awaiting manual approval.

![alt text](/images/pipeline_4.png)

![alt text](/images/pipeline_5_dev_stage_stacks.png)

Manual approval provided:
![alt text](/images/pipeline_6_manual_approval.png)

![alt text](/images/pipeline_7_prod_stage.png)

![alt text](/images/pipeline_8_prod_stack.png)

![alt text](/images/web_servers_ec2.png)

### Cleanup
Don’t forget to delete AWS resources to avoid unexpected costs, and also delete the S3 bucket if no longer needed. Use aws cloudformation delete-stack --stack-name for each stack.

### Conclusion
GitHub Actions simplifies infrastructure provisioning on AWS using CloudFormation and GitOps practices. With environment-specific templates, secure credentials management, and declarative pipelines, it brings modern DevOps practices closer to your infrastructure-as-code workflows. This approach ensures consistency, traceability, and repeatability across environments.

Whether you’re provisioning a small dev server or managing production-grade infrastructure, GitHub Actions provides a scalable and secure automation pipeline, fully integrated with your GitHub repository.

### References
GitHub Repo: https://github.com/chinmayto/cloudformation-gitops-with-github-actions