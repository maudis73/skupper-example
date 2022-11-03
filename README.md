# Connecting an AMQ broker and an AMQP client deployed on separate sites

#### Connect multiple OCP sites and VM using Skupper

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
* [Step 9: Deploy the client](#step-9-deploy-the-client)
* [Step 10: Run the client](#step-10-run-the-client)
* [Step 11: Install the broker on your laprop](#step-12-install-the-laptop-broker)
* [Step 12: Install the Skupper gateway](#step-12-install-the-skupper-gateway)
* [Step 13: Expose the laptop broker](#step-13-expose-the-laptop-broker)
* [Step 14: Connect to the laptop broker from ns2](#step-14-connect-to-the-laptop-broker-from-ns2)

* [Accessing the web console](#accessing-the-web-console)
* [Cleaning up](#cleaning-up)

## Overview

This example is a simple messaging application that shows how you
can use Skupper to access an AMQ broker at a remote site
without exposing it to the public internet.

It contains three services:

* An AMQ broker running in OCP cliester OCP1. 

* An AMQP client running in ns1 and ns2. We will logon with a terminal 
  and run producers and consumers.

* An AMQ broker running in on a VM.

For the broker, this example uses the the AMQ broker operator.

The example uses two OpenShift namespaces, "ns1" and "ns2".


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
`skupper` and `oc` commands use the `KUBECONFIG` environment
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

Use `oc create namespace` to create the namespaces you wish
to use (or use existing namespaces).  Use `oc project` to set the current namespace for each session.

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

_**Console for ns1:**_

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
oc apply -f broker1.yaml
~~~

_Sample output:_

~~~ console
$ oc apply -f broker1.yaml
activemqartemis.broker.amq.io/broker1 created
~~~

## Step 8: Expose the message broker

In the ns1 namespace, use `skupper expose` to expose the
broker on the Skupper network.

Then, in the ns2 namespace, use `oc get service/ns-broker`
to check that the service appears after a moment.

_**Console for ns1:**_

~~~ shell
skupper expose statefulset/broker1-ss --port 61616 --address ns-broker
~~~

_Sample output:_

~~~ console
$ skupper expose statefulset/broker1-ss --port 61616 --address ns-broker
statefulset broker1-ss exposed as ns-broker
~~~

_**Console for ns2:**_

~~~ shell
oc get service/ns-broker
~~~

_Sample output:_

~~~ console
$ oc get service/ns-broker
NAME        TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)     AGE
ns-broker   ClusterIP   172.30.97.216   <none>        61616/TCP   2m43s~~~
~~~

## Step 9: Deploy the client

In the public namespace, use `oc new-app` to deploy the client.

_**Console for ns2:**_

~~~ shell
oc new-app quay.io/mdiscepo/simple-shell@sha256:e2036d219b69580f2dff57426754597b28dd7f67010df538dfe614fa4082e3fb
~~~

_Sample output:_

~~~ console
$ oc new-app quay.io/mdiscepo/simple-shell@sha256:e2036d219b69580f2dff57426754597b28dd7f67010df538dfe614fa4082e3fb
--> Found container image a1bec26 (5 months old) from quay.io for "quay.io/mdiscepo/simple-shell@sha256:e2036d219b69580f2dff57426754597b28dd7f67010df538dfe614fa4082e3fb"

    * An image stream tag will be created as "simple-shell:latest" that will track this image

--> Creating resources ...
    imagestream.image.openshift.io "simple-shell" created
    deployment.apps "simple-shell" created
--> Success
    Run 'oc status' to view your app.
~~~


## Step 10: Run the client

Connect to the ns2 namespace simple-shell pod's terminal.

_**Console for simple-shell on ns2:**_

~~~ shell
artemis queue stat --url tcp://ns-broker:61616
~~~

_Sample output:_

~~~ console
$ artemis queue stat --url tcp://ns-broker:61616
Connection brokerURL = tcp://ns-broker:61616
|NAME                     |ADDRESS                  |CONSUMER_COUNT |MESSAGE_COUNT |MESSAGES_ADDED |DELIVERING_COUNT |MESSAGES_ACKED |SCHEDULED_COUNT |ROUTING_TYPE |
|DLQ                      |DLQ                      |0              |0             |0              |0                |0              |0               |ANYCAST      |
|ExpiryQueue              |ExpiryQueue              |0              |0             |0              |0                |0              |0               |ANYCAST      |
|TEST                     |TEST                     |0              |0             |0              |0                |0              |0               |ANYCAST      |
|activemq.management.e7a18022-280b-45a6-a72b-fda18f884fde|activemq.management.e7a18022-280b-45a6-a72b-fda18f884fde|1              |0             |0              |0                |0              |0               |MULTICAST    |
~~~


## Accessing the web console

Skupper includes a web console you can use to view the application
network.  To access it, use `skupper status` to look up the URL of
the web console.  Then use `oc get
secret/skupper-console-users` to look up the console admin
password.

**Note:** The `<console-url>` and `<password>` fields in the
following output are placeholders.  The actual values are specific
to your environment.

_**Console for ns1:**_

~~~ shell
skupper status
oc get secret/skupper-console-users -o jsonpath={.data.admin} | base64 -d
~~~

_Sample output:_

~~~ console
$ skupper status
Skupper is enabled for namespace "ns1" in interior mode. It is connected to 1 other site. It has 1 exposed service.
The site console url is: <console-url>
The credentials for internal console-auth mode are held in secret: 'skupper-console-users'

$ oc get secret/skupper-console-users -o jsonpath={.data.admin} | base64 -d
<password>
~~~

Navigate to `<console-url>` in your browser.  When prompted, log
in as user `admin` and enter the password.


## Step 11: Install the laptop broker

Follow the instructions in https://access.redhat.com/documentation/en-us/red_hat_amq_broker/7.10/html-single/getting_started_with_amq_broker/index#installing-broker-getting-started to install and start a samle broker.


## Step 12: Intall the skupper gateway
The `skupper gateway init` command starts a Skupper router on
your local system and links it to the Skupper router in the
current Kubernetes namespace.

_**Console for laptop:**_

~~~ shell
skupper gateway init --type podman
~~~

_Sample output:_

~~~ console
$ skupper gateway init --type podman
Skupper gateway: 'fedora-4gb-hel1-1-mdiscepo-maudis'. Use 'skupper gateway status' to get more information.
~~~


## Step 13: Expose the laptop broker

Use `skupper service create` to define a Skupper service called
`backend`.  Then use `skupper gateway bind` to attach your
running backend process as a target for the service.

_**Console for laptop:**_

~~~ shell
skupper service create ns-broker-local 61616
skupper gateway bind ns-broker-local localhost 61616
~~~

_Sample output:_

~~~ console
$ skupper service create backend 8080

$ skupper gateway bind backend localhost 8080
2022/11/03 15:24:43 CREATE io.skupper.router.tcpConnector fedora-4gb-hel1-1-mdiscepo-maudis-egress-ns-broker-local:61616 map[address:ns-broker-local:61616 host:localhost name:fedora-4gb-hel1-1-mdiscepo-maudis-egress-ns-broker-local:61616 port:61616 siteId:8b734fd8-32a1-4cfd-bfd8-dc29801dee0f]
~~~


## Step 14: Connect to the laptop broker from ns2

Connect to the ns2 namespace simple-shell pod's terminal.

_**Console for simple-shell on ns2:**_

~~~ shell
artemis queue stat --url tcp://ns-broker-local:61616
~~~

_Sample output:_

~~~ console
$ artemis queue stat --url tcp://ns-broker:61616
Connection brokerURL = tcp://ns-broker:61616
|NAME                     |ADDRESS                  |CONSUMER_COUNT |MESSAGE_COUNT |MESSAGES_ADDED |DELIVERING_COUNT |MESSAGES_ACKED |SCHEDULED_COUNT |ROUTING_TYPE |
|DLQ                      |DLQ                      |0              |0             |0              |0                |0              |0               |ANYCAST      |
|ExpiryQueue              |ExpiryQueue              |0              |0             |0              |0                |0              |0               |ANYCAST      |
|TEST                     |TEST                     |0              |0             |0              |0                |0              |0               |ANYCAST      |
|activemq.management.e7a18022-280b-45a6-a72b-fda18f884fde|activemq.management.e7a18022-280b-45a6-a72b-fda18f884fde|1              |0             |0              |0                |0              |0               |MULTICAST    |
~~~


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
