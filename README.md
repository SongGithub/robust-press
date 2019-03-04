
# Robust Press
[![Build Status](https://travis-ci.org/SongGithub/robust-press.svg?branch=master)](https://travis-ci.org/SongGithub/robust-press)

> a Robust Wordpress application


## System design goals

- HA: EC2 instances are spreaded into all 3 Availability Zones in Sydney region, managed by ASG
- Secure:
  - restricts SSH 22 port to separate `jumpbox` access only
- Acknowledged:
  - cloudwatch log integrated
- Easily reproducible. Most infrastructure are coded as templates, which allows it to be recreated as a separate stack with minimal reconfiguration (to `envs/*.yaml`)
- Idempotent. The use of Cloudformation helps with this property
- Ease of deployment. Code for both application and infrastructre is automatically deployed through Travis machine user, once pushed to master branch.
- Simplicity.
- Secure: RDS root secrets are saved to the Parameter Store, and use AWS Cloudformation `retrieve` function to decrypt

## Initial setup

### RDS secret deployment
`bin/deploy_secrets dev/prod` will deploy secrets to the Parameter store

### CI user (IAM) setup (Manual deploy)

To create a CI user in AWS account, locally run following after authenticated to AWS as Admin:
- `bin/deploy_cfn ci dev`

Then please go to AWS console, manually create AccessKey. And note the key ID and secret. This is an one-off task
, so simplicity overcomes repeatability. Then follow [instructions](https://docs.travis-ci.com/user/encryption-keys/)

### Setup VPC (Manual deploy)
locally run following after authenticated to AWS:

- `bin/deploy_cfn infra dev`

- The above script will create a VPC containing 3 public subnets, 3 private subnets
accoss 3 AZ, as well EIP, RouteTables, and NAT

*Note: Infrastructure as code is a good idea but it doesn't mean to put every thing
into build pipeline. Because the pipeline is there to save human operators
from tedious tasks such as repetitive operations to implement incremental changes.
Hence some foundational tasks that are both rarely-happened and high-risk
should be excluded from repeating pipelines. So that no mistake could be
easily made to infrastruture due to a bad code commit. Examples include:
vpc,secrets,and hosted zone etc*

### use of Bastion
- prerequisite: VPC stack is created.
- manully create keypair `wp` under AwsConsole/EC2. This action will
downloade a file `wp.pem` to your default Download directory,
 for instance `~/Downloads`
- run `chmod 400 ~/Downloads/wp.pem`
- run `ssh-add ~/Downloads/wp.pem`
- configure envs/dev.yaml to your CIDR range.
- run `bin/deploy_cfn cfn bastion dev dev` to create CFN stack for bastion
- scale up the bastion ASG to 1
- find public IP of the bastion instance
- `ssh -A ec2-user@<the-bastion-ip>`

*Note: Please ensure Bastion instance count is 0 after use, also there is a scheduled action that will scale off the Bastion ASG by 6pm everyday for security reason( to prevent cases that operators forgot to do so )*

### EC2/ASG/ELB
- chosen ami: `ami-09b42976632b27e9b`. It is a standard free tier AMI that optimised for ECS
- `bin/deploy_cfn app dev <BUILD NUMBER>`
- Above script will create ASG for the EC2 instances accross all 3 AZ for high availability purpose,
as well as ELB that will do healh checks on instances


### DNS setup

[http://dev-wp.midu.com.au](http://dev-wp.midu.com.au) is the current URL for the dev WordPress website,
while prod has an URL [http://wordpress.midu.com.au](http://wordpress.midu.com.au)
- run `bin/deploy_cfn dns dev`. This will create a CNAME record pointing to ELB DNS.

## Assumptions
- When the build pipeline is running, none of the Cloudformation stacks should be in `IN_PROGRESS` status

## Short-commings/TO-DOs
- Permissions given to the CI user could be narrowed down. It has been given full-access to many types of resources.
- config file `wp-config.php` could be compiled dynamically with env specific settings such as URL.
