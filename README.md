# aws-sysops-associate

## Table of Content

- [EC2 for SysOps](#ec2-for-sysops)
- [AMI](#ami)
- [Manage EC2 at scale System manager](#manage-ec2-at-scale-system-manager)
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

- overview:
  - a customization of ec2: configure system, software
  - built for a specific region
  - launch ec2 instances from: public ami, custom ami, or aws marketplace ami
- ami process: start ec2 instance, customize it, stop it, and make an ami, then launch instances from the ami
- ami no-reboot option: by default, aws will shutdown the instance before taking an EBS snapshot and create an AMI. But we can choose no-reboot option, then the system buffer will not be sent to disk before the snapshot is created
- aws backup plans to create ami:
  - aws backup does not reboot the instance when taking ebs snapshot(no-reboot behavior)
  - since it does not stop the instance while taking snapshot, so it is not guaranteed the file system integrity
  - to maintain integrity, you need to provide `reboot` parameter while taking images( eventbridge + lambda + createImage api with reboot)
- cross-account amia sharing: sharing does not affect the ownership of the ami.
  - can only share amis without encrypted volumes or the volumes encrypted with CMK(you must share the CMK as well)
- cross-account ami copy: if you copy the ami shared with you, then you are the new owner of the copy
  - the owner of source ami must give read permission of the ebs snapshot which backs the ami
  - if the backing snapshot is encrypted, then the owner must share the key
- ec2 image builder
  - used to automate the creation, maintain,validate, test of ami
- ami in production
  - force users to only launch ec2 with pre-approved amis(tagged with specific tags) using iam policies(using condition to check tags)
  - combine with aws config to find non-compliant ec2 instances

 

## Manage EC2 at scale System manager

- overview: help manage ec2 and on-prem systems; automation; detection; work for windows and linux; integrated with cloudwatch metrics; integrated with aws config
- SSM agent: installed by default on aws linux ami; make sure the ec2 instances have a proper iam role to allow ssm actions
- aws Tags: a service that can add key-value pairs(tags) to other aws resources(like ec2), which can be used to group or identify resources, automation,... (better to have too many than too few)
- create, view, manage logical group of resources thanks to `Tags`
  - applications, different layers of app stack, prod vs dev env
  - regional service
  - ec2, s3, dynamodb, lambda...
- ssm document: like scripts, but written in json or yml
  - we define parameters, actions
  - also there are existing aws documents
- ssm run-command: execute ssm documents or just a command
  - run command across multiple resources (resource group)
  - rate control/error control
  - integrated with iam, cloudtrail
  - no need for ssh
  - the output can be seen in the console, or sent to s3 or cloudwatch logs
  - send notifications to sns
  - can be invoked using eventbridge
- ssm automation: Simplifies common maintenance and deployment tasks of EC2 instances and other AWS resources
  - Automation runbook: ssm document of type automation
  - pre-defined runbooks(aws) or create custom runbooks
  - can be triggered: console, cli, sdk, eventbridge, on a schedule using `maintenance window`, aws config for rules remediations
- ssm parameter store: secure storage for configuration and secret; notification with aws eventbridge; integrated with cloudformation
- ssm parameter store hierarchy
- ssm parameter store advanced
  - parameter policies: allow to apply a TTL
- ssm inventory
  - collect metadata from your managed instances(ec2/on-prem)
  - metadata: installed software, os driver, configurations, updates,...
  - view data in aws console, or store in s3 and query using athena and quicksight
  - specify the metadata collection interval
  - can create custom inventory
- ssm state manager
  - automate the process of keeping your managed instances (ec2/on-prem) in a state that you define
  - use cases: patch os/software updates on a schedule,...
  - `state manager association`: Defines the state that you want to maintain to your managed instances. Specify a schedule when this configuration is applied
  - Uses SSM Documents to create an Association (e.g., SSM Document to configure CW Agent)
- ssm patch manager: Automates the process of patching managed instances
  - Patch on-demand or on a schedule using Maintenance Windows
  - Scan instances and generate patch compliance report (missing patches)
  - Patch compliance report can be sent to S3
  - patch baseline: Defines which patches should and shouldn’t be installed on your instances
    - pre-define patch baseline: `AWS-RunPatchBaseline (SSM Document)` apply both operating system and application patches 
    - custom patch baseline
  - patch group: Associate a set of instances with a specific Patch Baseline; one patch group with one patch baseline
- ssm maintenance window: Defines a schedule for when to perform actions on your instances
- ssm session manager: Allows you to start a secure shell on your EC2 and on- premises servers. Does not need SSH access, bastion hosts, or SSH keys
  - session logs: sent to s3 or cloudwatch logs


## High availability and scalability

- scalability and high availability
  - scalability: means that an application / system can handle greater loads by adapting. (vertical scalability and horizontal scalability:elasticity)
  - vertical scalability: common for non-ditributed system, such as DB
  - horizontal scalability: means increasing the number of instances / systems for your application. Scaling for distributed system, very common for modern apps
  - high availability: usually goes hand in hand with `horizontal scaling`. Meaning running your application / system in at least 2 data centers (== Availability Zones) in order to survive a data center loss. The high availability can be passive/active.
- load balancing: forward traffic to multiple servers (e.g., EC2 instances) downstream
- why use load balancers: separate public traffic from private traffic; expose a single point of access; apply ssl termination(https);...
- elb: a managed load balancer
- elb: health check (/health) to check if downstream services are available
- elb: alb(layer 7, http,https,websocket), nlb(layer 4 , tcp,tls,udp),gwlb(layer3 network layer-- ip protocol)
- elb: security group
- alb: target group (asg, ecs tasks, lambda functions, ip addresses), health check are at the target group level
- alb: good to know: fixed hostname; the downstream services cannot see the true ip of the client, but we can see it in the header: **X-Forwarded-For**
- nlb: can handle millions of requests, has **one static ip per AZ** and support elastic ip; not in aws free tier
- nlb: target group: ec2 instances, ip addresses, alb, health check: support tcp, http and https
- gwlb: Deploy, scale, and manage a fleet of 3rd party network virtual appliances in AWS. Example: Firewalls, Intrusion Detection and Prevention Systems, Deep Packet Inspection Systems, payload manipulation, ...
- gwlb: uses `GENEVE` protocal on port 6081
- gwlb: target group: ec2 instances, ip addresses(must be private ip)
- sticy session(session affinity): implement stickiness so that the same client is always redirected to the same instance behind a load balancer.
- sticy session--cookie names:
  - application-based cookies: custom cookie(generated by the target), application cookie(generated by the load balancer)
  - duration-based cookies: cookies generated by elb
- cross-zone load balancing:
  - with cross-zone load balancing: each load balancer instance distributes evenly across all registered instances in all AZ
  - without cross-zone load balancing: Requests are distributed in the instances of the node of the Elastic Load Balancer
  - for alb: it is enabled by default, no charges
  - for nlb: disabled by default, will be charged
- SSL(Secure Sockets Layer)/TLS(Transport Layer Security) - basics: An SSL Certificate allows traffic between your clients and your load balancer to be encrypted in transit (in-flight encryption)
- Public SSL certificates are issued by Certificate Authorities (CA). Also, the certificate needs to be renewed
- load balancer -- ssl certficate: clients can use SNI(server name indication to specify the hostname they reach)
- SSL - server name indication: solves the problem of loading multiple ssl certificates onto one web server
- elb -- ssl certificates: alb, nlb support mutliple ssl certificates, clients need to specify SNI
- connection draining:
  - feature naming: `connection draining` for clb, `deregistration delay` for alb, nlb
  - Time to complete “in-flight requests” while the instance is de-registering or unhealthy
- elb health checks:
  - target health status: initial, healthy, unhealthy, unused, draining, unavailable
  - if a target group contains only unhealthy targets, elb routes requests across its unhealthy targets
- elb error code: 200, 4xx, 5xx(503 service unavailable)
- elb monitoring: All Load Balancer metrics are directly pushed to CloudWatch metrics
- elb troubleshooting using metrics:
  - 400 bad requests
  - 503 service unavailable --> check health status of your service in every AZ
  - 504 gateway timeout --> check if `keep-alive` setting on ec2 is enabled
- elb access logs: stored in s3 buckets, encrypted
- alb: not integrated with x-ray yet
- target group settings:
  - deregistration delay
  - slow start: load balancer will gradually increase the number of request sent to the target which gives targets time to warm-up
  - load balancing algorithm type
    - least outstanding requests: The next instance to receive the request is the instance that has the lowest number of pending/unfinished requests(alb, clb)
    - round robin: Equally choose the targets from the target group(alb, clb)
    - flow hash: Selects a target based on the protocol, source/destination IP address, source/destination por t, and TCP sequence number. Each TCP/UDP connection is routed to a single target for the life of the connection. (nlb)
  - stickness enabled; stickness type; stickness app cookie name; stickness app cookie duration; stickness lb cookie duration
- alb -- listener rules: host-headers, http methods, path, source-ip, http headers, query-string
- target group weighting: Specify weight for each Target Group on a single Rule. Allows you to control the distribution of the traffic to your applications
- auto scaling group: scale in/out; re-create ec2 for any unhealthy
  - min, max, desired
- asg attributes: `launch template`, capacity, scaling policies
- auto scaling -- cloudwatch alarm
- asg -- scaling policies:
  - dynamic scaling: target tracking scaling; simple/step scaling
  - scheduled scaling: anticipate the usage pattern
  - predictive scaling
- metrics to scale on: cpu utilization, requests per target, average network in/out, or custom metrics
- asg -- scaling cooldown: After a scaling activity happens, you are in the cooldown period (default 300 seconds). During the cooldown period, the ASG will not launch or terminate additional instances (to allow for metrics to stabilize). **Advice**: Use a ready-to-use AMI to reduce configuration time in order to be serving request fasters and reduce the cooldown period
- asg -- lifecycle hooks: EC2_Instance_Launching, EC2_Instance_Terminating
- launch configuration vs launch template: launch configuration is a legacy. while launch template can have multiple versions, provision on-demand and spot instances, support placement group, can use T2 unlimited burst feature
- asg with sqs: cloudwatch metric - queue length: ApproximateNumberOfMessages, then trigger an alarm and use an alarm action to scale asg
- asg health checks: To make sure you have high availability, means you have least 2 instances running across 2 AZ in your ASG (must configure multi-AZ ASG). health-check: ec2 status, elb status, or custom health checks
- troubleshooting asg issues
  - <number of instances> instance(s) are already running: asg has reached the max capacity
  - Launching EC2 instances is failing: sg or key pair not exist
  - If the ASG fails to launch an instance for over 24 hours, it will automatically suspend the processes (administration suspension)
- cloudwatch metrics for asg:
  - asg-level metrics(opt-in)
  - ec2-level metrics(enabled)
- AWS Auto Scaling: Backbone service of auto scaling for scalable resources in AWS:
  - asg
  - ec2 spot fleet requests
  - ecs
  - dynamodb
  - aurora
- AWS Auto Scaling - scaling plan
  - dynamic scaling
  - predictive scaling



## Elastic Beanstalk

- overview: a developer centric view of deploying an application on AWS. It uses components like ec2, asg, elb, rds,...
  - managed service: handle underlying resources provisioning and configurations, just the application code is the responsibility of the dev
  - we still have full control over the configuration
- elastic beanstalk - components:
  - application: a collection of versions, environments...
  - application version: an iteration of your code
  - environments: collection of aws resources running in an application version
- web server tier vs worker tier
  - web server tier: elb, asg, ec2
  - worker tier: sqs, asg, ec2
- elastic beanstalk deployment mode
  - single instance for dev
  - HA with load balancer for prod

## Cloudformation

- overview: a declarative way of outlining your AWS Infrastructure, for any resources (most of them are supported)
- benefits of cloudformation
  - configurations can be version controlled
  - each resource has a tag within the stack so that you can estimate the cost
  - productivity: declarative, quickly recreate the stack
  - separation of concern: multiple stacks
  - existing templates (documention)
- how cloudformation works
  - templates uploaded to s3 bucket so that cloudformation can reference, then cloudformation will create stacks which will create resources
  - to update a template, we have to upload a new version(cannot edit the current one)
- deploy cloudformation templates
  - manually --> cloudformation console
  - automatically --> edit yml files, and use aws cli or CD tool to deploy your templates
- cloudformation building block
  - template's components:
    - resources: such as `AWS::EC2::Subnet`,
    - parameters: must provide when deploying the template. some options like `AllowedValues`, `Default`, `NoEcho`(boolean). Use **!Ref(Fn::Ref)** to reference the parameters. Also aws has offer some `pseudo parameters` to use
    - mappings: fixed variables. to access map, use **Fn::FindInMap(!FindInMap)**: `!FindInMap [MapName,TopLevelKey, SecondLevelKey]`
    - outputs: The Outputs section declares optional outputs values that we can import into other stacks (if you export them first)! Then later, in other template, we use function `Fn::ImportValue`
    - conditions: used to control the creation of resources or outputs based on a condition (intrinsic function:logical). Conditions can be applied to resources / outputs / etc...
    - aws template format version,...
  - template's helpers: references, functions
- cloudformation -- intrinsic functions
  - Fn::Ref
  - Fn::GetAtt
  - Fn::FindInMap
  - Fn::ImportValue
  - Fn::Base64 (example, ec2 user data)
  - condition functions
- cloudformation rollback: creation/update fails, check the logs
- cloudformation -- service role: `iam:PassRole` permission can be given to the users, who can then allows cloudformation to have enough permissions to work with resources
- cloudformation -- capabilities:
  - CAPABILITY_NAMED_IAM, CAPABILITY_IAM
  - CAPABILITY_AUTO_EXPAND
  - InsufficientCapabilitiesException
- cloudformation deletion policy
  - delete (be aware of s3 bucket)
  - retain
  - snapshot
- cloudformation stack policy: During a CloudFormation Stack update, all update actions are allowed on all resources (default). A Stack Policy is a JSON document that defines the update actions that are allowed on specific resources during Stack updates. Protect resources from unintentional updates.
- cloudformation termination protection: to protect the stack 
- cloudformation -- custom resources
  - used to define resources not supported by aws, define resources that can be outside of cloudformation, custom scripts run during create/update/delete through lambda function
  - service token: specify where cloudformation sends request to, such as lambda, sns
- cloudformation -- dynamic references: Reference external values stored in `Systems Manager Parameter Store` and `Secrets Manager` within CloudFormation templates
  - option 1: ManageMasterUserPassword – creates admin secret implicitly (RDS, Aurora will manage the secret in Secrets Manager and its rotation)
  - option 2: dynamic reference: create a secret, reference the secret in DB resource, then link the secret to the DB instance
- cloudformation -- helper scripts(python scripts come with aws linux ami)
  - Init: A config contains the following and is executed in that order -- packages(packages to download and install), groups, users, sources(download files), files(create files on the ec2), commands(commands to run), services(launch a list of sysvinit)
  - cfn-init: Used to retrieve and interpret the resource metadata, installing packages, creating files and starting services
  - cfn-signal & wait condition:
    - run `cfn-signal` after `cfn-init`
    - define a wait condition to block the template until it receives a signal from `cfn-signal`, we attach a `CreationPolicy` and `Count`
- cloudformation -- nested stacks: best for re-usable configurations, and To update a nested stack, always update the parent (root stack).
- cross stacks vs nested stacks:
  - cross stacks: Helpful when stacks have different lifecycles. Use Outputs Export and Fn::ImportValue. When you need to pass export values to
many stacks (VPC Id...)
  - nested stacks: Helpful when components must be re-used. The nested stack only is important to the higher-level stack (it’s not shared)
- cloudformation -- dependson
  - Applied automatically when using !Ref and !GetAtt
- cloudformation -- stackSets: Create, update, or delete stacks across multiple accounts and regions with a single operation/template
- cloudformation -- stackSet permission models:
  - self-managed permissions: admin account and trusted target accounts
  - service-managed permissions: aws organization
- cloudformation -- troubleshooting
  - delete_failed
  - update_rollback_failed
- cloudformation -- stackSet troubleshooting
  - a stack operation failed, and the stack instance status is OUTDATED: could be insufficient permissions, trying to create a global unique resources(s3), admin account has no trust relationship with target account, reached a limit (service quotas) in target account.


## Lambda

- overview: managed service, auto-scaling, short-execution, run on-demand
- benefits: free tier, easy to increase RAM&CPU, integrated with other services, easy to monitor
- s3 events notification
- lambda execution role: Grants the Lambda function permissions to AWS services / resources
  - When you use an event source mapping to invoke your function, Lambda uses the execution role to read event data.
  - Best practice: create one Lambda Execution Role per function
- lambda resource based policies: Use resource-based policies to give other accounts and AWS services permission to use your Lambda resources
- lambda logging and monitoring
  - cloudwatch logs (make sure lambda has permissions to write logs to the cloudwatch logs)
  - cloudwatch metrics: invocations, durations, errors, throttles, deadlettererrors(failed to send events to dlq), iteratorAge(event source mapping reads from stream), concurrentExecution
- lambda tracing with x-ray: need to enable, and sdk is required
- lambda configuration
  - RAM: from 128MB to 10GB in 1MB increments; The more RAM you add, the more vCPU credits you get; At 1,792 MB, a function has the equivalent of one full vCPU; After 1,792 MB, you get more than one CPU, and need to use multi-threading in your code to benefit from it (up to 6 vCPU)
  - timeout: 3s - 900s
- lambda execution context
  - The execution context is a temporary runtime environment that initializes any external dependencies of your lambda code
  - Great for database connections, HTTP clients, SDK clients...
  - The execution context is maintained for some time in anticipation of another Lambda function invocation
  - The next function invocation can “re-use” the context to execution time and save time in initializing connections objects
  - The execution context includes the /tmp directory
- lambda function /tmp space: 10GB max; use s3 if persistence is required; use KMS if encryption is required
- lambda concurrency and throttling
  - concurrency limit: 1000 (soft limit)
  - reserved concurrency: at function level
  - throttle behavior: synchronous(429 error), asynchronous(retry and retry interval increase exponentially from 1s to 5min, and then go to DLQ)
  - concurrency issue: if do not set reserved concurrency, then other services may get impacted
  - cold start(initialize dependencies, sdk,...) & provision concurrency(concurrency allocated, no cold start): 
- lambda monitoring -- cloudwatch alarms
- lambda monitoring -- cloudwatch logs
- lambda monitoring -- cloudwatch logs insights
  - Collects, aggregates, and summarizes: system-level metrics, diagnostic information

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
