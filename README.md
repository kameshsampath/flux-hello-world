# Hello World!, the GitOps way

A demo to show case how to deploy the traditional Hello World application using [GitOps](https://www.gitops.tech/). The goal of this demo is to get started with GitOps on your laptops, understand its principles etc.,

For this demo we will be using [FluxCD](https://fluxcd.io/) as the GitOps platform on Kubernetes.

## Tools

- [Task](https://taskfile.dev/)
- [Flux](https://fluxcd.io/flux/cmd/)
- [K3D](https://k3d.io)
- [Docker Desktop](https://www.docker.com/products/docker-desktop/)
- [GitHub CLI](https://github.com/cli/cli)

## Whats  required

- [GitHub PAT](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/creating-a-personal-access-token)
  
- A GitHub user to fork the GitOps Demo Fork

## Fork the demo repo

Run the following command to fork the demo repo under your account,

```shell
gh repo fork <https://github.com/kameshsampath/flux-hello-world>
cd flux-hello-world
```

For the rest of the tutorial this folder will be called as `$GITOPS_DEMO_HOME`.

## Create Cluster

Run the following command to create local kubernetes([k3s](https://k3s.io)) cluster,

```shell
task create_cluster
```

If all went well running the command `kubectl cluster-info` should show a similar output,

```text
Kubernetes control plane is running at https://0.0.0.0:62779
CoreDNS is running at https://0.0.0.0:62779/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy
Metrics-server is running at https://0.0.0.0:62779/api/v1/namespaces/kube-system/services/https:metrics-server:https/proxy
```

## Bootstrap

The demo setup will have the following components,

- FluxCD   - the GitOps platform
- Knative  - the serverless platform to deploy our services
- Weave GitOps OSS - the GitOps UI Dashboard

Run the following command to bootstrap FluxCD and all other infrastructure components,

> ***TIP**: All the commands we run as part of this demo has task associated. E.g. The following bootstrap command can be run using `task bootstrap`

```shell
flux bootstrap github \
  --owner=$GITHUB_USER \
  --repository=flux-hello-world \
  --branch=main \
  --personal \
  --path=./clusters/dev
```

The command instructs flux to hook the repo `$GITHUB_USER/flux-hello-world` and start watching it for changes on branch `main`, path `./clusters/dev`.

>**NOTE**: It will take few mins for bootstrap to complete, depending upon your bandwidth.

Ensure all the required infrastructure components are up and running,

### Knative

```shell
kubectl get pods -n knative serving
```

```shell
NAME                                     READY   STATUS      RESTARTS   AGE
domainmapping-webhook-78d8f55ff-gbvvk    1/1     Running     0          100s
webhook-d44b476b8-lwtnx                  1/1     Running     0          100s
autoscaler-6c94f894f7-wwxbf              1/1     Running     0          100s
activator-56948b7c57-hmx76               1/1     Running     0          100s
controller-8c6b99cb7-v2mfw               1/1     Running     0          100s
net-kourier-controller-c748f7c64-q7cdg   1/1     Running     0          100s
domain-mapping-9fd944b76-8zdk9           1/1     Running     0          100s
default-domain-9l96p                     0/1     Completed   0          100s
```

#### Knative Ingress Check

```shell
kubectl describe configmap/config-network --namespace knative-serving
```

Should display an output like,

```text
...(trimmed for brevity)
ingress-class:
----
kourier.ingress.networking.knative.dev
...
```

#### Knative Domain Check

```shell
kubectl describe configmap/config-domain --namespace knative-serving | less
```

Should display an output like,

```text
...(trimmed for brevity)
Data
====
127.0.0.1.sslip.io:
----
...
```

#### Deploy test app

```shell
cat <<EOF | kubectl apply -f -
apiVersion: serving.knative.dev/v1 # Current version of Knative
kind: Service
metadata:
  name: helloworld-go # The name of the app
  namespace: default # The namespace the app will use
spec:
  template:
    spec:
      containers:
        - image: gcr.io/knative-samples/helloworld-go # The URL to the image of the app
          env:
            - name: TARGET # The environment variable printed out by the sample app
              value: "All set to deploy Hello World, the GitOps way!!!"
EOF
```

Wait for the service to be ready,

```shell
watch kubectl get ksvc
```

It will take few seconds for the service to be ready, a ready service should show an output like,

```text
NAME            URL                                               LATESTCREATED         LATESTREADY           READY   REASON
helloworld-go   http://helloworld-go.default.127.0.0.1.sslip.io   helloworld-go-00001   helloworld-go-00001   True
```

If all went well you can cURL the service <http://helloworld-go.default.127.0.0.1.sslip.io:30080/>, it will return a response like `All set to deploy Hello World, the GitOps way!!!`.

Clean up the test service using the command,

```shell
kubectl delete ksvc helloworld-go
```

### GitOps UI

For this demo we will be using [Weave GitOps OSS](https://github.com/weaveworks/weave-gitops) for UI and Dashboard.

Check if the Weave GitOps OSS dashboard is up,

```shell
kubectl rollout status -n flux-system deploy/weave-gitops --timeout=60s
```

Once its running its accessible via <http://127.0.0.1.sslip.io:30091>. The default user credentials is `admin/flux`.

## Deploy Hello World Application

Check the [README.md](./apps/README.md) on how to deploy the Hello World Application using GitOps.

