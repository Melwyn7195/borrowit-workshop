---
title : "Data layer"
date : 2026-07-28
weight : 5
chapter : false
pre : " <b> 3.5. </b> "
---

`BorrowitData` creates the RDS instance and, as a side effect, the Secrets Manager
secret that holds its credentials. The sizing rationale is in
[3.3.2](../3.3-Architecture/3.3.2-database/).

This is the stack that holds state, so it is the one not to destroy casually.

#### The instance

```typescript
this.database = new rds.DatabaseInstance(this, 'Database', {
  engine: rds.DatabaseInstanceEngine.postgres({
    version: rds.PostgresEngineVersion.VER_16,
  }),
  vpc: props.vpc,
  vpcSubnets: { subnetType: ec2.SubnetType.PRIVATE_ISOLATED },
  securityGroups: [props.databaseSecurityGroup],
  instanceType: ec2.InstanceType.of(
    ec2.InstanceClass.BURSTABLE4_GRAVITON,
    ec2.InstanceSize.MICRO,
  ),
  allocatedStorage: 20,
  maxAllocatedStorage: 20,
  storageEncrypted: true,
  multiAz,
  publiclyAccessible: false,
  databaseName: 'borrowit',
  credentials: rds.Credentials.fromGeneratedSecret('borrowit'),
  backupRetention: cdk.Duration.days(1),
  removalPolicy: cdk.RemovalPolicy.DESTROY,
});
```

#### Isolation, and what it costs you

`PRIVATE_ISOLATED` subnets plus `publiclyAccessible: false` means there is no
network path from outside the VPC to this database. Nothing can connect to it, and
it cannot connect out.

The practical consequence appears immediately: **you cannot run `psql` against it
from your laptop.** That is why the schema is loaded from inside a running
container in [3.6.4](../3.6-Compute/3.6.4-database-init/), rather than from a
workstation.

<!-- TODO(prose): frame this as a tradeoff you accepted knowingly rather than a
     surprise you hit. The alternative - a bastion host at ~$8/month, or making
     RDS publicly accessible - were both considered and rejected. -->

#### Credentials

```typescript
credentials: rds.Credentials.fromGeneratedSecret('borrowit'),
```

RDS generates a random password at creation and writes it into AWS Secrets Manager
as a JSON document:

```json
{
  "username": "borrowit",
  "password": "...",
  "engine": "postgres",
  "host": "borrowitdata-....ap-southeast-1.rds.amazonaws.com",
  "port": 5432,
  "dbname": "borrowit"
}
```

CDK never prints it, never writes it to `cdk.json`, and it never appears in the
synthesised template. Verify that claim rather than trusting it:

```powershell
npx cdk synth BorrowitData | Select-String "password"
```

If you need the connection details yourself:

```powershell
$env:SECRET_ARN = (aws cloudformation describe-stacks `
  --stack-name BorrowitData `
  --query "Stacks[0].Outputs[?OutputKey=='DbSecretArn'].OutputValue" `
  --output text)

aws secretsmanager get-secret-value --secret-id $env:SECRET_ARN --query SecretString --output text
```

{{% notice warning %}}
That command prints the password to your terminal. **Do not screenshot its
output.** If you want an image for the report, take it in the Secrets Manager
console with the value still hidden.
{{% /notice %}}

#### Deletion behaviour

```typescript
removalPolicy: cdk.RemovalPolicy.DESTROY,
deleteAutomatedBackups: true,
```

`DESTROY` is deliberate and is **not** the production-safe choice.

`RETAIN` would leave an orphaned instance billing after `cdk destroy` — on a fixed
credit balance, a worse outcome than losing seed data that a script rebuilds. In a
system with real user data, `RETAIN` is correct and the cost is accepted.

<!-- TODO(prose): say this in your own words. It is a good example of a decision
     that is right here and wrong in general, which is exactly the kind of nuance
     worth demonstrating. -->

#### Deploy

RDS takes **5–10 minutes** to create.

```powershell
npx cdk deploy BorrowitData
```

#### Verify

<!-- SCREENSHOT: /images/3-Workshop/3.5-Data/rds-instance.png
     Status Available, class db.t4g.micro, Multi-AZ: No. -->

<!-- SCREENSHOT: /images/3-Workshop/3.5-Data/rds-connectivity.png
     Connectivity & security tab: "Publicly accessible: No", DatabaseSecurityGroup
     attached, subnet group listing the isolated subnets. -->

{{% notice note %}}
**Cost:** ~$16/month — `db.t4g.micro` plus 20 GB of gp2 storage, billed against
Free Plan credits from the first hour. There is no 12-month free tier on this
account.
{{% /notice %}}
