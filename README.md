AWS CloudFormation Fundamentals — Infrastructure as Code on AWS
A hands-on project documenting my journey learning AWS CloudFormation, from template fundamentals to deploying, troubleshooting, and improving an EC2-based web server stack.
Project focus: Learn how CloudFormation templates describe AWS infrastructure, how stacks are created and updated, how resources are connected, and how to troubleshoot failures instead of treating a successful stack deployment as proof that every bootstrap command worked.

Project overview
This project covered the core concepts and practical exercises needed to begin using AWS CloudFormation:
- CloudFormation templates and stacks
- YAML/JSON template structure and conversion
- Intrinsic functions, mappings, outputs, and pseudo parameters
- Dynamic AMI selection using AWS Systems Manager public parameters
- EC2 provisioning and User Data bootstrapping
- Security groups, EC2 instances, Elastic IPs, and resource references
- Systems Manager Session Manager access
- CloudFormation helper scripts (cfn-init, cfn-hup, and cfn-signal)
- Stack lifecycle management and troubleshooting failed deployments
- Template linting and testing tools
- Practical debugging of a web server that was created successfully but did not serve the expected page
This README intentionally documents both the learning exercises and the troubleshooting we performed. It does not claim that every optional lab or template was deployed successfully end to end.
1. CloudFormation foundations
Templates and stacks
- A template is the YAML or JSON definition of the infrastructure and configuration CloudFormation should create.
- A stack is the collection of AWS resources managed from a template.
- Reviewed the basic template structure, including Parameters, Mappings, Conditions, Resources, and Outputs.
- Practised using resource logical IDs and references to connect resources rather than hard-coding identifiers.
- Reviewed how stack creation, updates, rollback, and deletion affect resources.
Intrinsic functions and template values
Worked with the purpose and use of common intrinsic functions, including:
- !Ref — reference a parameter or resource and obtain its documented return value.
- !GetAtt — retrieve a resource attribute.
- !Sub — substitute parameters, resource attributes, and pseudo parameters into strings.
- !Join — join values into a string.
- !FindInMap — look up a value from a mapping.
- !GetAZs and !Select — work with Availability Zones and select a value.
- !ImportValue — consume an exported value from another stack when the export/import relationship is configured.
Also reviewed pseudo parameters such as AWS::Region, AWS::AccountId, AWS::StackName, and AWS::Partition.
Mappings and outputs
- Reviewed mappings as a way to define static lookup values, such as environment- or Region-specific values.
- Used outputs to expose useful information from a stack, such as a public IP address or a web URL.
- Considered how outputs can make a deployment easier to verify and how exported outputs can be consumed by other stacks.
YAML and JSON
- Reviewed that CloudFormation supports both YAML and JSON templates.
- Used/considered cfn-flip for converting templates between YAML and JSON.
- Used cfn-lint to check template structure and catch common template issues before deployment.
2. EC2 web server stack
A key practical exercise was a CloudFormation stack that created an EC2-based web server and related networking resources.

The template and debugging work involved:
- An EC2 instance (WebServerInstance)
- A security group (WebServerSecurityGroup)
- An Elastic IP associated with the instance
- Outputs for deployment information
- An Amazon Linux 2023 AMI selected through an SSM public parameter
- EC2 User Data intended to install and start a web server and publish a PHP page
- An IAM instance role/profile for Systems Manager access in the helper-script version
Linking the EC2 instance to its security group
We reviewed that declaring a security group as a separate resource does not automatically attach it to an EC2 instance. The instance needs a reference in its own Properties, for example:
Resources:
  WebServerInstance:
    Type: AWS::EC2::Instance
    Properties:
      # Other instance properties go here
      SecurityGroupIds:
        - !Ref WebServerSecurityGroup
The exact placement matters: SecurityGroupIds belongs inside the EC2 instance's Properties. The referenced security group must exist in a compatible VPC/network configuration.
Security-group review
The sample security group included inbound rules for HTTP on port 80 and SSH on port 22, with 0.0.0.0/0 used in the initial broad rule configuration. We discussed that exposing SSH to the entire internet is not a good production default. Restrict SSH to a trusted source IP/CIDR, or prefer Systems Manager Session Manager when configured.
3. AMI selection and multi-Region portability
The project reviewed using the public AWS Systems Manager parameter for the latest Amazon Linux 2023 AMI rather than hard-coding an AMI ID:
ImageId: '{{resolve:ssm:/aws/service/ami-amazon-linux-latest/al2023-ami-kernel-default-x86_64}}'
This helps a template select the appropriate published AMI value for the deployment Region. Always confirm that the parameter path and AMI architecture match the instance type and Region you intend to use.
4. User Data bootstrapping: what we tested and learned
The EC2 User Data script was intended to install Apache (httpd), PHP, and supporting tools; start and enable the web server; and create a PHP page that displayed instance metadata.

The actual troubleshooting was an important part of the project.
What happened
- CloudFormation created the EC2 instance, but the website did not respond as expected.
- The instance was reachable through Systems Manager and its status showed as Online.
- A request to the public IP 3.24.251.13 failed with connection refused on HTTP port 80.
- We inspected /var/log/cloud-init-output.log and found that the package-install command attempted to install curl while Amazon Linux 2023 already had curl-minimal.
- The package manager reported a conflict between curl and curl-minimal. Because the shell script stopped at that failed command, the later Apache/PHP setup and page creation commands did not run.
- Apache and PHP were then installed manually, and Apache was enabled/started.
- A request to http://localhost/ returned HTTP 403 and the default Apache “It works!” content rather than the intended PHP page. This was consistent with the custom page not having been created by the interrupted User Data script.
Lessons learned
1. A successful stack does not prove that User Data completed successfully. CloudFormation can create an EC2 instance even if a bootstrap command inside it fails.
2. Read the instance logs. /var/log/cloud-init-output.log is a useful first stop for User Data output; cloud-init status --long can also help assess cloud-init execution.
3. Avoid unnecessary package conflicts. On Amazon Linux 2023, check whether a package such as curl-minimal is already installed before asking dnf to install the full curl package.
4. Check each layer independently. Verify package installation, service state, listening ports, local HTTP response, security-group rules, and external connectivity separately.
5. Confirm the intended file exists. An Apache default page or 403 response does not establish that a custom PHP page was successfully deployed.
6. Make bootstrap scripts easier to diagnose. Use clear logging, fail-fast behavior deliberately, and validate the final service/page state before considering the bootstrap successful.
Useful commands from the troubleshooting process:
# Inspect User Data/cloud-init output
sudo tail -n 200 /var/log/cloud-init-output.log

# Check cloud-init's status
sudo cloud-init status --long

# Check Apache service status
sudo systemctl status httpd --no-pager

# Check whether Apache is listening on HTTP
sudo ss -lntp | grep ':80'

# Test the local web server
curl -i http://localhost/

# Inspect the default document root
ls -la /var/www/html/
5. Systems Manager Session Manager
We used Systems Manager access to troubleshoot the instance without relying solely on SSH. This provided a way to inspect cloud-init logs and repair the web server after the initial bootstrap failure.

For Session Manager to work, the instance needs the appropriate instance role/permissions, the SSM Agent, and network access to the Systems Manager endpoints. An instance appearing as Online is useful evidence that Session Manager is available, but it does not prove that the web application itself is healthy.
6. CloudFormation helper scripts
We also reviewed a more structured bootstrap pattern using CloudFormation helper scripts:
- cfn-init — applies configuration metadata, including packages, files, commands, and services.
- cfn-hup — monitors CloudFormation metadata and can run hooks when metadata changes, when configured.
- cfn-signal — reports success or failure to CloudFormation, commonly used with a CreationPolicy/resource signal requirement.
The helper-script template work included an EC2 instance, an SSM-capable IAM role/profile, a security group, an Elastic IP, an Amazon Linux 2023 AMI parameter, and stack outputs. It was structured as a learning template; it should be validated against the target account, Region, VPC, AMI, and helper-script/package compatibility before being treated as production-ready.
Why signals matter
Without a signal-based readiness check, CloudFormation may finish creating an EC2 resource before the application is fully configured. A CreationPolicy combined with cfn-signal can make stack creation wait for the bootstrap result and fail if the expected signal is not received in time.
7. Template validation and testing tools
cfn-lint
- Installed/used cfn-lint version 1.58.0.
- Used linting as a pre-deployment check for template structure and common CloudFormation errors.
- Linting helps catch issues, but it does not prove that runtime commands, permissions, network access, or application health will work.
Example:
cfn-lint template.yaml
TaskCat and the S3 permissions boundary
- Installed/used TaskCat version 0.9.58 as a template testing tool.
- During testing, TaskCat encountered an AWS Organizations Service Control Policy (SCP) that explicitly denied s3:CreateBucket.
- This was an account-level permissions restriction, not something that should be bypassed by changing the template or attempting to evade the organization policy.
- The correct next step is to ask the AWS account/organization administrator whether an approved test bucket or a permitted testing workflow is available, or to run the test in an authorized sandbox/account with the required permissions.
This was a useful reminder that a locally valid template and an installed test tool do not override AWS identity permissions or organization-level controls.
8. Stack lifecycle and rollback troubleshooting
We reviewed a common CloudFormation lifecycle issue: a stack can enter ROLLBACK_COMPLETE after a failed initial creation. In that state, a new create operation using the same stack name generally cannot proceed as though the stack does not exist.

The practical recovery sequence for a failed initial stack is usually:
1. Inspect the stack Events and identify the first underlying resource failure.
2. Fix the root cause before trying again.
3. Delete the failed stack if it is in a state that requires deletion before recreation.
4. Wait for deletion to finish.
5. Re-create the stack with the corrected template.
Example AWS CLI commands:
aws cloudformation describe-stacks \
  --stack-name demo \
  --region ap-southeast-2

aws cloudformation describe-stack-events \
  --stack-name demo \
  --region ap-southeast-2

aws cloudformation delete-stack \
  --stack-name demo \
  --region ap-southeast-2

aws cloudformation wait stack-delete-complete \
  --stack-name demo \
  --region ap-southeast-2

aws cloudformation create-stack \
  --stack-name demo \
  --template-body file://template.yaml \
  --capabilities CAPABILITY_IAM \
  --region ap-southeast-2
Only include --capabilities CAPABILITY_IAM when the template creates or modifies IAM resources that require acknowledgement. Review the actual template and stack state before running lifecycle commands, and do not delete a stack that contains resources you need to preserve.
9. AWS CLI environment used
The exercises used an AWS CLI profile named cloudformation-stack-user and the ap-southeast-2 Region (Sydney).

Examples:
aws sts get-caller-identity --profile cloudformation-stack-user

aws cloudformation describe-stacks \
  --stack-name demo \
  --profile cloudformation-stack-user \
  --region ap-southeast-2
The AWS account and instance identifiers are intentionally not published in this README. Before sharing terminal screenshots or logs publicly, redact account IDs, public IPs, private IPs, ARNs, access details, and any other environment-specific identifiers.
10. Troubleshooting checklist
When a CloudFormation deployment fails or a deployed application is unhealthy, investigate in layers:
1. Template validation: run cfn-lint and review YAML indentation, property names, and resource types.
2. CloudFormation events: find the first resource that failed and read its detailed status reason.
3. Permissions: check the active identity, IAM permissions, and any AWS Organizations SCP restrictions.
4. Resource wiring: confirm references such as !Ref WebServerSecurityGroup are placed under the correct resource properties.
5. Network configuration: confirm VPC/subnet compatibility, inbound/outbound rules, routes, and public addressing as applicable.
6. Bootstrap logs: inspect /var/log/cloud-init-output.log and cloud-init status --long.
7. Service health: check systemctl, listening sockets, local HTTP responses, and expected files.
8. Stack state: account for ROLLBACK_COMPLETE and other terminal states before retrying.
9. Runtime verification: test the actual application endpoint; do not rely only on the CloudFormation stack status.
10. Cleanup: delete lab resources that are no longer needed and confirm deletion completed.
11. Key takeaways
- CloudFormation provides repeatable infrastructure deployment, but the template, AWS permissions, network configuration, and runtime bootstrap all need to be correct.
- Intrinsic functions and outputs make templates reusable and easier to connect.
- Dynamic SSM AMI parameters reduce hard-coded Region-specific AMI IDs.
- User Data and cloud-init logs are essential debugging tools.
- Resource references must be explicitly wired in the right place.
- cfn-init and cfn-signal can make configuration and readiness checks more structured.
- cfn-lint is valuable before deployment; integration testing still needs suitable AWS permissions.
- AWS Organizations SCPs can block an operation even when a user has relevant IAM permissions.
- Stack events and lifecycle states should be understood before retrying or deleting resources.
- Keep public-facing security rules minimal and avoid unrestricted SSH access.
12. Suggested next steps
- Refactor the EC2 User Data script to avoid the curl/curl-minimal conflict and make bootstrap failures obvious.
- Add a CreationPolicy and cfn-signal so the stack can validate bootstrap completion.
- Test the template in an approved sandbox with the required permissions.
- Add parameters for environment, instance type, and trusted SSH CIDR instead of hard-coding values.
- Use SSM Session Manager where possible instead of exposing SSH.
- Add a CloudFormation change set workflow before applying updates.
- Practise stack outputs and exports/imports with a small multi-stack example.
- Explore drift detection and CloudFormation Guard after becoming comfortable with the fundamentals.
LinkedIn project title
AWS CloudFormation Fundamentals: Infrastructure as Code, EC2 Deployment & Troubleshooting
LinkedIn project description
Completed a hands-on AWS CloudFormation fundamentals project covering template anatomy, stacks, intrinsic functions, mappings, outputs, pseudo parameters, dynamic Amazon Linux 2023 AMI selection with SSM public parameters, EC2 web-server provisioning, security-group references, Elastic IPs, Session Manager, User Data, and CloudFormation helper scripts. I also practised template linting and testing with cfn-lint and TaskCat, and documented real troubleshooting scenarios involving a curl/curl-minimal package conflict, incomplete User Data execution, Apache returning HTTP 403 instead of the expected PHP page, stack rollback states, and an AWS Organizations SCP denying s3:CreateBucket. The project reinforced the importance of checking CloudFormation events, cloud-init logs, IAM/SCP permissions, resource references, and application health—not just whether a stack was created.
Hashtags
#AWS #AWSCloudFormation #InfrastructureAsCode #AWSCloud #DevOps #CloudEngineering #EC2 #SystemsManager #CloudSecurity #LearningInPublic
