# Why we need to restore status when restoring Crossplane resources?

## Reproduce the issue

1. Setup the environment, create the claim, wait until it is ready:

```shell
kubectl apply -f setup
kubectl apply -f claim.yaml
kubectl wait -f claim.yaml --for condition=ready
```

2. Take backup:

```shell
kubectl get managed -o yaml > managed.yaml
kubectl get composite -o yaml > composite.yaml
kubectl get claim -o yaml > claim.yaml
```

3. Cleanup cluster specific data on all resources, by manually removing the following fields:

```yaml
metadata.creationTimestamp
metadata.generation
metadata.ownerReferences
metadata.resourceVersion
metadata.uid
```

Please note, Velero [removes](https://github.com/vmware-tanzu/velero/blob/a81e049d362557c311cf8615c2c9c8bf77edf969/pkg/restore/restore.go#L2045) these fields before restoration as well.

4. Restore the resources on the target cluster:

```shell
kubectl apply -f setup
kubectl wait provider.pkg provider-nop --for condition=healthy --timeout 2m
kubectl wait functions.pkg function-patch-and-transform --for condition=healthy --timeout 2m

kubectl apply -f managed.yaml
kubectl apply -f composite.yaml
kubectl apply -f claim.yaml
```

5. You will notice that the composite will first lose the reference to the second managed resource, and then it will be
compose another one. At the end, you will have three managed resources instead of two, leaking the old one.
