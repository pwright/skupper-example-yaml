# Skupper Hello World using YAML

[![main](https://github.com/ssorj/skupper-example-yaml/actions/workflows/main.yaml/badge.svg)](https://github.com/ssorj/skupper-example-yaml/actions/workflows/main.yaml)

#### A minimal HTTP application deployed across Kubernetes clusters using Skupper

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
* [Step 3: Install Skupper in your clusters](#step-3-install-skupper-in-your-clusters)
* [Step 4: Set up your namespaces](#step-4-set-up-your-namespaces)
* [Step 5: Apply your YAML resources](#step-5-apply-your-yaml-resources)
* [Step 6: Link your namespaces](#step-6-link-your-namespaces)
* [Step 7: Test the application](#step-7-test-the-application)
* [Accessing the web console](#accessing-the-web-console)
* [Cleaning up](#cleaning-up)
* [About this example](#about-this-example)

## Overview

This example is a variant of [Skupper Hello World][hello-world] that
is deployed using YAML resource definitions instead of imperative
commands.

It contains two services:

* A backend service that exposes an `/api/hello` endpoint.  It
  returns greetings of the form `Hi, <your-name>.  I am <my-name>
  (<pod-name>)`.

* A frontend service that sends greetings to the backend and
  fetches new greetings in response.

The two services run in two different clusters.  The frontend runs
in a namespace on cluster 1 called West, and the backend runs in a
namespace on cluster 2 called East.

<img src="images/entities.svg" width="640"/>

Skupper enables you to place the backend in one cluster and the
frontend in another and maintain connectivity between the two
services without exposing the backend to the public internet.

[hello-world]: https://github.com/skupperproject/skupper-example-hello-world

## Prerequisites

* The `kubectl` command-line tool, version 1.15 or later
  ([installation guide][install-kubectl])

* Access to at least one Kubernetes cluster, from [any provider you
  choose][kube-providers]

[install-kubectl]: https://kubernetes.io/docs/tasks/tools/install-kubectl/
[kube-providers]: https://skupper.io/start/kubernetes.html

## Step 1: Configure separate console sessions

Skupper is designed for use with multiple namespaces, usually on
different clusters.  The `skupper` and `kubectl` commands use your
[kubeconfig][kubeconfig] and current context to select the
namespace where they operate.

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

_**Console for West:**_

~~~ shell
export KUBECONFIG=~/.kube/config-west
~~~

_**Console for East:**_

~~~ shell
export KUBECONFIG=~/.kube/config-east
~~~

## Step 2: Access your clusters

The procedure for accessing a Kubernetes cluster varies by
provider. [Find the instructions for your chosen
provider][kube-providers] and use them to authenticate and
configure access for each console session.

[kube-providers]: https://skupper.io/start/kubernetes.html

## Step 3: Install Skupper in your clusters

Use the `kubectl apply` command to install the Skupper
controller in each cluster.

_**Console for West:**_

~~~ shell
kubectl apply -f skupper.yaml
~~~

_Sample output:_

~~~ console
$ kubectl apply -f skupper.yaml
namespace/skupper-site-controller created
serviceaccount/skupper-site-controller created
clusterrole.rbac.authorization.k8s.io/skupper-site-controller created
clusterrolebinding.rbac.authorization.k8s.io/skupper-site-controller created
deployment.apps/skupper-site-controller created
~~~

_**Console for East:**_

~~~ shell
kubectl apply -f skupper.yaml
~~~

_Sample output:_

~~~ console
$ kubectl apply -f skupper.yaml
namespace/skupper-site-controller created
serviceaccount/skupper-site-controller created
clusterrole.rbac.authorization.k8s.io/skupper-site-controller created
clusterrolebinding.rbac.authorization.k8s.io/skupper-site-controller created
deployment.apps/skupper-site-controller created
~~~

## Step 4: Set up your namespaces

Use `kubectl create namespace` to create the namespaces you wish
to use (or use existing namespaces).  Use `kubectl config
set-context` to set the current namespace for each session.

_**Console for West:**_

~~~ shell
kubectl create namespace west
kubectl config set-context --current --namespace west
~~~

_**Console for East:**_

~~~ shell
kubectl create namespace east
kubectl config set-context --current --namespace east
~~~

## Step 5: Apply your YAML resources

To configure our example sites and service bindings, we are
using the following resources:

West:

* [site.yaml](west/site.yaml) - Skupper configuration for West
* [frontend.yaml](west/frontend.yaml) - The Hello World frontend

East:

* [site.yaml](east/site.yaml) - Skupper configuration for East
* [backend.yaml](east/backend.yaml) - The Hello World backend

Let's look at some of these resources in more detail.

#### Resources in West

The `site` ConfigMap defines a Skupper site for its associated
Kubernetes namespace.  This is where you set site configuration
options.  We are setting the `console` and `flow-collector`
options here in order to enable the console.  See the [config
reference][config] for more information.

[config]: https://skupper.io/docs/declarative/index.html

[site.yaml](west/site.yaml):

~~~ yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: skupper-site
data:
  name: west
  console: "true"
  flow-collector: "true"
~~~

#### Resources in East

Like the one for West, here is the Skupper site definition for
the East.  It includes the `ingress: "false"` setting since no
ingress is required at this site for the Hello World example.

[site.yaml](east/site.yaml):

~~~ yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: skupper-site
data:
  name: east
  ingress: "false"
~~~

In East, the `backend` deployment has an annotation named
`skupper.io/proxy` with the value `tcp`.  This tells Skupper to
expose the backend on the Skupper network.  As a consequence,
the frontend in West will be able to see the backend and call
its API.

[backend.yaml](east/backend.yaml):

<pre>apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
  labels:
    app: backend
  <b>annotations:
    skupper.io/proxy: tcp</b>
spec:
  selector:
    matchLabels:
      app: backend
  replicas: 3
  template:
    metadata:
      labels:
        app: backend
    spec:
      containers:
        - name: backend
          image: quay.io/skupper/hello-world-backend
          ports:
            - containerPort: 8080</pre>

Now we're ready to apply everything.  Use the `kubectl apply`
command with the resource definitions for each site.

**Note:** If you are using Minikube, [you need to start
`minikube tunnel`][minikube-tunnel] before you create the
Skupper sites.

[minikube-tunnel]: https://skupper.io/start/minikube.html#running-minikube-tunnel

_**Console for West:**_

~~~ shell
kubectl apply -f west/site.yaml -f west/frontend.yaml
~~~

_Sample output:_

~~~ console
$ kubectl apply -f west/site.yaml -f west/frontend.yaml
configmap/skupper-site created
deployment.apps/frontend created
service/frontend created
~~~

_**Console for East:**_

~~~ shell
kubectl apply -f east/site.yaml -f east/backend.yaml
~~~

_Sample output:_

~~~ console
$ kubectl apply -f east/site.yaml -f east/backend.yaml
configmap/skupper-site created
deployment.apps/backend created
~~~

## Step 6: Link your namespaces

You can configure sites and service bindings declaratively, but
linking sites is different.  To create a link, you must have the
authentication secret and connection details of the remote site.
Since these cannot be known in advance, linking must be
procedural.

**Note:** There are several ways to automate the generation and
distribution of tokens across sites, using for example Ansible,
Backstage, or Vault. <!-- See [Token distribution]() for more
information. -->

This example uses the Skupper command line tool to generate the
secret token in West and create the link in East.

To install the Skupper command:

~~~ shell
curl https://skupper.io/install.sh | sh
~~~

For more installation options, see [Installing
Skupper][install].

Once the command is installed, use `skupper token create` in
West to generate the token.  Then, use `skupper link create` in
East to create a link.

[install]: https://skupper.io/install/index.html

_**Console for West:**_

~~~ shell
skupper token create ~/secret.token
~~~

_Sample output:_

~~~ console
$ skupper token create ~/secret.token
Token written to ~/secret.token
~~~

_**Console for East:**_

~~~ shell
skupper link create ~/secret.token
~~~

_Sample output:_

~~~ console
$ skupper link create ~/secret.token
Site configured to link to https://10.105.193.154:8081/ed9c37f6-d78a-11ec-a8c7-04421a4c5042 (name=link1)
Check the status of the link using 'skupper link status'.
~~~

If your console sessions are on different machines, you may need
to use `scp` or a similar tool to transfer the token securely.  By
default, tokens expire after a single use or 15 minutes after
creation.

## Step 7: Test the application

Now we're ready to try it out.  Use `kubectl get service/frontend`
to look up the external IP of the frontend service.  Then use
`curl` or a similar tool to request the `/api/health` endpoint at
that address.

**Note:** The `<external-ip>` field in the following commands is a
placeholder.  The actual value is an IP address.

_**Console for West:**_

~~~ shell
kubectl get service/frontend
curl http://<external-ip>:8080/api/health
~~~

_Sample output:_

~~~ console
$ kubectl get service/frontend
NAME       TYPE           CLUSTER-IP      EXTERNAL-IP     PORT(S)          AGE
frontend   LoadBalancer   10.103.232.28   <external-ip>   8080:30407/TCP   15s

$ curl http://<external-ip>:8080/api/health
OK
~~~

If everything is in order, you can now access the web interface by
navigating to `http://<external-ip>:8080/` in your browser.

## Accessing the web console

Skupper includes a web console you can use to view the application
network.  To access it, use `skupper status` to look up the URL of
the web console.  Then use `kubectl get
secret/skupper-console-users` to look up the console admin
password.

**Note:** The `<console-url>` and `<password>` fields in the
following output are placeholders.  The actual values are specific
to your environment.

_**Console for West:**_

~~~ shell
skupper status
kubectl get secret/skupper-console-users -o jsonpath={.data.admin} | base64 -d
~~~

_Sample output:_

~~~ console
$ skupper status
Skupper is enabled for namespace "west". It is connected to 1 other site. It has 1 exposed service.
The site console url is: <console-url>
The credentials for internal console-auth mode are held in secret: 'skupper-console-users'

$ kubectl get secret/skupper-console-users -o jsonpath={.data.admin} | base64 -d
<password>
~~~

Navigate to `<console-url>` in your browser.  When prompted, log
in as user `admin` and enter the password.

## Cleaning up

To remove Skupper and the other resources from this exercise, use
the following commands.

_**Console for West:**_

~~~ shell
kubectl delete -f west/site.yaml -f west/frontend.yaml
kubectl delete -f skupper.yaml
~~~

_**Console for East:**_

~~~ shell
kubectl delete -f east/site.yaml -f east/backend.yaml
kubectl delete -f skupper.yaml
~~~

## Next steps

Check out the other [examples][examples] on the Skupper website.

## About this example

This example was produced using [Skewer][skewer], a library for
documenting and testing Skupper examples.

[skewer]: https://github.com/skupperproject/skewer

Skewer provides utility functions for generating the README and
running the example steps.  Use the `./plano` command in the project
root to see what is available.

To quickly stand up the example using Minikube, try the `./plano demo`
command.
