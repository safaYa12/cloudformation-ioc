# AWS CloudFormation Fundamentals: Infrastructure as Code, Deployment & Troubleshooting

## Project Overview

This project documents my hands-on learning journey with **AWS CloudFormation**, from understanding the fundamentals of Infrastructure as Code (IaC) to deploying AWS resources, configuring an EC2 web server, and troubleshooting real deployment issues.

The project covered CloudFormation templates, stacks, intrinsic functions, mappings, outputs, pseudo parameters, dynamic AMI selection, EC2 provisioning, User Data, Systems Manager Session Manager, CloudFormation helper scripts, template validation, and stack lifecycle management.

Along with learning the core concepts, I worked through practical issues involving failed bootstrap scripts, Apache web server responses, package conflicts, resource references, stack rollback states, and AWS Organizations Service Control Policies (SCPs).

## Project Objectives

- Understand the fundamentals of AWS CloudFormation and Infrastructure as Code.
- Create and manage CloudFormation templates and stacks.
- Use intrinsic functions to connect resources and make templates reusable.
- Provision EC2 instances and configure web servers using User Data.
- Configure security groups, Elastic IPs, and IAM permissions.
- Use AWS Systems Manager Session Manager to access and troubleshoot instances.
- Understand CloudFormation helper scripts and resource creation signals.
- Validate templates using `cfn-lint` and explore testing with TaskCat.
- Troubleshoot CloudFormation deployment and application-level failures.
- Apply security best practices when deploying AWS infrastructure.

## Technologies and Tools

- **AWS CloudFormation** — Infrastructure as Code
- **Amazon EC2** — Virtual server provisioning
- **Amazon Linux 2023** — Operating system for the web server
- **AWS Systems Manager Session Manager** — Instance access and troubleshooting
- **AWS IAM** — Roles and permissions
- **Amazon VPC and Security Groups** — Network configuration
- **Elastic IP** — Public IPv4 addressing
- **AWS CLI** — Command-line management of AWS resources
- **cfn-lint** — CloudFormation template validation
- **TaskCat** — CloudFormation template testing
- **cfn-flip** — YAML and JSON template conversion
- **Apache HTTP Server and PHP** — Web server configuration

## 1. CloudFormation Fundamentals

I learned how CloudFormation uses templates to define AWS infrastructure and how stacks manage the resources created from those templates.

### Key concepts covered

- CloudFormation templates and stacks
- YAML and JSON template formats
- Template sections: `Parameters`, `Mappings`, `Conditions`, `Resources`, and `Outputs`
- Logical resource IDs and resource properties
- Stack creation, updates, rollback, and deletion
- Resource dependencies and references
- Infrastructure provisioning through declarative templates

## 2. Intrinsic Functions

I explored CloudFormation intrinsic functions to reference resources, retrieve attributes, construct strings, and make templates more reusable.

Functions covered:

- `!Ref` — References a parameter or resource.
- `!GetAtt` — Retrieves an attribute from a resource.
- `!Sub` — Substitutes variables into strings.
- `!Join` — Combines values into a string.
- `!FindInMap` — Retrieves values from mappings.
- `!GetAZs` — Retrieves Availability Zones for a Region.
- `!Select` — Selects an item from a list.
- `!ImportValue` — Imports an exported value from another stack.

I also reviewed pseudo parameters such as:

- `AWS::Region`
- `AWS::AccountId`
- `AWS::StackName`
- `AWS::Partition`

These concepts help reduce hard-coded values and make CloudFormation templates easier to maintain.

## 3. Mappings, Outputs, and Template Reusability

I learned how mappings can store static lookup values, including environment-specific or Region-specific configuration.

I also worked with stack outputs to expose useful deployment information, such as public IP addresses and application URLs.

Topics covered:

- Defining and referencing mappings
- Exporting stack outputs
- Using outputs to retrieve deployment information
- Reusing values across stacks with exports and imports
- Reducing hard-coded values in templates

## 4. YAML and JSON Conversion

CloudFormation supports both YAML and JSON templates.

I explored `cfn-flip` as a utility for converting templates between these formats and `cfn-lint` for validating template syntax and identifying common CloudFormation issues.

Example:

```bash
cfn-lint template.yaml
```

Template validation is an important step before deployment, although linting alone cannot guarantee that AWS permissions, networking, bootstrap scripts, or application services will work correctly.

## 5. Dynamic AMI Selection Using AWS Systems Manager

I explored selecting the latest Amazon Linux 2023 AMI using an AWS Systems Manager public parameter rather than hard-coding an AMI ID.

Example:

```yaml
ImageId: '{{resolve:ssm:/aws/service/ami-amazon-linux-latest/al2023-ami-kernel-default-x86_64}}'
```

This approach helps templates select the appropriate published AMI for the deployment Region.

The AMI architecture, instance type, and parameter path must still be compatible with the target environment.

## 6. EC2 Web Server Deployment

A major practical exercise involved deploying an EC2-based web server through CloudFormation.

The template work included:

- An EC2 instance named `WebServerInstance`
- A security group named `WebServerSecurityGroup`
- An Elastic IP associated with the instance
- Stack outputs for deployment information
- An Amazon Linux 2023 AMI selected through an SSM public parameter
- User Data intended to install Apache, PHP, and supporting tools
- A PHP page intended to display instance metadata
- An IAM role/profile for Systems Manager access in the helper-script version

### Connecting the EC2 Instance to the Security Group

I learned that defining a security group as a separate CloudFormation resource does not automatically attach it to an EC2 instance. The instance must explicitly reference the security group in its properties.

Example:

```yaml
Resources:
  WebServerInstance:
    Type: AWS::EC2::Instance
    Properties:
      SecurityGroupIds:
        - !Ref WebServerSecurityGroup
```

This reference must be placed inside the EC2 instance's `Properties` section. The security group must also be compatible with the instance's VPC configuration.

### Security Group Review

The initial example included inbound rules for HTTP on port `80` and SSH on port `22`.

During the review, I identified that allowing SSH from `0.0.0.0/0` exposes the SSH service to the entire internet. A safer configuration restricts SSH to a trusted IP address or CIDR range, or uses Systems Manager Session Manager where possible.

## 7. User Data Bootstrapping and Troubleshooting

The EC2 User Data script was intended to install Apache, PHP, and supporting tools, start and enable Apache, and create a PHP page displaying instance metadata.

The deployment revealed an important lesson: **CloudFormation can successfully create an EC2 instance even when its User Data script fails.**

### Issue 1: HTTP Connection Refused

After the instance was created, accessing the web server through its public IP address resulted in a connection-refused error on port `80`.

The EC2 instance was accessible through Systems Manager, which allowed me to investigate the problem from inside the instance.

### Issue 2: Package Conflict on Amazon Linux 2023

While inspecting the cloud-init output, I found a package conflict involving `curl` and the preinstalled `curl-minimal` package.

The User Data script attempted to install `curl`, but the package manager reported a conflict with `curl-minimal`. Because the command failed, the subsequent bootstrap commands did not complete as intended.

As a result, the Apache/PHP setup and creation of the custom PHP page were not completed by the original script.

### Issue 3: Apache Returned HTTP 403

I manually installed Apache and PHP and enabled Apache.

A request to:

```bash
curl -i http://localhost/
```

returned an HTTP `403` response with the default Apache “It works!” content rather than the expected PHP page.

This was consistent with the custom PHP page not having been created by the interrupted User Data script.

### Commands Used During Troubleshooting

Inspect cloud-init output:

```bash
sudo tail -n 200 /var/log/cloud-init-output.log
```

Check cloud-init status:

```bash
sudo cloud-init status --long
```

Check Apache service status:

```bash
sudo systemctl status httpd --no-pager
```

Check whether Apache is listening on port 80:

```bash
sudo ss -lntp | grep ':80'
```

Test the local web server:

```bash
curl -i http://localhost/
```

Inspect the web document root:

```bash
ls -la /var/www/html/
```

### Lessons Learned

- A successful CloudFormation stack does not guarantee successful application bootstrapping.
- Cloud-init logs are essential when diagnosing User Data failures.
- Package conflicts can interrupt a bootstrap script before later commands execute.
- Apache service status, listening ports, local HTTP responses, and security-group rules should be checked separately.
- A default Apache page does not prove that the intended application has been deployed.
- User Data scripts should provide clear logs and deliberate failure handling.
- Application health should be verified independently of the CloudFormation stack status.

## 8. AWS Systems Manager Session Manager

I used Systems Manager Session Manager to access the EC2 instance and investigate the web server deployment without relying solely on SSH.

This allowed me to inspect cloud-init logs, check Apache, and perform manual troubleshooting after the initial User Data script failed.

For Session Manager to work, an instance needs the appropriate IAM role and permissions, a functioning SSM Agent, and network access to the relevant Systems Manager endpoints.

An instance showing as `Online` in Systems Manager confirms useful connectivity, but it does not prove that the web application is functioning correctly.

## 9. CloudFormation Helper Scripts

I also reviewed a more structured CloudFormation configuration approach using helper scripts.

### `cfn-init`

Used to apply configuration metadata, including package installations, files, commands, and service settings.

### `cfn-hup`

Monitors CloudFormation metadata and can execute configured hooks when metadata changes.

### `cfn-signal`

Sends a success or failure signal to CloudFormation. When used with a `CreationPolicy`, it can help ensure that a resource is not considered ready until its bootstrap process reports the expected result.

The helper-script template work included an EC2 instance, an IAM role/profile for Systems Manager, a security group, an Elastic IP, an Amazon Linux 2023 AMI parameter, and outputs.

This was a learning template and should be validated against the target account, Region, network configuration, and package/helper-script compatibility before production use.

### Why Resource Signals Matter

Without a readiness check, CloudFormation may finish creating an EC2 resource before the application has completed its setup.

A `CreationPolicy` combined with `cfn-signal` can make stack creation wait for bootstrap completion and fail if the expected signal is not received within the configured time.

## 10. Template Validation with cfn-lint

I installed and used `cfn-lint` version `1.58.0` to validate CloudFormation templates and identify common template issues before deployment.

Example:

```bash
cfn-lint template.yaml
```

Linting is useful for catching structural and property-level problems early. It does not replace deployment testing, permissions checks, or application-level health verification.

## 11. TaskCat Testing and AWS Organizations SCP Restrictions

I installed and explored TaskCat version `0.9.58` as a tool for testing CloudFormation templates.

During testing, TaskCat encountered an AWS Organizations Service Control Policy (SCP) that explicitly denied:

```text
s3:CreateBucket
```

This prevented the operation required by the test workflow.

The issue demonstrated that a template or testing tool cannot override an explicit organization-level restriction.

The appropriate next step is to consult the AWS organization or account administrator about an approved test bucket, a permitted testing workflow, or an authorized sandbox account with the required permissions.

I did not attempt to bypass the SCP.

### Key Takeaway

Successful template validation does not guarantee that a deployment or testing workflow has the AWS permissions it needs. IAM policies and AWS Organizations SCPs must also be considered.

## 12. CloudFormation Stack Lifecycle and Rollback Troubleshooting

I reviewed how CloudFormation handles failed stack creation and why a stack in `ROLLBACK_COMPLETE` may need to be deleted before the same stack name can be used for a fresh creation.

### Recovery Workflow

1. Inspect CloudFormation stack events.
2. Identify the first underlying resource failure.
3. Correct the template, permissions, or configuration issue.
4. Delete the failed stack if its state requires deletion before recreation.
5. Wait for deletion to complete.
6. Create the stack again with the corrected template.

Example AWS CLI commands:

```bash
aws cloudformation describe-stacks \
  --stack-name demo \
  --region ap-southeast-2
```

Inspect stack events:

```bash
aws cloudformation describe-stack-events \
  --stack-name demo \
  --region ap-southeast-2
```

Delete a failed lab stack when appropriate:

```bash
aws cloudformation delete-stack \
  --stack-name demo \
  --region ap-southeast-2
```

Wait for deletion:

```bash
aws cloudformation wait stack-delete-complete \
  --stack-name demo \
  --region ap-southeast-2
```

Create the stack again:

```bash
aws cloudformation create-stack \
  --stack-name demo \
  --template-body file://template.yaml \
  --region ap-southeast-2
```

If the template creates or modifies IAM resources that require acknowledgement, include the appropriate capability flag, such as `--capabilities CAPABILITY_IAM`.

Stack deletion should only be performed after confirming that the resources are safe to remove and that no required data or infrastructure will be lost.

## 13. AWS CLI Profile and Region

The exercises used the AWS CLI profile:

```text
cloudformation-stack-user
```

The Region used was:

```text
ap-southeast-2
```

Verify the active AWS identity:

```bash
aws sts get-caller-identity \
  --profile cloudformation-stack-user
```

Inspect a stack:

```bash
aws cloudformation describe-stacks \
  --stack-name demo \
  --profile cloudformation-stack-user \
  --region ap-southeast-2
```

Before publishing terminal screenshots or logs, sensitive and environment-specific information such as account IDs, IP addresses, ARNs, and access details should be redacted.

## 14. Complete Troubleshooting Checklist

When a CloudFormation deployment fails or an application does not work as expected, I now approach the issue layer by layer.

1. **Template validation:** Run `cfn-lint` and inspect YAML indentation, property names, and resource types.
2. **CloudFormation events:** Find the first failed resource and read the detailed status reason.
3. **Permissions:** Check IAM permissions, the active identity, and any AWS Organizations SCP restrictions.
4. **Resource references:** Confirm that references such as `!Ref WebServerSecurityGroup` are placed under the correct resource properties.
5. **Networking:** Verify VPC/subnet compatibility, security-group rules, routes, and public addressing.
6. **Bootstrap logs:** Inspect `/var/log/cloud-init-output.log` and run `cloud-init status --long`.
7. **Service health:** Check `systemctl`, listening sockets, local HTTP responses, and expected files.
8. **Stack state:** Understand states such as `ROLLBACK_COMPLETE` before attempting another deployment.
9. **Application verification:** Test the actual application endpoint instead of relying only on the stack status.
10. **Cleanup:** Remove unneeded lab resources and confirm deletion completed.

## 15. Key Takeaways

This project reinforced several important AWS and Infrastructure as Code concepts:

- CloudFormation enables repeatable infrastructure deployment, but templates, permissions, networking, and bootstrap commands must all be configured correctly.
- Intrinsic functions make templates more reusable and allow resources to reference one another.
- Dynamic SSM AMI parameters help avoid hard-coding Region-specific AMI IDs.
- User Data failures can occur even when an EC2 resource is successfully created.
- Cloud-init logs are essential for diagnosing bootstrap problems.
- Security groups must be explicitly associated with resources where required.
- `cfn-init` and `cfn-signal` can make configuration and readiness checks more structured.
- `cfn-lint` is useful before deployment, but integration testing still requires appropriate AWS permissions.
- AWS Organizations SCPs can deny actions even when a user has relevant IAM permissions.
- CloudFormation events and stack lifecycle states are important when recovering from failures.
- Application health must be verified independently from infrastructure creation.
- Restricting SSH access and using Session Manager where appropriate improves security.

## 16. Next Steps

The next improvements for this project are:

- Refactor User Data to avoid the `curl`/`curl-minimal` package conflict.
- Add clear bootstrap logging and final application health checks.
- Add a `CreationPolicy` and `cfn-signal` to validate initialization.
- Test in an approved sandbox with the required permissions.
- Parameterize the environment, instance type, and trusted SSH CIDR.
- Prefer Session Manager over unrestricted public SSH access.
- Practise CloudFormation change sets before applying stack updates.
- Build a small multi-stack example using exports and imports.
- Explore CloudFormation drift detection and CloudFormation Guard.

