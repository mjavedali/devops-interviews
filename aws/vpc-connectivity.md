# Connecting 20 AWS VPCs: When Transit Gateway Makes Sense

If 20 Amazon Virtual Private Clouds, or VPCs, all need to communicate, connecting every pair through VPC peering creates **190 peering connections**:

```text
Connections = n × (n − 1) / 2
            = 20 × 19 / 2
            = 190
```

Each connection also needs routes and appropriate security rules. Adding the twenty-first VPC introduces another 20 peering connections.

For this kind of network, **AWS Transit Gateway** is usually a practical choice. It provides a central routing hub, allowing each VPC to connect through one attachment.

Assume all 20 VPCs are in the same AWS Region and have non-overlapping IP address ranges. That gives us 20 VPC attachments instead of 190 peering connections.

## Why peering becomes difficult

VPC peering connects two VPCs directly. It works well when only a few networks need connectivity.

Its limitation is that peering does not support **transitive routing**. If VPC A peers with B, and B peers with C, A cannot reach C through B. A and C need their own peering connection. [Amazon Virtual Private Cloud](https://docs.aws.amazon.com/vpc/latest/peering/vpc-peering-basics.html?utm_source=chatgpt.com)

For 20 VPCs requiring full connectivity, that produces a full mesh. Troubleshooting means checking routes and access rules across many individual relationships.

Transit Gateway changes the topology. Traffic travels from the source VPC to the gateway, which selects the destination attachment using its route table.

## Attachments alone do not provide connectivity

Suppose an application in VPC A, using `10.1.0.0/16`, needs to reach a service in VPC B, using `10.2.0.0/16`.

The forward path needs three routing decisions:

1. VPC A’s relevant subnet route table sends `10.2.0.0/16` traffic to Transit Gateway.
2. The Transit Gateway route table used by A’s attachment sends that destination to B’s attachment.
3. VPC B delivers the traffic to the destination resource.

The return path needs corresponding routes back to VPC A. Security groups and network access control lists must also permit the traffic.

There are therefore **two routing layers to configure**: VPC route tables and Transit Gateway route tables. Propagating VPC routes into Transit Gateway does not automatically update the VPC subnet route tables.

Transit Gateway separates two concepts:

- **Association** determines which gateway route table handles traffic arriving from an attachment.
- **Propagation** adds an attachment’s routes to selected gateway route tables.

An attachment can propagate routes into multiple tables. This distinction becomes useful when controlling connectivity between environments. [AWS Architecture Blog](https://aws.amazon.com.cdn.amazon.com/blogs/architecture/field-notes-working-with-route-tables-in-aws-transit-gateway/?utm_source=chatgpt.com)

## Central connectivity does not require unrestricted access

Twenty VPCs may need access to shared infrastructure without needing unrestricted access to one another.

For example, development and production might both use a shared monitoring service, while development must remain isolated from production.

Separate Transit Gateway route tables can enforce that routing boundary. Security groups then restrict access to the required services and ports.

A route makes a destination reachable; it does not grant application access or inspect traffic. If inspection is required, design a path through AWS Network Firewall or another firewall solution, including the return path.

Plan non-overlapping CIDR blocks, which define each VPC’s IP address ranges, before connecting the networks. Transit Gateway does not resolve overlapping addressing automatically.

## Extend the design across accounts and on-premises

If the VPCs belong to different AWS accounts, a networking account can own Transit Gateway and share it through **AWS Resource Access Manager**, or AWS RAM. Application accounts attach their VPCs while the networking team manages central routing.

On-premises connectivity can use an AWS Site-to-Site VPN attachment. With Direct Connect, the standard integration uses a **transit virtual interface and a Direct Connect gateway associated with Transit Gateway**. A Direct Connect circuit does not attach directly to it. [AWS CLI 2.4.19 Command Reference](https://awscli.amazonaws.com/v2/documentation/api/2.4.19/reference/directconnect/create-transit-virtual-interface.html?utm_source=chatgpt.com)

Transit Gateway is regional. VPCs spread across Regions require an additional design, such as regional transit gateways connected through peering.

## Check the cost against the connectivity requirement

Transit Gateway simplifies topology, but it adds attachment-hour and data-processing charges. Traffic volume matters alongside the number of VPCs. [aws.amazon.com](https://aws.amazon.com/transit-gateway/pricing/?utm_source=chatgpt.com)

If only a few VPC pairs need connectivity, selective peering may be simpler and cheaper. Twenty VPCs do not automatically require a full mesh.

When all 20 need broad connectivity, Transit Gateway offers a manageable foundation: one attachment per VPC, central routing, and explicit boundaries between environments. The essential work remains defining which paths should exist, configuring both directions, and allowing only the traffic each application needs.