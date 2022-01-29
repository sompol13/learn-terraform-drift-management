## Manage Resource Drift
In this tutorial, you will detect and fix differences between your state file and real infrastructure. First, you will create an EC2 instance and security group with Terraform. Then, you will manually edit them via the AWS CLI. Next, you will identify and resolve the discrepancies between Terraform state and your infrastructure.

### Create infrastructure
- `ssh-keygen -t rsa -C "your_email@example.com" -f ./key`
- Open the `terraform.tfvars` file and edit the region to match your AWS CLI configuration.
- `terraform init`
- `terraform apply`

### Introduce drift
To introduce a change to your configuration outside the Terraform workflow, create a new security group with the AWS CLI and export that value as an environment variable.
- ` export SG_ID=$(aws ec2 create-security-group --group-name "sg_web" --description "allow 8080" --output text)`
- `aws ec2 authorize-security-group-ingress --group-name "sg_web" --protocol tcp --port 8080 --cidr 0.0.0.0/0`
- `aws ec2 modify-instance-attribute --instance-id $(terraform output -raw instance_id) --groups $SG_ID`

### Run a refresh-only plan
- Run `terraform plan -refresh-only` to determine the drift between your current state file and actual configuration.
- Apply these changes to make your state file match your real infrastructure, but not your Terraform configuration `terraform apply -refresh-only`.

### Add the security group to configuration
- Import the `sg_web` security group resource to your state file to bring it under Terraform management.
- Add the security group ID to your instance resource.

*Import the security group*
- Run `terraform import` to associate your resource definition with the security group created in the AWS CLI.
- `terraform import aws_security_group.sg_web $SG_ID`
- Import your security group rule.
- `terraform import aws_security_group_rule.sg_web "$SG_ID"_ingress_tcp_8080_8080_0.0.0.0/0`
- Run `terraform state list` to return the list of resources Terraform is managing, which now includes the imported resources.
- `terraform state list`

### Update your resources
- `terraform apply`

### Access the instance
- Confirm your instance allows SSH. Enter `yes` when prompted to connect to the instance.
- `ssh ubuntu@$(terraform output -raw public_ip) -i key`
- Confirm your instance allows port 8080 access.
- `curl $(terraform output -raw public_ip):8080`

### Reference
https://learn.hashicorp.com/tutorials/terraform/resource-drift