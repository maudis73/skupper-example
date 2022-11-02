# Connecting an AMQ broker and an AMQP client deployed on separate sites

#### Use public cloud resources to process data from a private message broker

#### Contents

* [Overview](#overview)
* [Prerequisites](#prerequisites)
* [Step 1: Configure separate console sessions](#step-1-configure-separate-console-sessions)
* [Step 2: Access your clusters](#step-2-access-your-clusters)
* [Step 3: Set up your namespaces](#step-3-set-up-your-namespaces)
* [Step 4: Install Skupper in your namespaces](#step-4-install-skupper-in-your-namespaces)
* [Step 5: Check the status of your namespaces](#step-5-check-the-status-of-your-namespaces)
* [Step 6: Link your namespaces](#step-6-link-your-namespaces)
* [Step 7: Deploy the message broker](#step-7-deploy-the-message-broker)
* [Step 8: Expose the message broker](#step-8-expose-the-message-broker)
* [Step 9: Run the client](#step-9-run-the-client)
* [Accessing the web console](#accessing-the-web-console)
* [Cleaning up](#cleaning-up)

## Overview

This example is a simple messaging application that shows how you
can use Skupper to access an AMQ broker at a remote site
without exposing it to the public internet.

It contains two services:

* An AMQ broker running in OCP cliester OCP1. The broker
  has a queue named "broker1".

* An AMQP client running in OCP cliester OCP2. We will logon with a terminal 
  and run producerd and consumers.

For the broker, this example uses the the AMQ broker operator.

The example uses two OpenShift namespaces, "OC1" and "OC2".


## Prerequisites

* The `oc` command-line tool

* The `skupper` command-line tool

* Access to at least one both OCP clusters


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

_**Console for ns1:**_

~~~ shell
export KUBECONFIG=~/.kube/config-ns1
~~~

_**Console for ns2:**_

~~~ shell
export KUBECONFIG=~/.kube/config-ns2
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

_**Console for ns1:**_

~~~ shell
oc create namespace ns1
oc project ns1
~~~

_Sample output:_

~~~ console
$ oc create namespace ns1
namespace/ns1 created

$ oc project ns1
Now using project "ns1" on server ....
~~~

_**Console for ns2:**_

~~~ shell
oc create namespace ns2
oc project ns2
~~~

_Sample output:_

~~~ console
$ oc create namespace ns2
namespace/ns1 created

$ oc project ns2
Now using project "ns2" on server ....
~~~

## Step 4: Install Skupper in your namespaces

The `skupper init` command installs the Skupper router and service
controller in the current namespace.  Run the `skupper init` command
in each namespace.

_**Console for ns2:**_

~~~ shell
skupper init
~~~

_Sample output:_

~~~ console
$ skupper init
Waiting for LoadBalancer IP or hostname...
Skupper is now installed in namespace 'ns1'.  Use 'skupper status' to get more information.
~~~

_**Console for ns2:**_

~~~ shell
skupper init
~~~

_Sample output:_

~~~ console
$ skupper init
Waiting for LoadBalancer IP or hostname...
Skupper is now installed in namespace 'ns2'.  Use 'skupper status' to get more information.
~~~

## Step 5: Check the status of your namespaces

Use `skupper status` in each console to check that Skupper is
installed.

_**Console for ns1:**_

~~~ shell
skupper status
~~~

_Sample output:_

~~~ console
$ skupper status
Skupper is enabled for namespace "ns1" in interior mode. It is connected to 1 other site. It has 1 exposed service.
The site console url is: <console-url>
The credentials for internal console-auth mode are held in secret: 'skupper-console-users'
~~~

_**Console for ns2:**_

~~~ shell
skupper status
~~~

_Sample output:_

~~~ console
$ skupper status
Skupper is enabled for namespace "ns2" in interior mode. It is connected to 1 other site. It has 1 exposed service.
The site console url is: <console-url>
The credentials for internal console-auth mode are held in secret: 'skupper-console-users'
~~~

As you move through the steps below, you can use `skupper status` at
any time to check your progress.

## Step 6: Link your namespaces

Creating a link requires use of two `skupper` commands in
conjunction, `skupper token create` and `skupper link create`.

The `skupper token create` command generates a secret token that
signifies permission to create a link.  The token also carries the
link details.  Then, in a remote namespace, The `skupper link
create` command uses the token to create a link to the namespace
that generated it.

**Note:** The link token is truly a *secret*.  Anyone who has the
token can link to your namespace.  Make sure that only those you
trust have access to it.

First, use `skupper token create` in one namespace to generate the
token.  Then, use `skupper link create` in the other to create a
link.

_**Console for ns1:**_

~~~ shell
skupper token create ~/secret.token
~~~

_Sample output:_

~~~ console
$ skupper token create ~/secret.token
Token written to ~/secret.token
~~~

_**Console for ns2:**_

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

## Step 7: Deploy the message broker

In the ns1 namespace, use the `oc apply` command to
install the broker.

_**Console for ns1:**_

~~~ shell
oc apply -f broker
~~~

_Sample output:_

~~~ console
oc apply -f broker
deployment.apps/broker created
~~~

## Step 8: Expose the message broker

In the private namespace, use `skupper expose` to expose the
broker on the Skupper network.

Then, in the public namespace, use `kubectl get service/broker`
to check that the service appears after a moment.

_**Console for private:**_

~~~ shell
skupper expose deployment/broker --port 5672
~~~

_Sample output:_

~~~ console
$ skupper expose deployment/broker --port 5672
deployment broker exposed as broker
~~~

_**Console for public:**_

~~~ shell
kubectl get service/broker
~~~

_Sample output:_

~~~ console
$ kubectl get service/broker
NAME     TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)    AGE
broker   ClusterIP   10.100.58.95   <none>        5672/TCP   2s
~~~

## Step 9: Run the client

In the public namespace, use `kubectl run` to run the client.

_**Console for public:**_

~~~ shell
kubectl run client --attach --rm --restart Never --image quay.io/skupper/activemq-example-client --env SERVER=broker
~~~

_Sample output:_

~~~ console
$ kubectl run client --attach --rm --restart Never --image quay.io/skupper/activemq-example-client --env SERVER=broker
__  ____  __  _____   ___  __ ____  ______
 --/ __ \/ / / / _ | / _ \/ //_/ / / / __/
 -/ /_/ / /_/ / __ |/ , _/ ,< / /_/ /\ \
--\___\_\____/_/ |_/_/|_/_/|_|\____/___/
2022-05-27 11:19:07,149 INFO  [io.sma.rea.mes.amqp] (main) SRMSG16201: AMQP broker configured to broker:5672 for channel incoming-messages
2022-05-27 11:19:07,170 INFO  [io.sma.rea.mes.amqp] (main) SRMSG16201: AMQP broker configured to broker:5672 for channel outgoing-messages
2022-05-27 11:19:07,198 INFO  [io.sma.rea.mes.amqp] (main) SRMSG16212: Establishing connection with AMQP broker
2022-05-27 11:19:07,212 INFO  [io.sma.rea.mes.amqp] (main) SRMSG16212: Establishing connection with AMQP broker
2022-05-27 11:19:07,215 INFO  [io.quarkus] (main) client 1.0.0-SNAPSHOT on JVM (powered by Quarkus 2.9.2.Final) started in 0.397s.
2022-05-27 11:19:07,215 INFO  [io.quarkus] (main) Profile prod activated.
2022-05-27 11:19:07,215 INFO  [io.quarkus] (main) Installed features: [cdi, smallrye-context-propagation, smallrye-reactive-messaging, smallrye-reactive-messaging-amqp, vertx]
Sent message 1
Sent message 2
Sent message 3
Sent message 4
Sent message 5
Sent message 6
Sent message 7
Sent message 8
Sent message 9
Sent message 10
2022-05-27 11:19:07,434 INFO  [io.sma.rea.mes.amqp] (vert.x-eventloop-thread-0) SRMSG16213: Connection with AMQP broker established
2022-05-27 11:19:07,442 INFO  [io.sma.rea.mes.amqp] (vert.x-eventloop-thread-0) SRMSG16213: Connection with AMQP broker established
2022-05-27 11:19:07,468 INFO  [io.sma.rea.mes.amqp] (vert.x-eventloop-thread-0) SRMSG16203: AMQP Receiver listening address notifications
Received message 1
Received message 2
Received message 3
Received message 4
Received message 5
Received message 6
Received message 7
Received message 8
Received message 9
Received message 10
Result: OK
~~~

## Accessing the web console

Skupper includes a web console you can use to view the application
network.  To access it, use `skupper status` to look up the URL of
the web console.  Then use `kubectl get
secret/skupper-console-users` to look up the console admin
password.

**Note:** The `<console-url>` and `<password>` fields in the
following output are placeholders.  The actual values are specific
to your environment.

_**Console for public:**_

~~~ shell
skupper status
kubectl get secret/skupper-console-users -o jsonpath={.data.admin} | base64 -d
~~~

_Sample output:_

~~~ console
$ skupper status
Skupper is enabled for namespace "public" in interior mode. It is connected to 1 other site. It has 1 exposed service.
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

_**Console for private:**_

~~~ shell
kubectl delete -f broker
skupper delete
~~~

_**Console for public:**_

~~~ shell
skupper delete
~~~

## Next steps


Check out the other [examples][examples] on the Skupper website.
