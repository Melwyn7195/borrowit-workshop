---
title : "Load balancer and health checks"
date : 2026-07-28
weight : 3
chapter : false
pre : " <b> 4.6.3 </b> "
---

The ALB is the most expensive component on the request path — roughly
**$17/month**, billed hourly whether or not any request arrives. Only the five
interface VPC endpoints cost more, and they are not on the request path at all.
It is worth understanding exactly what that $17 buys.

#### What the ALB provides

| Function | Why it matters here |
|---|---|
| Stable DNS name | Fargate tasks get a new IP every replacement; the ALB name never changes |
| Health checking | Removes unhealthy tasks from rotation automatically |
| Cross-AZ distribution | Required for a Multi-AZ demonstration |
| Connection draining | Lets in-flight requests finish before a task is stopped |
| Metrics | Request counts, latency percentiles, 4xx/5xx — the basis of [3.8](../../4.8-Observability/) |

<!-- TODO(prose): a Network Load Balancer is cheaper per hour and an API Gateway
     HTTP API is cheaper still at low volume. If you considered either, say why the
     ALB won - path-based routing and richer HTTP metrics are the usual answers. -->

#### Configuration

```typescript
const alb = new elbv2.ApplicationLoadBalancer(this, 'LoadBalancer', {
  vpc: props.vpc,
  internetFacing: true,
  securityGroup: props.albSecurityGroup,
  vpcSubnets: { subnetType: ec2.SubnetType.PUBLIC },
});

const listener = alb.addListener('Listener', { port: 80, open: false });

const targetGroup = listener.addTargets('AppTargets', {
  port: 3456,
  protocol: elbv2.ApplicationProtocol.HTTP,
  targets: [service],
  deregistrationDelay: cdk.Duration.seconds(15),
  healthCheck: {
    path: '/health',
    interval: cdk.Duration.seconds(30),
    timeout: cdk.Duration.seconds(5),
    healthyThresholdCount: 2,
    unhealthyThresholdCount: 3,
  },
});
```

`open: false` on the listener matters: without it, CDK would add its own ingress
rule to the security group, duplicating the one already defined in
[3.4](../../4.4-Network/) and making the source of truth ambiguous.

#### Two health checks answering different questions

This is the most important design point in the compute layer.

| | Liveness | Readiness |
|---|---|---|
| Owner | ECS task definition | ALB target group |
| Path | `/health/live` | `/health` |
| Touches the database | **No** | **Yes** — runs `SELECT 1` |
| On failure | ECS kills and replaces the task | ALB stops routing to the task |

The container check asks **"is the process alive?"** If it fails, the task is
killed and replaced.

The load balancer check asks **"should this task receive traffic?"** It runs a real
query, so a task that has lost its database connection is pulled out of rotation
without being destroyed.

**Why the split matters.** If the liveness check also queried the database, a brief
RDS interruption — a failover, a network blip — would fail liveness on every task
simultaneously. ECS would kill them all, the replacements would start against the
same unavailable database, fail again, and the service would enter a restart loop
that outlasts the original problem by a long margin.

Separating them means a database problem **degrades** the service instead of
**destroying** it. The tasks stay up, stop receiving traffic, and resume serving
automatically when the database returns.

<!-- TODO(prose): this is the single best thing in the report to be able to explain
     under questioning. Write it in your own words. If you measured the behaviour
     during a failover test, reference the result here. -->

#### Connection draining

```typescript
deregistrationDelay: cdk.Duration.seconds(15),
```

This must **exceed** the application's own drain delay (5 seconds). If it were
shorter, the task would close its socket while the load balancer was still routing
requests to it, producing 502s during every deployment.

#### Deploy and verify

Retrieve the URL:

```powershell
$env:API = (aws cloudformation describe-stacks --stack-name BorrowitApp `
  --query "Stacks[0].Outputs[?OutputKey=='ApiUrl'].OutputValue" --output text)
curl "$env:API/health"
```

#### Proving the isolation

The security group design in [4.3.3](../../4.3-Architecture/4.3.3-networking/)
claims that a task holding a public IP is still unreachable. Demonstrate it:

```powershell
$env:CLUSTER = (aws cloudformation describe-stacks --stack-name BorrowitApp `
  --query "Stacks[0].Outputs[?OutputKey=='ClusterName'].OutputValue" --output text)
$env:TASK = (aws ecs list-tasks --cluster $env:CLUSTER --query "taskArns[0]" --output text)

$env:ENI = (aws ecs describe-tasks --cluster $env:CLUSTER --tasks $env:TASK `
  --query "tasks[0].attachments[0].details[?name=='networkInterfaceId'].value" --output text)
$env:TASK_IP = (aws ec2 describe-network-interfaces --network-interface-ids $env:ENI `
  --query "NetworkInterfaces[0].Association.PublicIp" --output text)

# Must fail - the security group accepts only the ALB on 3456
curl "http://$($env:TASK_IP):3456/health" --max-time 10
```

{{% notice note %}}
**Cost:** ~$17/month for the ALB, plus LCU charges that are negligible at this
traffic level. This is the component to remove first if the system must be made
cheaper — and removing it means removing the stable entry point, so there is no
partial saving available.
{{% /notice %}}
