---
title : "Network foundation"
date : 2026-07-28
weight : 4
chapter : false
pre : " <b> 3.4. </b> "
---

`BorrowitFoundation` creates the VPC, four security groups, an S3 gateway endpoint,
five interface VPC endpoints and the ECR repository. Everything in it is free or
costs cents **except the interface endpoints**, which are ~$36/month and make this
the second most expensive stack in the project. The application stack is destroyed
and rebuilt against this one repeatedly, so it stays deployed — and so does that
charge.

The reasoning behind the NAT-free design is in
[3.3.3](../3.3-Architecture/3.3.3-networking/); this page builds it.

#### The VPC

```typescript
this.vpc = new ec2.Vpc(this, 'Vpc', {
  maxAzs: 2,
  natGateways: 0,
  subnetConfiguration: [
    { name: 'public', subnetType: ec2.SubnetType.PUBLIC, cidrMask: 24 },
    { name: 'isolated', subnetType: ec2.SubnetType.PRIVATE_ISOLATED, cidrMask: 24 },
  ],
});
```

Four subnets across two Availability Zones:

| Subnet type | Contains | Route to internet |
|---|---|---|
| `PUBLIC` ×2 | ALB, Fargate tasks | Via internet gateway |
| `PRIVATE_ISOLATED` ×2 | RDS instance | **None, either direction** |

`maxAzs: 2` is the minimum an ALB accepts — it requires subnets in at least two
AZs. This is also what makes a Multi-AZ demonstration possible without changing
the network.

{{% notice warning %}}
Do not change the Fargate subnets to `PRIVATE_WITH_EGRESS`. That subnet type
silently provisions a NAT Gateway, adding ~$33/month without any warning at synth
time.
{{% /notice %}}

#### The security group chain

Three groups, each accepting inbound traffic from exactly one source:

```typescript
this.albSecurityGroup.addIngressRule(
  ec2.Peer.anyIpv4(), ec2.Port.tcp(80), 'HTTP from the internet');

this.serviceSecurityGroup.addIngressRule(
  this.albSecurityGroup, ec2.Port.tcp(3456), 'Load balancer to app');

this.databaseSecurityGroup.addIngressRule(
  this.serviceSecurityGroup, ec2.Port.tcp(5432), 'App to Postgres');
```

The chain means a packet from the internet can only reach the database by first
passing the ALB, then the task. There is no rule anywhere that would let it skip a
step.

Two details that matter:

+ **Rules reference security groups, not CIDR blocks.** When ECS replaces a task
  and the public IP changes, the rule is still correct. A CIDR-based rule would
  need updating on every task replacement.
+ **`allowAllOutbound: false` on the database group.** The database has no reason
  to initiate outbound connections, so it cannot.

A fourth group, `EndpointSecurityGroup`, sits outside this chain and guards the
interface endpoint ENIs. It is covered below.

#### S3 gateway endpoint

```typescript
this.vpc.addGatewayEndpoint('S3Endpoint', {
  service: ec2.GatewayVpcEndpointAwsService.S3,
});
```

Gateway endpoints are **free** — unlike the interface endpoints priced out in
[3.3.3](../3.3-Architecture/3.3.3-networking/). This one adds a route to the
route tables so S3 traffic goes over the AWS network rather than out through the
internet gateway. Lower latency, no data transfer charge, no cost.

It is also load-bearing rather than decorative: ECR stores image layers in S3, so
this endpoint carries part of every task startup, not just user uploads.

#### Interface endpoints

The same stack puts ECR, Secrets Manager, CloudWatch Logs and SSM behind
PrivateLink, so that traffic never leaves the VPC. These deploy **every time** —
there is no flag to turn them on. The ~$36/month that costs in one AZ is argued in
[3.3.3](../3.3-Architecture/3.3.3-networking/).

```typescript
const endpointSecurityGroup = new ec2.SecurityGroup(this, 'EndpointSecurityGroup', {
  vpc: this.vpc,
  description: 'BorrowIt interface VPC endpoints',
  allowAllOutbound: false,
});
endpointSecurityGroup.addIngressRule(
  this.serviceSecurityGroup, ec2.Port.tcp(443), 'Fargate tasks to AWS service endpoints');

this.vpc.addInterfaceEndpoint('SecretsManagerEndpoint', {
  service: ec2.InterfaceVpcEndpointAwsService.SECRETS_MANAGER,
  subnets,
  securityGroups: [endpointSecurityGroup],
  privateDnsEnabled: true,
});
```

Five endpoints are created this way: `ecr.api`, `ecr.dkr`, `logs`,
`secretsmanager` and `ssmmessages`. Two choices are worth copying:

+ **An explicit security group.** Left to itself `addInterfaceEndpoint` generates
  one that allows the entire VPC CIDR on 443. This one accepts 443 from
  `ServiceSecurityGroup` only.
+ **`privateDnsEnabled: true`.** This is what makes the regional service hostname
  resolve to the endpoint ENI. Without it, traffic keeps using the internet gateway
  and the endpoints are billed for nothing.

`subnets` pins them to one AZ. `npm run endpoints:ha` redeploys them across both
for ~$73/month, which is worth doing for a resilience demonstration and not
worth leaving on.

{{% notice warning %}}
This is the one line in `BorrowitFoundation` that costs real money, and it keeps
costing it while `BorrowitApp` is destroyed — this stack is never torn down. If the
credit balance gets tight, removing the endpoint block is the largest single saving
available outside `BorrowitApp`.
{{% /notice %}}

#### ECR repository

```typescript
this.repository = new ecr.Repository(this, 'Repository', {
  repositoryName: 'borrowit-be',
  imageScanOnPush: true,
  lifecycleRules: [{ maxImageCount: 10, description: 'Keep the last 10 images' }],
});
```

Placed in this stack rather than in `BorrowitApp` deliberately: destroying the
application layer must not discard images already built and pushed. The lifecycle
rule caps storage at ten images so old builds do not accumulate.

#### Deploy

```powershell
cd infra
npx cdk deploy BorrowitFoundation
```

#### Verify

Open the [VPC console](https://ap-southeast-1.console.aws.amazon.com/vpcconsole/home?region=ap-southeast-1#vpcs:)
and check the resource map: four subnets, two AZs, one internet gateway, and
**no NAT Gateway**.

<!-- SCREENSHOT: /images/3-Workshop/3.4-Network/vpc-resource-map.png -->

<!-- SCREENSHOT: /images/3-Workshop/3.4-Network/security-groups.png
     The three groups with inbound rules expanded, showing each references the
     previous group rather than a CIDR. -->

Confirm the isolated subnets have no route to `0.0.0.0/0`:

```powershell
aws ec2 describe-route-tables --region ap-southeast-1 `
  --filters "Name=vpc-id,Values=<VpcId from the stack output>" `
  --query "RouteTables[].{Name:Tags[?Key=='Name']|[0].Value,Routes:Routes[].DestinationCidrBlock|join(', ',@)}" `
  --output table
```

Only the two public route tables carry a default route:

```
BorrowitFoundation/Vpc/isolatedSubnet1    10.0.0.0/16
BorrowitFoundation/Vpc/isolatedSubnet2    10.0.0.0/16
BorrowitFoundation/Vpc/publicSubnet1      10.0.0.0/16, 0.0.0.0/0
BorrowitFoundation/Vpc/publicSubnet2      10.0.0.0/16, 0.0.0.0/0
```

The isolated subnets route only inside the VPC. There is no path from the database
to the internet, and none back — which is what makes the absence of a NAT Gateway
a design decision rather than an omission.

List the endpoints:

```powershell
aws ec2 describe-vpc-endpoints --region ap-southeast-1 `
  --query "VpcEndpoints[].{Service:ServiceName,Type:VpcEndpointType,DNS:PrivateDnsEnabled}" `
  --output table
```

Expect six rows: the S3 gateway endpoint plus the five interface endpoints. The
**five interface endpoints** must show `PrivateDnsEnabled` true — a `false` there
means the endpoint is being billed and bypassed.

The S3 gateway endpoint shows `False`, and that is correct rather than a fault.
Gateway endpoints are not DNS-based at all: they work by adding a prefix-list route
to the subnet route tables, so there is no private hostname to enable. Only
interface endpoints have a `PrivateDnsEnabled` setting worth reading.

<!-- SCREENSHOT: /images/3-Workshop/3.4-Network/vpc-endpoints.png
     The endpoint list with private DNS enabled. -->

{{% notice note %}}
**Cost:** approximately **$36/month**, effectively all of it the five interface
endpoints. VPCs, subnets, route tables, security groups and gateway endpoints are
free, and ECR storage for ten small images is a few cents. That makes this the
second most expensive stack in the project.
{{% /notice %}}
