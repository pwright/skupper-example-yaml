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
* [Step 3: Apply your YAML resources](#step-3-apply-your-yaml-resources)
* [Step 4: Link your namespaces](#step-4-link-your-namespaces)
* [Step 5: Test the application](#step-5-test-the-application)
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

## Step 3: Apply your YAML resources

To configure our example sites and service bindings, we are
using the following resources:

West:

* [frontend.yml](west/frontend.yaml) - The Hello World frontend
* [skupper.yaml](west/skupper.yaml) - The Skupper controller
* [site.yaml](west/site.yaml) - Configuration for site `west`
* [listener.yaml](west/listener.yaml) - The listener for the `backend` service

East:

* [backend.yaml](east/backend.yaml) - The Hello World backend
* [skupper.yaml](east/skupper.yaml) - The Skupper controller
* [site.yaml](east/site.yaml) - Configuration for site `east`
* [connector.yaml](east/connector.yaml) - The connector for the `backend` service

Let's look at some of these resources in more detail.

#### Resources in west

The `site` ConfigMap defines a Skupper site for its associated
Kubernetes namespace.  This is where you set site configuration
options.  See the [config reference][config] for more
information.

[config]: https://github.com/ssorj/refdog

[site.yaml](west/site.yaml):

~~~ yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: skupper-site
  namespace: west
  labels:
    skupper.io/type: site
data:
  name: west
~~~

The Hello World example has a frontend workload in west that
sends HTTP requests to a backend service.  To make that service
available in west, create a `listener` resource with
`routing-key: backend:http`.  Connections to the listener are
routed to connectors in remote sites with matching routing keys.

[listener.yaml](west/listener.yaml):

~~~ yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: skupper-listener-backend
  namespace: west
  labels:
    skupper.io/type: listener
data:
  routing-key: backend:http
  hostname: backend
  port: 8080
~~~

#### Resources in east

Like the one for `west`, here is the Skupper site definition for
the `east` namespace.  It includes the `ingress: none` setting
since no ingress inquired at this site for the Hello World
example.

[site.yaml](east/site.yaml):

~~~ yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: skupper-site
  namespace: east
  labels:
    skupper.io/type: site
data:
  name: east
  ingress: none
~~~

We have the `listener` defined.  Now we need the corresponding
`connector`.  It has the same routing key and includes a pod
selector for identifying the target workload.

[connector.yaml](east/connector.yaml):

~~~ yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: skupper-connector-backend
  namespace: east
  labels:
    skupper.io/type: connector
data:
  routing-key: backend:http
  port: 8080
  selector: app=backend
~~~

Now we're ready to apply everything.  Use `kubectl apply`
command with the resource definitions for each site.

_**Console for west:**_

~~~ shell
kubectl apply -f west/frontend.yaml -f west/skupper.yaml -f west/site.yaml -f west/listener.yaml
~~~

_Sample output:_

~~~ console
$ kubectl apply -f west/frontend.yaml -f west/skupper.yaml -f west/site.yaml -f west/listener.yaml
namespace/west created
deployment.apps/frontend created
service/frontend created
serviceaccount/skupper-site-controller created
role.rbac.authorization.k8s.io/skupper-site-controller created
rolebinding.rbac.authorization.k8s.io/skupper-site-controller created
deployment.apps/skupper-site-controller created
configmap/skupper-site created
~~~

_**Console for east:**_

~~~ shell
kubectl apply -f east/backend.yaml -f east/skupper.yaml -f east/site.yaml -f east/connector.yaml
~~~

_Sample output:_

~~~ console
$ kubectl apply -f east/backend.yaml -f east/skupper.yaml -f east/site.yaml -f east/connector.yaml
namespace/east created
deployment.apps/backend created
serviceaccount/skupper-site-controller created
role.rbac.authorization.k8s.io/skupper-site-controller created
rolebinding.rbac.authorization.k8s.io/skupper-site-controller created
deployment.apps/skupper-site-controller created
configmap/skupper-site created
~~~

## Step 4: Link your namespaces

You can configure sites and service bindings declaratively, but
linking sites is different.  To create a link, you must have the
authentication secret and connection details of the remote site.
Since these cannot be known in advance, linking must be
procedural.

**Note:** There are several ways to automate the generation and
distribution of tokens across sites, using for example Ansible,
Backstage, or Vault.  See [Token distribution]() for more
information.

This example uses the Skupper command line tool to generate the
secret token in west and create the link in east.

To install the Skupper command:

~~~ shell
curl https://skupper.io/install.sh | sh
~~~

For more installation options, see [Installing
Skupper][install].

Once the command is installed, use `skupper token create` in
west to generate the token.  Then, use `skupper link create` in
east to create a link.

[install]: https://skupper.io/install/index.html

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

If your console sessions are on different machines, you may need
to use `sftp` or a similar tool to transfer the token securely.
By default, tokens expire after a single use or 15 minutes after
creation.

## Step 5: Test the application

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
kubectl delete -f west/frontend.yaml -f west/skupper.yaml -f west/site.yaml -f west/listener.yaml
~~~

_**Console for east:**_

~~~ shell
kubectl delete -f east/backend.yaml -f east/skupper.yaml -f east/site.yaml -f east/connector.yaml
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
