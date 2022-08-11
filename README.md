# Accessing an HTTP server using Skupper

[![main](https://github.com/skupperproject/skupper-example-http/actions/workflows/main.yaml/badge.svg)](https://github.com/skupperproject/skupper-example-http/actions/workflows/main.yaml)

#### Securely connect to an HTTP server on a remote Kubernetes cluster


This example is part of a [suite of examples][examples] showing the
different ways you can use [Skupper][website] to connect services
across cloud providers, data centers, and edge sites.

[website]: https://skupper.io/
[examples]: https://skupper.io/examples/index.html


#### Contents

* [Overview](#overview)
* [Prerequisites](#prerequisites)
* [Step 1: Configure separate console sessions](#step-1-configure-separate-console-sessions)
* [Step 2: Access your clusters](#step-2-access-your-clusters)
* [Step 3: Set up your namespaces](#step-3-set-up-your-namespaces)
* [Step 4: Install Skupper in your namespaces](#step-4-install-skupper-in-your-namespaces)
* [Step 5: Check the status of your namespaces](#step-5-check-the-status-of-your-namespaces)
* [Step 6: Deploy the HTTP server](#step-6-deploy-the-http-server)
* [Step 7: Expose the HTTP server as a Skupper service](#step-7-expose-the-http-server-as-a-skupper-service)
* [Step 8: Run the HTTP client](#step-8-run-the-http-client)
* [Cleaning up](#cleaning-up)

## Overview

This example shows how you can use Skupper to expose a HTTP server that runs in a different namespace on the service network (VAN).

## Prerequisites


* The `kubectl` command-line tool, version 1.15 or later
  ([installation guide][install-kubectl])

* The `skupper` command-line tool, the latest version ([installation
  guide][install-skupper])

* Access to at least one Kubernetes cluster, from any provider you
  choose

[install-kubectl]: https://kubernetes.io/docs/tasks/tools/install-kubectl/
[install-skupper]: https://skupper.io/install/index.html


## Step 1: Configure separate console sessions

Skupper is designed for use with multiple namespaces, typically on
different clusters.  The `skupper` command uses your
[kubeconfig][kubeconfig] and current context to select the
namespace where it operates.

[kubeconfig]: https://kubernetes.io/docs/concepts/configuration/organize-cluster-access-kubeconfig/

Your kubeconfig is stored in a file in your home directory.  The
`skupper` and `kubectl` commands use the `KUBECONFIG` environment
variable to locate it.

A single kubeconfig supports only one active context per user.
Since you will be using multiple contexts at once in this
exercise, you need to create distinct kubeconfigs.

Start a console session for each of your namespaces.  Set the
`KUBECONFIG` environment variable to a different path in each
session.

_**Console for public:**_

~~~ shell
export KUBECONFIG=~/.kube/config-public
~~~

_**Console for private:**_

~~~ shell
export KUBECONFIG=~/.kube/config-private
~~~

## Step 2: Access your clusters

The methods for accessing your clusters vary by Kubernetes
provider. Find the instructions for your chosen providers and use
them to authenticate and configure access for each console
session.  See the following links for more information:

* [Minikube](https://skupper.io/start/minikube.html)
* [Amazon Elastic Kubernetes Service (EKS)](https://skupper.io/start/eks.html)
* [Azure Kubernetes Service (AKS)](https://skupper.io/start/aks.html)
* [Google Kubernetes Engine (GKE)](https://skupper.io/start/gke.html)
* [IBM Kubernetes Service](https://skupper.io/start/ibmks.html)
* [OpenShift](https://skupper.io/start/openshift.html)
* [More providers](https://kubernetes.io/partners/#kcsp)

## Step 3: Set up your namespaces

Use `kubectl create namespace` to create the namespaces you wish
to use (or use existing namespaces).  Use `kubectl config
set-context` to set the current namespace for each session.

_**Console for public:**_

~~~ shell
kubectl create namespace public
kubectl config set-context --current --namespace public
~~~

_Sample output:_

~~~ console
$ kubectl create namespace public
namespace/public created

$ kubectl config set-context --current --namespace public
Context "minikube" modified.
~~~

_**Console for private:**_

~~~ shell
kubectl create namespace private
kubectl config set-context --current --namespace private
~~~

_Sample output:_

~~~ console
$ kubectl create namespace private
namespace/private created

$ kubectl config set-context --current --namespace private
Context "minikube" modified.
~~~

## Step 4: Install Skupper in your namespaces

The `skupper init` command installs the Skupper router and service
controller in the current namespace.  Run the `skupper init` command
in each namespace.

**Note:** If you are using Minikube, [you need to start `minikube
tunnel`][minikube-tunnel] before you install Skupper.

[minikube-tunnel]: https://skupper.io/start/minikube.html#running-minikube-tunnel

_**Console for public:**_

~~~ shell
skupper init
~~~

_Sample output:_

~~~ console
$ skupper init
Waiting for LoadBalancer IP or hostname...
Skupper is now installed in namespace '@namespace@'.  Use 'skupper status' to get more information.
~~~

## Step 5: Check the status of your namespaces

Use `skupper status` in each console to check that Skupper is
installed.

_**Console for public:**_

~~~ shell
skupper status
~~~

_Sample output:_

~~~ console
$ skupper status
Skupper is enabled for namespace "@namespace@" in interior mode. It is connected to 1 other site. It has 1 exposed service.
The site console url is: <console-url>
The credentials for internal console-auth mode are held in secret: 'skupper-console-users'
~~~

As you move through the steps below, you can use `skupper status` at
any time to check your progress.

## Step 6: Deploy the HTTP server

In the private namespace, use the `kubectl apply` command to
install the server.
You also need to expose that deployment as a Kubernetes service.

_**Console for private:**_

~~~ shell
kubectl apply -f http-server/kubernetes.yaml
kubectl expose deployment/http-server --port 8080 --target-port 80
~~~

_Sample output:_

~~~ console
$ kubectl apply -f http-server/kubernetes.yaml
deployment.apps/http-server created
~~~

## Step 7: Expose the HTTP server as a Skupper service

In the public namespace, use `skupper expose` to expose the
HTTP server running in the private namespace on the Skupper network.

Then, use `kubectl get
service/http-server` to check that the `http-server` service
appears after a moment.

_**Console for public:**_

~~~ shell
skupper expose service http-server.private --port 8080 --address http-server
kubectl get service/http-server
~~~

_Sample output:_

~~~ console
$ skupper expose service http-server.private --port 8080 --address http-server
http-server exposed as http-server

$ kubectl get service/http-server
NAME          TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)    AGE
http-server   ClusterIP   10.100.58.95   <none>        8080/TCP   2s
~~~

## Step 8: Run the HTTP client

In the public namespace, use `kubectl run` to run the HTTP client.

_**Console for public:**_

~~~ shell
kubectl run http-client --attach --rm --image docker.io/library/nginx --restart Never -- curl -sf http://http-server:8080/
~~~

_Sample output:_

~~~ console
$ kubectl run http-client --attach --rm --image docker.io/library/nginx --restart Never -- curl -sf http://http-server:8080/
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
html { color-scheme: light dark; }
body { width: 35em; margin: 0 auto;
font-family: Tahoma, Verdana, Arial, sans-serif; }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
pod "http-client" deleted
~~~

## Cleaning up

To remove Skupper and the other resources from this exercise, use
the following commands.

_**Console for private:**_

~~~ shell
kubectl delete -f http-server/kubernetes.yaml
~~~

_**Console for public:**_

~~~ shell
skupper delete
~~~

## Next steps


Check out the other [examples][examples] on the Skupper website.
