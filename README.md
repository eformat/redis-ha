# Reliable, Scalable Redis on OpenShift

The following document describes the deployment of a reliable, multi-node Redis on OpenShift.

## Redis Clustering

- https://redis.io/topics/cluster-spec

See `cluster-statefulset` directory.

```
oc apply -f ./cluster-statefulset
```

Redis Cluster provides a way to run a Redis installation where data is automatically sharded across multiple Redis nodes.

## Redis Sentinel

- https://redis.io/topics/sentinel

Redis Sentinel provides high availability for Redis.

The following documentation deploys a master with replicated slaves, as well as replicated redis sentinels which are use for health checking and failover.

## Build configuration

Create project
```
oc new-project redis-ha --description "Redis HA" --display-name="Redis HA"
```

1. (Optional) Use latest redis Image stream

```
oc import-image --all --confirm -n openshift registry.redhat.io/rhel8/redis-5
```

2. Create image stream and build config

```
oc process -f openshift/build/redis-build.yml \
   -p REDIS_BASE_IMAGE_NAME=redis-5:latest \
   -p REDIS_IMAGE_NAME=redis-ha \
   -p GIT_REPO=https://github.com/eformat/redis-ha.git \
   | oc create -f -
```

(Optional) If using authenticated registry
```
oc create secret generic imagestreamsecret --from-literal=.dockerconfigjson=`oc get secret samples-registry-credentials -n openshift -o jsonpath="{.data['\.dockerconfigjson']}" |base64 -d` --type=kubernetes.io/dockerconfigjson
oc secrets link default imagestreamsecret --for=pull
oc secrets link builder imagestreamsecret
```

3. Start the build

```
oc start-build redis-ha-build
```

## Deployment Configuration

### Persistent

Create a persistent storage, the storage must be **RWX**.

### Create a new persistent storage (RWX)

```yml
cat <<EOF | oc apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: backend-redis-storage
spec:
  accessModes:
  - ReadWriteMany
  resources:
    requests:
      storage: 1Gi
EOF
```

### Bootstrap an initial master/sentinel pod

Create a bootstrap master and sentinels with relative service
```
export REDIS_PREFIX=backend
export REDIS_NAME=${REDIS_PREFIX}-redis

oc process -f openshift/templates/redis-master.yml \
    -p REDIS_SERVICE_PREFIX=${REDIS_PREFIX} \
    -p REDIS_IMAGE=redis-ha:latest \
    -p REDIS_PV=${REDIS_NAME}-storage \
    | oc create -f -
```

### Create a replicated redis slave servers

Create a deployment config for redis slave servers
```
oc process -f openshift/templates/redis-slave.yml \
    -p REDIS_SERVICE_PREFIX=${REDIS_PREFIX} \
    -p REDIS_IMAGE=redis-ha:latest \
    -p REDIS_PV=${REDIS_NAME}-storage \
    | oc create -f -
```

Create a deployment config for redis sentinels
```
oc process -f openshift/templates/redis-sentinel.yml \
    -p REDIS_SERVICE_PREFIX=${REDIS_PREFIX} \
    -p REDIS_IMAGE=redis-ha:latest \
    | oc create -f -
```

### Scale down the bootstrap master

Scale down the original master pod

```
oc scale --replicas=0 dc ${REDIS_NAME}-master
```

### Test

Sentinel
```
# role
for x in $(oc get pods -l app=backend-redis -o name); do echo $x && oc exec ${x##pod/} -- /usr/bin/redis-cli role; done

# scale master bootstrap to zero (sentinel will vote for new master from slaves)
oc scale dc/backend-redis-master --replicas=0

# write to master with some data
oc exec pod/backend-redis-1-xs95h -- /usr/bin/redis-cli hmset employees e000001 'Carmina Chilcote' e000002 'Werner Whobrey' e000003 'Jenna Jarmon' e000004 'Norbert Niswonger' e000004 'Randell Reimers' e000005 'Janay Jacobi' e000006 'Tammara Theobald' e000007 'Margret Michelin' e000008 'Daron Desrosier' e000009 'Raymon Riggenbach'

# read from master/slaves - for clustered redis - use `-c` here
for x in $(oc get pods -l app=backend-redis -o name); do echo $x && oc exec ${x##pod/} -- /usr/bin/redis-cli hvals employees; echo; done

# flush data from writable master
oc exec pod/backend-redis-1-xs95h -- /usr/bin/redis-cli flushall
```

### Failover

### Recommended setup

For a recommended setup that can resist more failures, set the replicas to 5 (default) for Redis and Sentinel.

>
> With 3 sentinels, a maximum of 1 can go down for a failover begin.
>
> With 5 or 6 sentinels, a maximum of 2 can go down for a failover begin.
>
> With 7 sentinels, a maximum of 3 nodes can go down.
>

### Custom settings

|       Environment         |  Default Value   | Note                                                                                                      | 
| ------------------------- | ---------------- | --------------------------------------------------------------------------------------------------------- |
| REDIS_DOWN_AFTER_MILLIS   | 30000            | The time in milliseconds an instance should not be reachable for a Sentinel starting to think it is down  |    
| REDIS_FAILOVER_TIMEOUT    | 180000           | Specifies the failover timeout in milliseconds.                                                           |

For a fast failover under 1 minute

* REDIS_DOWN_AFTER_MILLIS=20000
* REDIS_FAILOVER_TIMEOUT=30000

## Migration from existing single Redis instance

[Migration from existing single redis instance](./management/migrate/README.md)

## Backup and Recovery

[Backup and Recovery](./management/backup/README.md)

## References

* https://redis.io/topics/sentinel
* https://github.com/kubernetes/examples/blob/master/staging/storage/redis/README.md
* https://github.com/sclorg/redis-container/blob/master/README.md
* https://github.com/mjudeikis/redis-openshift/blob/master/README.md
