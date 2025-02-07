---
title: "9.1 Orphaned Resources"
weight: 901
sectionnumber: 9.1
---

This lab contains demonstrates how to find orphaned top-level resources with Argo CD. Orphaned resources are not managed by Argo CD and could be potentially removed from cluster.


## Task   .1: Create application and project

```bash
argocd app create argo-$STUDENT --repo https://github.com/alexandrust88/argocd-training-examples  --path 'example-app' --dest-server https://kubernetes.default.svc --dest-namespace $STUDENT
argocd app sync argo-$STUDENT
```

Create new Argo CD project without restrictions for Git source repository (--src) nor destination cluster/namespace (--dest)
```bash
argocd proj create --src "*" --dest "*,*" apps-$STUDENT
```

Enable visualization and monitoring of Orphaned Resources for the newly created project `apps-<username>`
```bash
argocd proj set apps-$STUDENT --orphaned-resources --orphaned-resources-warn
```

{{% alert title="Note" color="primary" %}}
The flag `--orphaned-resources` enables the determinability of orphaned resources in Argo CD. After a refresh you will see them in the user interface on the project when selecting the checkbox _Orphaned Resources_.
With the flag `--orphaned-resources-warn` enabled, for each Argo CD application with orphaned resources in the destination namespace a warning will be shown in the user interface.
{{% /alert %}}


## Task   .2: Assign application to project

Assign application to newly created project
```bash
argocd app set argo-$STUDENT --project apps-$STUDENT
```

Ensure that the application is now assigned to the new project `apps-<username>`
```bash
argocd app get argo-$STUDENT
```

Refresh the application
```bash
argocd app get --refresh argo-$STUDENT
```


## Task   .3: Create orphaned resource

Now create the orphan service `black-hole` in the same target namespace the Argo CD application has:

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  name: black-hole
spec:
  ports:
  - port: 1234
    targetPort: 1234
EOF
```

{{% alert title="Note" color="primary" %}}
This service will be detected as orphaned resource because it is not managed by Argo CD. All resources which are managed by Argo CD are marked with the label `app.kubernetes.io/instance` per default. The key of the label can be changed with the setting `application.instanceLabelKey`. See [documentation](https://argoproj.github.io/argo-cd/faq/#why-is-my-app-out-of-sync-even-after-syncing) for further details.
{{% /alert %}}


Print all resources
```bash
argocd app resources argo-$STUDENT
```

You see in the output that the manually created service `black-hole` is marked as orphaned:
```
GROUP  KIND        NAMESPACE    NAME            ORPHANED
       Service     <username>   simple-example  No
apps   Deployment  <username>   simple-example  No
       Service     <username>   black-hole      Yes
```

When viewing the details of the application you will see the warning about the orphaned resource
```bash
argocd app get --refresh argo-$STUDENT
```

```
...
CONDITION                MESSAGE                               LAST TRANSITION
OrphanedResourceWarning  Application has 1 orphaned resources  2021-09-02 16:20:36 +0200 CEST
...
```


## Task   .4: Housekeeping

Clean up the resources created in this lab

```bash
argocd app delete argo-$STUDENT -y
argocd proj delete apps-$STUDENT
```

Find more detailed information about [Orphaned Resources in the docs](https://argoproj.github.io/argo-cd/user-guide/orphaned-resources/).
