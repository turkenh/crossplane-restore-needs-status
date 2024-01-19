# Why we need to restore status when restoring Crossplane resources?

I would like to restore Crossplane state that I have backed up from another Crossplane installation. Roughly, I am
following the below steps:

1. Restore all providers/configurations.
2. Restore all managed resources with paused annotations.
3. Restore all composites/claims with paused annotations.
4. Remove paused annotation from claims/composites.

After the 4th step, I expect all composites to take ownership of composed resources using the `spec.resourceRefs` field. 
However, if the composition logic includes some dependency on the status of a composed resource, during its absence, it
does not output an already existing composed resource, and the composition pipeline decides to remove its references
from `spec.resourceRef` assuming the pipeline decided to delete it. So, before removing paused annotations, I need to
make sure all data including the status restored. Otherwise, we will lose the reference to the previously composed
resources eventually leaking them.

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
