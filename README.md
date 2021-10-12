# RabbitMQ on Kubernetes

## Motivation

## Clustering
https://www.rabbitmq.com/clustering.html

## Cluster Operator for Kubernetes
https://www.rabbitmq.com/kubernetes/operator/operator-overview.html

## Deploying RabbitMQ to Kubernetes: What's involved?
https://blog.rabbitmq.com/posts/2020/08/deploying-rabbitmq-to-kubernetes-whats-involved/

## Notes 

What is Replicated?
All data/state required for the operation of a RabbitMQ broker is replicated across all nodes except the message queues, which by default reside on one node, though they are visible and reachable from all nodes. 
There aare 2 ways to replicate queues across nodes in a cluster
(1) classic queue mirroring - this example uses classic queue mirroring
(2) quorum queues

## Nodes to Nodes authentication : the Erlang Cookie
RabbitMQ nodes and CLI tools (e.g. rabbitmqctl) use a cookie to determine whether they are allowed to communicate with each other. For two nodes to be able to communicate they must have the same shared secret called the Erlang cookie. The cookie is just a string of alphanumeric characters up to 255 characters in size. It is usually stored in a local file. The file must be only accessible to the owner (e.g. have UNIX permissions of 600 or similar). Every cluster node must have the same cookie.

## how to grab existing erlang cookie
```
docker exec -it rabbit-1 cat /var/lib/rabbitmq/.erlang.cookie
```

## roles based security 
Pods needs access to the api server. RBAC is a method of regulating access to computer or network resources based on the roles of individual users. RBAC authorization uses the rbac.authorization.k8s.io API group to drive authorization decisions, allowing you to dynamically configure policies through the Kubernetes API. To enable RBAC, start the API server with the --authorization-mode flag set to a comma-separated list that includes RBAC; for example:
https://kubernetes.io/docs/reference/access-authn-authz/rbac/

ServiceAccount
https://kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/

in yaml, create ServiceAccount, create Role, create RoleBinding

## service discovery 
for service discovery I am using 3 plugins
1) rabbitmq_federation - federate and synchronize queues and messages across instances
2) rabbitmq_management - user interface and dashboard
3) rabbitmq_peer_discovery_k8s - peer discovery for kubernetes

## plugins
```
cluster_formation.peer_discovery_backend  = rabbit_peer_discovery_k8s # use peer discovery plugin
cluster_formation.k8s.host = kubernetes.default.svc.cluster.local     # location of api server
cluster_formation.k8s.address_type = hostname # hostname or ip
cluster_formation.node_cleanup.only_log_warning = true
```

## statefulset
every pod gets their own persistence volume and storage class
specify the accoutn name that the pods can use to communicate with the api server
persistent volumes 

## hack
create writable mount pouints for rmq. see lines 19 thru 28. busybox container.

## headless service. every pod gets a fqdn
```
apiVersion: v1
kind: Service
metadata:
  name: rabbitmq
spec:
  clusterIP: None
  ports:
  - port: 4369
    targetPort: 4369
    name: discovery
  - port: 5672
    targetPort: 5672
    name: amqp
  selector:
    app: rabbitmq
```

## port forward so that http://localhost:8080
```
kubectl -n rabbits port-forward rabbitmq-0 8080:5672 guest/guest
```

## mirroring
https://www.rabbitmq.com/ha.html#ways-to-configure


## Namespace - optional step

```
kubectl create ns rabbits
```

## Storage Class

```
kubectl get storageclass
NAME                 PROVISIONER             RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
standard (default)   rancher.io/local-path   Delete          WaitForFirstConsumer   false                  84s
```

## Deployment - uses namespace created from step 1

```
kubectl apply -n rabbits -f .\kubernetes\rabbit-rbac.yaml
kubectl apply -n rabbits -f .\kubernetes\rabbit-configmap.yaml
kubectl apply -n rabbits -f .\kubernetes\rabbit-secret.yaml
kubectl apply -n rabbits -f .\kubernetes\rabbit-statefulset.yaml
```

## Access the UI - uses namespace created from step 1

```
kubectl -n rabbits port-forward rabbitmq-0 8080:15672

Go to htttp://localhost:8080 <br/>
Username: `guest` <br/>
Password: `guest` <br/>
```

## Automatic Synchronization

https://www.rabbitmq.com/ha.html#unsynchronised-mirrors

```
rabbitmqctl set_policy ha-fed \
    ".*" '{"federation-upstream-set":"all", "ha-sync-mode":"automatic", "ha-mode":"nodes", "ha-params":["rabbit@rabbitmq-0.rabbitmq.rabbits.svc.cluster.local","rabbit@rabbitmq-1.rabbitmq.rabbits.svc.cluster.local","rabbit@rabbitmq-2.rabbitmq.rabbits.svc.cluster.local"]}' \
    --priority 1 \
    --apply-to queues
```

