# Moving from On-Premises to AWS: What to Get Right Before Cutover

Moving a server to AWS is straightforward compared with moving everything that depends on it.

An application might start successfully on an EC2 instance but still fail to generate invoices because its shared folder remains in the data center. Or it might work correctly but respond slowly because every database query now crosses a network connection.

A successful migration starts by understanding those dependencies, then deciding how much change the business can safely absorb.

## Understand the application before choosing AWS services

Consider an order-processing application with web servers, a PostgreSQL database, a shared invoice folder, and a nightly integration with a finance system.

A server inventory captures the machines. It does not explain how orders flow through them.

Before designing the AWS environment, map the application’s connections, scheduled jobs, authentication requirements, and storage access. Include dependencies that run infrequently, such as month-end reporting.

Measure current performance too. CPU and memory matter, but so do database latency, storage throughput, peak traffic, and response times. These measurements help you size the target environment and establish whether the migrated application performs acceptably.

Agree on business constraints early: acceptable downtime, recovery requirements, data residency, and budget. Those constraints should drive the architecture.

## Choose how much to change during the move

You do not need to redesign every application to migrate it.

**Rehosting** moves the application with minimal changes, typically onto EC2 virtual machines. **Replatforming** changes part of its platform, such as moving PostgreSQL to Amazon RDS. **Refactoring** changes the application’s architecture more substantially.

For the order-processing application, rehosting the web servers and replatforming the database might provide a manageable first step.

That still requires checking database versions, extensions, privileges, and application compatibility. A managed database reduces some administrative work; it does not remove responsibility for how the application uses it.

Some workloads should remain on-premises or be retired. Others may be better replaced by a commercial service. The right decision depends on business value, technical constraints, and the team’s ability to operate the result.

## Treat the hybrid network as part of the application

During migration, AWS and the data center often operate together.

Plan IP address ranges, routing, DNS, firewall rules, and connectivity before moving workloads. Overlapping address ranges can complicate routing. Missing DNS resolution can make a reachable service appear unavailable.

Latency deserves particular attention. If processing one order requires many sequential database calls, placing the web servers in AWS while keeping the database on-premises can slow the entire workflow. Closely connected components often need to move together.

A VPN or AWS Direct Connect can provide connectivity, but their security properties differ. Direct Connect does not encrypt traffic by default, so encryption must be planned explicitly where required. [docs.aws.amazon.com](https://docs.aws.amazon.com/directconnect/latest/UserGuide/encryption-in-transit.html?utm_source=chatgpt.com)

## Plan the data cutover and rollback together

Copying existing data is only the first step when the application continues accepting orders.

For supported databases, replication can transfer changes while the source remains active. At cutover, you typically pause writes, let replication catch up, validate the target, and switch the application.

This can shorten downtime, but it does not eliminate the need for a controlled transition.

Rollback becomes harder once the AWS database accepts new orders. The original database is now missing those transactions. Switching the application back without reconciling them can lose business data.

Define that boundary before migration. Specify when rollback remains safe, how later transactions would be recovered, and who decides whether to proceed. Rehearse the procedure, including background jobs and integrations that might continue writing after users are disconnected.

## Prove that the application can be operated

Before cutover, establish access controls, encryption, logging, alerts, backups, and incident ownership. Test restoration rather than assuming a successful backup guarantees recovery.

Validate business workflows as well as infrastructure. Place an order, generate its invoice, and confirm that finance receives it. Run realistic performance tests and compare them with the original environment.

Estimate costs beyond compute: storage, database capacity, backups, networking, logs, licenses, and the temporary expense of running both environments.

Keep the source environment until a defined observation period confirms that the migrated application works, scheduled integrations complete, and recovery procedures are usable. That evidence gives you a sound basis for retiring the old system.