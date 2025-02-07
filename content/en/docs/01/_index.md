```yaml
---
kind: Namespace
apiVersion: v1
metadata:
  name: argocd123
---
apiVersion: operators.coreos.com/v1
kind: OperatorGroup
metadata:
  name: argocd
  namespace: argocd123
spec:
  targetNamespaces:
    - argocd123
---
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: "argocd-operator"
  namespace: "argocd123"
  annotations: 
    olm.operatorGroup: argocd
spec:
  channel: "alpha" 
  installPlanApproval: "Automatic"
  source: "community-operators" 
  sourceNamespace: "openshift-marketplace"
  name: "argocd-operator"
---
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: "prometheus"
  namespace: "argocd123"
  annotations: 
    olm.operatorGroup: argocd
spec:
  channel: "beta" 
  installPlanApproval: "Automatic"
  source: "community-operators" 
  sourceNamespace: "openshift-marketplace"
  name: "prometheus"
---
apiVersion: argoproj.io/v1alpha1
kind: ArgoCD
metadata:
  name: demolab-argocd
  namespace: argocd123
spec:
  # version: v1.5.5
  usersAnonymousEnabled: true
  ha:
    enabled: true
  dex:
    image: quay.io/redhat-cop/dex
    version: v2.22.0-openshift
    # version: v2.21.0
    openShiftOAuth: true
  grafana:
    enabled: true
    route: 
      enabled: true
  prometheus:
    enabled: true
    route: 
      enabled: true
  rbac:
    defaultPolicy: 'role:readonly'
    policy: |
      g, system:cluster-admins, role:admin
      g, argocd-users, role:readonly
      g, argocd-admins, role:admin
      g, tektonbot, role:admin
    scopes: '[groups]'
  applicationInstanceLabelKey: paolocarta.me/argoapp
  repositories: |
    - url: https://github.com/paolocarta/ocp-gitops-argocd.git
  server:
    enabled: true
    route: 
      enabled: true
    autoscale:
      enabled: true
      hpa:
        maxReplicas: 3
        minReplicas: 2
        scaleTargetRef:
          apiVersion: extensions/v1beta1
          kind: Deployment
          name: demolab-argocd-server
        targetCPUUtilizationPercentage: 50
  resourceExclusions: |
    - apiGroups:
      - tekton.dev
      kinds:
      - PipelineRun
      - TaskRun
      clusters:
      - "https://kubernetes.default.svc"
---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: cluster-admin-argocd
    # generateName: cluster-admin-argocd
subjects:
  - kind: ServiceAccount
    name: argocd-application-controller
    namespace: argocd123
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
---
apiVersion: security.openshift.io/v1
kind: SecurityContextConstraints
metadata:
  annotations:
    kubernetes.io/description: anyuid provides all features of the restricted SCC
      but allows users to run with any UID and any GID.
  name: anyuid
allowHostDirVolumePlugin: false
allowHostIPC: false
allowHostNetwork: false
allowHostPID: false
allowHostPorts: false
allowPrivilegeEscalation: true
allowPrivilegedContainer: false
allowedCapabilities: []
defaultAddCapabilities: []
fsGroup:
  type: RunAsAny
groups:
- system:cluster-admins
priority: 10
readOnlyRootFilesystem: false
requiredDropCapabilities:
- MKNOD
runAsUser:
  type: RunAsAny
seLinuxContext:
  type: MustRunAs
supplementalGroups:
  type: RunAsAny
users:
- system:serviceaccount:argocd123:argocd-redis-ha
volumes:
- configMap
- downwardAPI
- emptyDir
- persistentVolumeClaim
- projected
- secret
---

```




```yaml
---
kind: argoproj.io/v1alpha1
metadata:
  name: argocd
  namespace: argocd123
spec:
  server:
    route:
      enabled: true
```

---
title: "1. Getting started"
weight: 1
sectionnumber: 1
---


## Task   .1: Web IDE

The first thing we're going to do is to explore our lab environment and get in touch with the different components.

The namespace with the name corresponding to your username is going to be used for all the hands-on labs. And you will be using the `argocd tool` or the ArgoCD webconsole, to verify what resources and objects Argo CD created for you.

{{% alert title="Note" color="primary" %}}You can also use your local installation of the cli tools. Make sure you completed [the setup](../../setup/) before you continue with this lab.{{% /alert %}}

{{% alert title="Note" color="primary" %}}The URL and Credentials to the Web IDE will provided by the teacher. Use Chrome for the best experience.{{% /alert %}}


Once you're successfully logged into the web IDE open a new Terminal by hitting `CTRL + SHIFT + ¨` or clicking the Menu button --> Terminal --> new Terminal and check the installed {{% param cliToolName %}}version by executing the following command:

```bash
{{% param cliToolName %}} version
```

The Web IDE Pod consists of the following tools:

* oc
* kubectl
* kustomize
* helm
* kubectx
* kubens
* tekton cli
* odo
* argocd

The files in the home directory under `/home/project` are stored in a persistence volume, so please make sure to store all your persistence data in this directory.


### Task   .1.1: Local Workspace Directory

During the lab, you’ll be using local files (eg. YAML resources) which will be applied in your lab project.

Create a new folder for your `<workspace>` in your Web IDE  (for example `argocd-training` under `/home/project/argocd-training`). Either you can create it with `right-mouse-click -> New Folder` or in the Web IDE terminal

```bash
mkdir argocd-training && cd argocd-training
```


### Task   .1.2: Login on ArgoCD using argocd CLI

You can access Argo CD via Web UI (Credentials are provided by your teacher) or using the CLI. The Argo CD CLI Tool is already installed on the web IDE.

```bash
argocd login {{% param argoCdUrl %}} --grpc-web --username <username>
```
{{% onlyWhen openshift %}}


### Task   .1.3: Lab Setup


Most of the labs will be done inside the {{% param distroName %}} project with your username. Verify that your oc tool is configured to point to the right project:


```s
oc project
```


```
Using project "<username>" on server "https://<theClusterAPIURL>".
```

The returned project name should correspond to your username.
{{% /onlyWhen  %}}


## Task   .2: Argo CD CLI

The [Argo CD CLI](https://argoproj.github.io/argo-cd/cli_installation/) is a powerful tool to manage Argo CD and different applications. It's a self contained binary written in Go and available for Linux, Mac OS and Windows. Thanks to the fact that the CLI is implemented in Go, it can be easily integrated into scripts and build servers for automation purposes.


### Task   .2: Getting familiar with the CLI

Print out the help of the CLI by typing

```bash
argocd --help
```

You will see a list with the available commands and flags. If you prefer to browse the manual in the browser you'll find it in the [online documentation](https://argoproj.github.io/argo-cd/user-guide/commands/argocd/).

```
argocd controls a Argo CD server

Usage:
  argocd [flags]
  argocd [command]

Available Commands:
  account     Manage account settings
  admin       Contains a set of commands useful for Argo CD administrators and requires direct Kubernetes access
  app         Manage applications
  cert        Manage repository certificates and SSH known hosts entries
  cluster     Manage cluster credentials
  completion  output shell completion code for the specified shell (bash or zsh)
  context     Switch between contexts
  gpg         Manage GPG keys used for signature verification
  help        Help about any command
  login       Log in to Argo CD
  logout      Log out from Argo CD
  proj        Manage projects
  relogin     Refresh an expired authenticate token
  repo        Manage repository connection parameters
  repocreds   Manage repository connection parameters
  version     Print version information

Flags:
      --auth-token string               Authentication token
      --client-crt string               Client certificate file
      --client-crt-key string           Client certificate key file
      --config string                   Path to Argo CD config (default "/home/bbuehlmann/.argocd/config")
      --core                            If set to true then CLI talks directly to Kubernetes instead of talking to Argo CD API server
      --grpc-web                        Enables gRPC-web protocol. Useful if Argo CD server is behind proxy which does not support HTTP2.
      --grpc-web-root-path string       Enables gRPC-web protocol. Useful if Argo CD server is behind proxy which does not support HTTP2. Set web root.
  -H, --header strings                  Sets additional header to all requests made by Argo CD CLI. (Can be repeated multiple times to add multiple headers, also supports comma separated headers)
  -h, --help                            help for argocd
      --http-retry-max int              Maximum number of retries to establish http connection to Argo CD server
      --insecure                        Skip server certificate and domain verification
      --logformat string                Set the logging format. One of: text|json (default "text")
      --loglevel string                 Set the logging level. One of: debug|info|warn|error (default "info")
      --plaintext                       Disable TLS
      --port-forward                    Connect to a random argocd-server port using port forwarding
      --port-forward-namespace string   Namespace name which should be used for port forwarding
      --server string                   Argo CD server address
      --server-crt string               Server certificate file
```

The `--help` flag is available for every command and subcommand of the CLI. Beside the documentation for every flag and subcommand in the current context, it prints out example command lines for the most common use cases.

Now get the help of the `app create` subcommand and find the examples and documentation of the flags.

```bash
argocd app create --help
```


### Task   .2: Autocompletion

{{% alert title="Note" color="primary" %}}This step is only needed, when you're not working with the Web IDE we've provided. The autocompletion is already installed in the Web IDE{{% /alert %}}

A productivity booster when working with the CLI is the autocompletion feature. It can be used for `bash` and `zsh` shells. You can enable the autocompletion for the current `bash` with the following command:

```bash
source <(argocd completion bash)
```

After typing `argocd` you can autocomplete the subcommands with a double tap the tabulator key. This works even for deployed artifacts on the cluster: A double tab after `argocd app get` lists all defined Argo CD applications.

To install autocompletion permanently you can add the following command in the `~/.bashrc` file.

```bash
echo "source <(argocd completion bash)" >> ~/.bashrc
source ~/.bashrc
```

Find further information in the [official documentation](https://argoproj.github.io/argo-cd/user-guide/commands/argocd_completion/)
