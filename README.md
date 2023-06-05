# Skupper Hello World with YAML

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
* [Step 3: Set up your namespaces](#step-3-set-up-your-namespaces)
* [Step 4: Apply YAML](#step-4-apply-yaml)
* [Step 5: Link your namespaces](#step-5-link-your-namespaces)
* [Step 6: Special extra step](#step-6-special-extra-step)
* [Step 7: Test the application](#step-7-test-the-application)
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

Skupper enables you place the backend in one cluster and the
frontend in another and maintain connectivity between the two
services without exposing the backend to the public internet.

<img src="images/entities.svg" width="640"/>

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

_**Console for west:**_

~~~ shell
export KUBECONFIG=~/.kube/config-west
~~~

_**Console for east:**_

~~~ shell
export KUBECONFIG=~/.kube/config-east
~~~

## Step 2: Access your clusters

The procedure for accessing a Kubernetes cluster varies by
provider. [Find the instructions for your chosen
provider][kube-providers] and use them to authenticate and
configure access for each console session.

[kube-providers]: https://skupper.io/start/kubernetes.html

## Step 3: Set up your namespaces

Use `kubectl create namespace` to create the namespaces you wish
to use (or use existing namespaces).  Use `kubectl config
set-context` to set the current namespace for each session.

_**Console for west:**_

~~~ shell
kubectl create namespace west
kubectl config set-context --current --namespace west
~~~

_**Console for east:**_

~~~ shell
kubectl create namespace east
kubectl config set-context --current --namespace east
~~~

## Step 4: Apply YAML

XXX

_**Console for west:**_

~~~ shell
kubectl apply -f install.yaml -f west/frontend.yaml -f west/site.yaml -f west/listener.yaml
~~~

_**Console for east:**_

~~~ shell
kubectl apply -f install.yaml -f east/backend.yaml -f east/site.yaml -f east/connector.yaml
~~~

## Step 5: Link your namespaces

XXX

On Linux or Mac, you can use the install script (inspect it
[here][install-script]) to download and extract the command:

~~~ shell
curl https://skupper.io/install.sh | sh
~~~

For Windows and other installation options, see [Installing
Skupper][install-docs].

First, use `skupper token create` in one namespace to generate the
token.  Then, use `skupper link create` in the other to create a
link.

[install-script]: https://github.com/skupperproject/skupper-website/blob/main/docs/install.sh
[install-docs]: https://skupper.io/install/index.html

_**Console for west:**_

~~~ shell
skupper token create ~/secret.token
~~~

_Sample output:_

~~~ console
$ skupper token create ~/secret.token
Token written to ~/secret.token
~~~

_**Console for east:**_

~~~ shell
skupper link create ~/secret.token
~~~

_Sample output:_

~~~ console
$ skupper link create ~/secret.token
Site configured to link to https://10.105.193.154:8081/ed9c37f6-d78a-11ec-a8c7-04421a4c5042 (name=link1)
Check the status of the link using 'skupper link status'.
~~~

## Step 6: Special extra step

This will disappear when we have the new service binding YAML.

_**Console for east:**_

~~~ shell
skupper expose deployment/backend
~~~

## Step 7: Test the application

Now we're ready to try it out.  Use `kubectl get service/frontend`
to look up the external IP of the frontend service.  Then use
`curl` or a similar tool to request the `/api/health` endpoint at
that address.

**Note:** The `<external-ip>` field in the following commands is a
placeholder.  The actual value is an IP address.

_**Console for west:**_

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

## Cleaning up

To remove Skupper and the other resources from this exercise, use
the following commands.

_**Console for west:**_

~~~ shell
kubectl delete -f install.yaml -f west/frontend.yaml -f west/site.yaml -f west/listener.yaml
~~~

_**Console for east:**_

~~~ shell
kubectl delete -f install.yaml -f east/backend.yaml -f east/site.yaml -f east/connector.yaml
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
