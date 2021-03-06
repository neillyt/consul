---
name: 'ACL Replication for Multiple Datacenters'
content_length: 15
id: acl-replication
layout: content_layout
products_used:
  - Consul
description: Configure tokens, policies, and roles to work across multiple datacenters.
---

You can configure tokens, policies and roles to work across multiple datacenters. ACL replication has several benefits. 

1. It enables authentication of nodes and services between multiple datacenters. 
1. The secondary datacenter can provide failover for all ACL components created in the primary datacenter. 
1. Sharing policies reduces redundancy for the operator.

## Prerequisites

Before starting this guide, each datacenter will need to have ACLs enabled, the process is outlined in the [Securing Consul with ACLs
guide](/consul/security-networking/production-acls). This guide includes the additional ACL replication configuration for the Consul
agents not covered in the Securing Consul with ACL guide. 

Additionally,
[Basic Federation with WAN Gossip](/consul/security-networking/datacenters) is required. 

## Introduction 

In this guide, you will setup ACL replication. This is a multi-step process
that includes:

- Setting the `primary_datacenter` parameter on all Consul agents in the primary datacenter.  
- Creating the replication token.  
- Configuring the `primary_datacenter` parameter on all Consul agents in the secondary datacenter.  
- Enabling token replication on the servers in the secondary datacenter.  
- Applying the replication token to all the servers in the secondary datacenter. 

You should complete this guide during the initial ACL bootstrapping
process. 

-> After ACLs are enabled you must have a privileged token to complete any
operation on either datacenter. You can use the initial
`bootstrap` token as your privileged token.

## Configure the Primary Datacenter

~> Note, if your primary datacenter uses the default `datacenter` name of
`dc1`, you must set a different `datacenter` parameter on each secondary datacenter.
Otherwise, both datacenters will be named `dc1` and there will be conflicts.

### Consul Servers and Clients

You should explicitly set the `primary_datacenter` parameter on all servers
and clients, even though replication is enabled by default on the primary
datacenter. Your agent configuration should be similar to the example below.  

```json 
{ 
"datacenter": "primary_dc", 
"primary_datacenter": "primary_dc",
"acl": { 
  "enabled": true, 
  "default_policy": "deny", 
  "down_policy": "extend-cache",
  "enable_token_persistence": true 
  } 
} 
```

The `primary_datacenter`
[parameter](https://www.consul.io/docs/agent/options.html#primary_datacenter)
sets the primary datacenter to have authority for all ACL information. It
should also be set on clients, so that the they can forward API
requests to the servers. 

Finally, start the agent.

```sh 
$ consul agent -config-file=server.json 
```

Complete this process on all agents. If you are configuring ACLs for the
first time, you will also need to [compelete the bootstrapping process](/consul/security-networking/production-acls) now.

## Create the Replication Token for ACL Management

Next, create the replication token for managing ACLs
with the following privileges.

- acl = "write" which will allow you to replicate tokens.  
- operator = "read" for replicating proxy-default configuration entries.
- service_prefix, policy = "read" and intentions = "read" for replicating
service-default configuration entries, CA, and intention data. 

```hcl 
acl = "write"

operator = "read"

service_prefix "" { 
  policy = "read" 
  intentions = "read" 
} 
```

Now that you have the ACL rules defined, create a policy with those rules. 

```sh 
$ consul acl policy create -name replication -rules @replication-policy.hcl 
ID:           240f1d01-6517-78d3-ec32-1d237f92ab58
Name:         replication 
Description: Datacenters: 
Rules: acl = "write"

operator = "read"

service_prefix "" { policy = "read" intentions = "read" } 
```

Finally, use your newly created policy to create the replication token.

```sh 
$ consul acl token create description "replication token" -policy-name replication 
AccessorID:   67d55dc1-b667-1835-42ab-64658d64a2ff 
SecretID:     fc48e84d-3f4d-3646-4b6a-2bff7c4aaffb 
Description:  replication token 
Local:        false
Create Time:  2019-05-09 18:34:23.288392523 +0000 UTC 
Policies: 
  240f1d01-6517-78d3-ec32-1d237f92ab58 - replication 
```

## Enable ACL Replication on the Secondary Datacenter

Once you have configured the primary datacenter and created the replication
token, you can setup the secondary datacenter. 

-> Note, your initial `bootstrap` token can be used for the necessary
privileges to complete any action on the secondary servers. 

### Configure the Servers 

You will need to set the `primary_datacenter` parameter to the name of your
primary datacenter and `enable_token_replication` to true on all the servers.  

```json 
{ 
"datacenter": "dc_secondary", 
"primary_datacenter": "primary_dc", 
"acl": { 
  "enabled": true, 
  "default_policy": "deny", 
  "down_policy": "extend-cache", 
  "enable_token_persistence": true,
  "enable_token_replication": true
  } 
} 
```

Now you can start the agent.

```sh 
$ consul agent -config-file=server.json 
``` 

Repeat this process on all the servers.

### Apply the Replication Token to the Servers

Finally, apply the replication token to all the servers using the CLI. 

```sh 
$ consul acl set-agent-token replication <token> 
ACL token "replication" set successfully 
```

Once token replication has been enabled, you will also be able to create
datacenter local tokens.

Repeat this process on all servers. If you are configuring ACLs for the
first time, you will also need to [set the agent token](/consul/security-networking/production-acls#add-the-token-to-the-agent).

Note, the clients do not need the replication token.

### Configure the Clients

For the clients, you will need to set the `primary_datacenter` parameter to the
name of your primary datacenter and `enable_token_replication` to true.

```json 
{ 
"datacenter": "dc_secondary", 
"primary_datacenter": "primary_dc",
"acl": { 
  "enabled": true, 
  "default_policy": "deny", 
  "down_policy": "extend-cache", 
  "enable_token_persistence": true, 
  "enable_token_replication": true 
  } 
} 
```

Now you can start the agent.

```sh 
$ consul agent -config-file=server.json 
``` 

Repeat this process on all clients. If you are configuring ACLs for the
first time, you will also need to [set the agent token](/consul/security-networking/production-acls#add-the-token-to-the-agent). 

## Check Replication 

Now that you have set up ACL replication, you can use the [HTTP API](https://www.consul.io/api/acl/acl.html#check-acl-replication) to check
the configuration.

```sh 
$ curl http://localhost:8500/v1/acl/replication?pretty
{
  "Enabled":true,
  "Running":true,
  "SourceDatacenter":"primary_dc",
  "ReplicationType":"tokens",
  "ReplicatedIndex":19,
  "ReplicatedTokenIndex":22,
  "LastSuccess":"2019-05-09T18:54:09Z",
  "LastError":"0001-01-01T00:00:00Z"
}
```

Notice, the "ReplicationType" should be "tokens". This means tokens, policies,
and roles are being replicated. 

## Summary

In this guide you setup token replication on multiple datacenters. You can complete this process on an existing datacenter, with minimal 
modifications. Mainly, you will need to restart the Consul agent when updating
agent configuration with ACL parameters. 

If you have not configured other secure features of Consul,
[certificates](consul/security-networking/certificates) and
[encryption](consul/security-networking/agent-encryption),
we recommend doing so now. 
