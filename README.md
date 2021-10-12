# RabbitMQ on Kubernetes

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
```
Go to htttp://localhost:8080 <br/>
Username: `guest` <br/>
Password: `guest` <br/>

# Automatic Synchronization

https://www.rabbitmq.com/ha.html#unsynchronised-mirrors

```
rabbitmqctl set_policy ha-fed \
    ".*" '{"federation-upstream-set":"all", "ha-sync-mode":"automatic", "ha-mode":"nodes", "ha-params":["rabbit@rabbitmq-0.rabbitmq.rabbits.svc.cluster.local","rabbit@rabbitmq-1.rabbitmq.rabbits.svc.cluster.local","rabbit@rabbitmq-2.rabbitmq.rabbits.svc.cluster.local"]}' \
    --priority 1 \
    --apply-to queues
```

