+++
title = "Redis Streams (supports Redis Sentinel)"
availability = "v2.5+"
maintainer = "Community"
description = "Redis Streams scaler with support for Redis Sentinel topology"
go_file = "redis_streams_scaler"
+++

### Trigger Specification

Redis 5.0 introduced [Redis Streams](https://redis.io/topics/streams-intro) which is an append-only log data structure.

One of its features includes [`Consumer Groups`](https://redis.io/topics/streams-intro#consumer-groups), that allows a group of clients to co-operate consuming a different portion of the same stream of messages.

This specification describes the `redis-sentinel-streams` trigger that scales based on the *Pending Entries List* (see [`XPENDING`](https://redis.io/commands/xpending)) for a specific Consumer Group of a Redis Stream and supports a Redis Sentinel setup.


```yaml
triggers:
- type: redis-sentinel-streams
  metadata:
    addresses: localhost:26379 # Required if hosts and ports are not provided. Format - comma separated list of host:port
    hosts: localhost # Comma separated lists of hosts. Required if address is not provided
    ports: "26379" # Comma separated lists of ports. Required if addresses are not provided and hosts has been provided.
    usernameFromEnv: REDIS_USERNAME # optional (can also use authenticationRef)
    passwordFromEnv: REDIS_PASSWORD # optional (can also use authenticationRef)
    stream: my-stream # Required - name of the Redis Stream
    consumerGroup: my-consumer-group # Required - name of consumer group associated with Redis Stream
    pendingEntriesCount: "10" # Required - number of entries in the Pending Entries List for the specified consumer group in the Redis Stream
    enableTLS: "false" # optional
    unsafeSsl: "false" # optional
    # Alternatively, you can use existing environment variables to read configuration from:
    # See details in "Parameter list" section
    addressesFromEnv: REDIS_ADDRESSES # Optional. You can use this instead of `addresses` parameter
    hostsFromEnv: REDIS_HOSTS # Optional. You can use this instead of `hosts` parameter
    portsFromEnv: REDIS_PORTS # Optional. You can use this instead of `ports` parameter
```

**Parameter list:**

- `addresses` - Comma separated list of hosts and ports of Redis Sentinel nodes in the format `host:port` for example `node1:26379, node2:26379, node3:26379`.

> As an alternative to the `addresses` field, the user can specify `hosts` and `ports` parameters.

- `hosts` - Comma separated list of hosts of Redis Sentinel nodes.

> It is not required if `addresses` has been provided.

- `ports`: Comma separated list of ports for corresponding hosts of Redis Sentinel nodes.

> It is only to be used along with the `hosts`/`hostsFromEnv` attribute and not required if `addresses` has been provided.

- `usernameFromEnv` - Name of the environment variable your deployment uses to get the Redis username. (Optional)
- `passwordFromEnv` - Name of the environment variable your deployment uses to get the Redis password. (Optional)

- `sentinelUsernameFromEnv` - Name of the environment variable your deployment uses to get the Redis Sentinel username. (Optional)
- `sentinelPasswordFromEnv` - Name of the environment variable your deployment uses to get the Redis Sentinel password. (Optional)

- `sentinelMaster` - The name of the master in Sentinel to get the Redis server address for.
- `stream` - Name of the Redis Stream.
- `consumerGroup` - Name of the Consumer group associated with Redis Stream.
- `pendingEntriesCount` - Threshold for the number of `Pending Entries List`. This is the average target value to scale the workload. (Default: `5`, Optional)
- `enableTLS` - Allow a connection to Redis using tls. (Values: `true`, `false`, Default: `false`, Optional)
- `unsafeSsl` - Used for skipping certificate check e.g: using self signed certs. (Values: `true`,`false`, Default: `false`, Optional, This requires `enableTLS: true`)

Some parameters could be provided using environmental variables, instead of setting them directly in metadata. Here is a list of parameters you can use to retrieve values from environment variables:

- `addressesFromEnv` - The hosts and corresponding ports of Redis Sentinel nodes, similar to `addresses`, but reads it from an environment variable on the scale target. Name of the environment variable your deployment uses to get the URLs of Redis Sentinel nodes. The resolved hosts should follow a format like `node1:26379, node2:26379, node3:26379 ...`.
- `hostsFromEnv` - The hosts of the Redis Sentinel nodes, similar to `hosts`, but reads it from an environment variable on the scale target.
- `portsFromEnv` - The corresponding ports for the hosts of Redis Sentinel nodes, similar to `ports`, but reads it from an environment variable on the scale target.
- `sentinelMasterFromEnv` - The name of the master in Sentinel to get the Redis server address for, similar to `sentinelMaster`, but reads it from an environment variable on the scale target.

### Authentication Parameters

The scaler supports two modes of authentication:

#### Using username/password authentication

Use the `username` and `password` field in the `metadata` to specify the name of an environment variable that your deployment uses to get the Redis username/password.

This is usually resolved from a `Secret V1` or a `ConfigMap V1` collections. `env` and `envFrom` are both supported.

Here is an example:

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: redis-streams-scaledobject
  namespace: default
spec:
  scaleTargetRef:
    name: redis-streams-consumer
  pollingInterval: 15
  cooldownPeriod: 200
  maxReplicaCount: 25
  minReplicaCount: 1
  triggers:
    - type: redis-sentinel-streams
      metadata:
        addressesFromEnv: REDIS_ADDRESSES
        usernameFromEnv: REDIS_USERNAME # name of the environment variable in the Deployment
        passwordFromEnv: REDIS_PASSWORD # name of the environment variable in the Deployment
        stream: my-stream
        consumerGroup: consumer-group-1
        pendingEntriesCount: "10"
        sentinelMaster: "mymaster"
```

#### Using `TriggerAuthentication`

You can use `TriggerAuthentication` CRD to configure the authentication. For example:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: redis-streams-auth
type: Opaque
data:
  redis_username: <encoded redis username>
  redis_password: <encoded redis password>
---
apiVersion: keda.sh/v1alpha1
kind: TriggerAuthentication
metadata:
  name: keda-redis-stream-triggerauth
spec:
  secretTargetRef:
    - parameter: username
      name: redis-streams-auth # name of the Secret
      key: redis_username # name of the key in the Secret
    - parameter: password
      name: redis-streams-auth # name of the Secret
      key: redis_password # name of the key in the Secret
---
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: redis-streams-scaledobject
  namespace: default
spec:
  scaleTargetRef:
    name: redis-streams-consumer
  pollingInterval: 15
  cooldownPeriod: 200
  maxReplicaCount: 25
  minReplicaCount: 1
  triggers:
    - type: redis-sentinel-streams
      metadata:
        address: node1:26379, node2:26379, node3:26379
        stream: my-stream
        consumerGroup: consumer-group-1
        pendingEntriesCount: "10"
        sentinelMaster: "mymaster"
      authenticationRef:
        name: keda-redis-stream-triggerauth # name of the TriggerAuthentication resource
```
