# aws-sysops-associate

## Table of Content

- [EC2 for SysOps](#ec2-for-sysops)
- [AMI](#ami)
- [Manage EC2 at scale](#manage-ec2-at-scale)
- [High availability and scalability](#high-availability-and-scalability)
- [Elastic Beanstalk](#elastic-beanstalk)
- [Cloudformation](#cloudformation)
- [Lambda](#lambda)
- [EC2 storage and data management](#ec2-storage-and-data-management)
- [S3](#s3)
- [S3 advanced](#s3-advanced)
- [S3 security](#s3-security)
- [Advanced storage solutions](#advanced-storage-solutions)
- [Cloudfront](#cloudfront)
- [Databases](#databases)
- [Monitoring and audit and performance](#monitoring-and-audit-and-performance)
- [Account management](#account-management)
- [Disaster recovery](#disaster-recovery)
- [Security and compliance](#security-and-compliance)
- [Identity](#identity)
- [VPC](#vpc)
- [Route 53](#route-53)
- [Other services](#other-services)


## EC2 for SysOps

- changing instance types (only for EBS-backed instances): stop the instance, apply change, then restart the instance
- enhanced networking:
  - EC2 enhanced networking(sr-iov): elastic network adapter(higher bandwidth)
  - Elastic fabric adapter: improved ENA, `only works for linux`, great for tightly coupled workloads
- placement groups:
  - cluster: cluster instances into a low-latency group in a AZ, if AZ fails, all instances will fail
  - spread: 7 instances per group per AZ (critical apps)
  - partition: up to 7 partitions per AZ, up to 100 instances per group
- shutdown behavior & termination protection
  - not from `aws console` or `aws cli`, but `$shutdown` from within the ec2 system, then stop(default) or terminate will be performed(two options)
  - termination protection `only protect against termination of aws console or aws cli`
- ec2 launch troublshooting
  - **InstanceLimitExceeded**: means that you have reached your limit of max number of vCPUs per region
    - Note: vCPU-based limits only apply to running On-Demand instances and Spot instances
  - **InsufficientInstanceCapacity**: means AWS does not have that enough On-Demand capacity in the particular AZ where the instance is launched.
  - **InstanceTerminates Immediately**: goes from pending to terminated
    - reach EBS volume limit
    - EBS snapshot is corrupt
    - root EBS is encrypted, but you have no permission to decrypt it
    - AMI is missing some required part
- ec2 ssh troubleshooting
  - make sure the permission code `400` for the private key file`.pem`
  - make sure the username for the OS is giving correctly when logging via ssh.
  - `connection time out`: SG/NACL is not configured correctly; check the route table; instance does not have a public ip; cpu load is high...
- ssh vs ec2 instance connect(one time public key)
- ec2 instance purchasing options
  - on-demand instance
  - reserved(1 or 3 years): reserved instances / convertible reserved instances
  - savings plans(1 or 3 years): commitment to an amount of usage
  - spot instances
  - dedicated hosts: compliance requirement or use your own existing server-bound software licenses
  - dedicated instances
  - capacity reservations: combine with reserved or saving plans
- ec2 spot instance requests
  - max spot price: otherwise choose to stop/terminate with a 2 min grace period
  - spot block: (1 - 6 hours) 
  - not for critical jobs or databases
  - cancel a spot request does not terminate instances. So first cancel a spot request, and then terminate instances
- spot fleets: set of Spot Instances + (optional) On-Demand Instances
  - try to meet the target capacity with price constraints
  - allow to automatically request spot instances with lowest price
  - strategies to allocate spot instances:
    - lowestPrices
    - diversified
    - capacityOptimized
    - priceCapacityOptimized
- bustable instances(T2/T3)
  - when a spike of load strikes, cpu can burst using `burst credits`, if all the credits are gone, then cpu will become `BAD`.
  - when a machine stops bursting, then credits are accumulated over time.
  - but if your instances consistently runs low on credits, maybe it is the time to change to a non-burstable instance
- T2/T3 unlimited: extra money, be careful
- Elastic IPs
  - do not pay for it when using it
  - pay for it when not using it
  - can have up to 5 elastic ip
  - should avoid using it
- cloudwatch metrics for ec2
  - aws provided metrics: basic monitoring(default), detail monitoring(pay), no RAM included (CPU, network, status check(instance status, system status, attached EBS status), disk)
  - custom metric 
- unified cloudwatch agent
  - procstat plugin: Collect metrics and monitor system utilization of individual processes. support both linux and windows
- status checks
  - system status checks: aws systems(software and hardware), (`personal health dashboard`), stop and start the instance
  - instance status checks: reboot the instance
  - EBS status checks: reboot the instance or replace EBS
- status checks - cloudwatch metric and recovery
  - cloudwatch alarm: action - recovery
  - auto scaling group
- ec2 hibernate
  - write RAM state into root EBS volume; the EBS volume must be encrypted
  - instance RAM: less than 150 GB
  - available for on-demand, reserved, spot
  - hibernate no more than 60 days



## AMI
## Manage EC2 at scale
## High availability and scalability
## Elastic Beanstalk
## Cloudformation
## Lambda
## EC2 storage and data management
## S3
## S3 advanced
## S3 security
## Advanced storage solutions
## Cloudfront
## Databases
## Monitoring and audit and performance
## Account management
## Disaster recovery
## Security and compliance
## Identity
## VPC
## Route 53
## Other services
