[![NPM version](https://badge.fury.io/js/low-cost-ecs.svg)](https://www.npmjs.com/package/low-cost-ecs)
[![PyPI version](https://badge.fury.io/py/low-cost-ecs.svg)](https://pypi.org/project/low-cost-ecs)
[![Release](https://github.com/rajyan/low-cost-ecs/workflows/release/badge.svg)](https://github.com/rajyan/low-cost-ecs/actions/workflows/release.yml)
[<img src="https://constructs.dev/badge?package=low-cost-ecs" width="150">](https://constructs.dev/packages/low-cost-ecs)

# Low-Cost ECS

A CDK construct that provides easy and [low-cost](#cost) ECS on EC2 server setup without a load balancer.

**This construct is for development purposes only**. See [Limitations](#limitations).

# Try it out!

The easiest way to see what this construct creates is to clone this repository and deploy the sample server.
Edit settings in `bin/low-cost-ecs.ts` and deploy the cdk construct. [Public hosted zone](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/AboutHZWorkingWith.html) is required.

```
git clone https://github.com/rajyan/low-cost-ecs.git
# edit settings in bin/low-cost-ecs.ts
npx cdk deploy
```

Access to configured `recordDomainNames` and see that the nginx sample server has been deployed.

# Installation

To use this construct in your cdk stack as a library,

```
npm install low-cost-ecs
```

```ts
import { Stack, StackProps } from 'aws-cdk-lib';
import { Construct } from 'constructs';
import { LowCostECS } from 'low-cost-ecs';

class SampleStack extends Stack {
    constructor(scope: Construct, id: string, props?: StackProps) {
        super(scope, id, props);

        const vpc = { /** Your VPC */ };
        const securityGroup = {/** Your security group */ };
        const serverTaskDefinition = {/** Your task definition */ };

        new LowCostECS(this, 'LowCostECS', {
            hostedZoneDomain: "rajyan.net",
            email: "kitakita7617@gmail.com",
            vpc: vpc,
            securityGroup: securityGroup,
            serverTaskDefinition: serverTaskDefinition
        });
    }
}
```

The required fields are `hostedZoneDomain` and `email`.
You can configure your server task definition and other props. Read [`LowCostECSProps` documentation](https://github.com/rajyan/low-cost-ecs/blob/main/API.md#low-cost-ecs.LowCostECSProps) for details.

# Why

ECS may often seem expensive when used for personal development purposes, because of the cost of the load balancer.
The application load balancer is a great service because it is easy to set up managed ACM certificates, it scales, and has dynamic port mappings and so on, 
but it is over-featured for running 1 ECS service.

However, to run an ECS server without a load balancer, you need to associate an Elastic IP to the host instance and install your certificate by yourself.
This construct aims to automate these works and to make it easy to deploy resources to run a low-cost ECS server.

# Overview

Resources generated in this stack

* Route53 A record
  * Forwarding to host instance Elastic IP
* Certificate State Machine
  * Install and renew certificates to EFS using [certbot-dns-route53](https://certbot-dns-route53.readthedocs.io/en/stable/)
  * Scheduled automated renewal every 60 days
  * Email notification on certbot task failure
* ECS on EC2 host instance
  * ECS-optimized Amazon Linux 2 AMI instance auto-scaling group
  * Automatically associated with Elastic IP on instance initialization
* ECS Service
  * TLS/SSL certificate installation on default container startup
  * Certificate EFS mounted on `/etc/letsencrypt`
* Others
  * VPC with only public subnets (no NAT Gateways to decrease cost)
  * Security groups with minimum inbounds
  * IAM roles with minimum privileges

# Cost

All resources except Route53 HostedZone should be included in [AWS Free Tier](https://docs.aws.amazon.com/whitepapers/latest/how-aws-pricing-works/get-started-with-the-aws-free-tier.html)
***if you are in the 12 Months Free period***.
After your 12 Months Free period, setting [`hostInstanceSpotPrice`](https://github.com/rajyan/low-cost-ecs/blob/main/API.md#low-cost-ecs.LowCostECSProps.property.hostInstanceSpotPrice) to use spot instances is recommended.

* EC2
  * t2.micro 750 instance hours (12 Months Free Tier)
  * 30GB EBS volume (12 Months Free Tier)
* ECS
  * No additional charge because using ECS on EC2
* EFS
  * Usage is very small, it should be free
* Cloud Watch
  * Usage is very small, and it should be included in the free tier
  * Enabling [`containerInsights`](https://github.com/rajyan/low-cost-ecs/blob/main/API.md#low-cost-ecs.LowCostECSProps.property.containerInsights) will charge for custom metrics

# Debugging

* SSM Session Manager

SSM manager is pre-installed (in ECS-optimized Amazon Linux 2 AMI) in the host instance and `AmazonSSMManagedInstanceCore` is added to the host instance role
to access and debug in your host instance.

```
aws ssm start-session --target $INSTANCE_ID
```

* ECS Exec

Service ECS Exec is enabled, so execute commands can be used to debug in your server task container.

```
aws ecs execute-command \
--cluster $CLUSTER_ID \
--task $TASK_ID \
--container nginx \
--command bash \
--interactive
```

# Limitations

The ECS service occupies the host port, only one service can be run at a time.
The old task must be terminated before the new task launches, and this causes downtime on release.

Also, if you make changes that require recreating service, you may need to manually terminate the task of old the service.
