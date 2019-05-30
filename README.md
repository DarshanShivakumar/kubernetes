# COGNIGY.AI in k8s
This is the official Kubernetes (``k8s``) manifest repository for COGNIGY.AI. The following notes should give you a basic understand on how our product can be deployed on top of Kubernetes.

The API-objects we are using are tested with Kubernetes 1.9 - everything should work with more modern version of K8s as well.

## Introduction
This repository has the following folder-structure:
```
| - core
| --- config-maps
| --- deployments
| --- nlp-deployments
| ------- de
| ------- en
| ------- ge
| --- ingress
| --- reverse-proxy.dist
| --- secrets.dist
| --- services
| --- stateful-deployments
| --- volume-claims
| --- volumes.dist
|
| - livechat
| --- deployments
| --- ingress
| --- secrets
| --- services
|
| - management-ui
| --- deployments
| --- services
|
| - monitoring
| --- config-maps
| --- deployments
| --- secrets.dist
```

The main folders are ``core``, ``livechat``, ``management-ui`` and ``monitoring``. All of these direcotries potential contain sub-folders which contain K8s API objects of various sorts, such as:
- deployments
- ingress objects
- secrets
- services
- volume-claims
- volumes
- config-maps

We selected this folder structure to make it a bit more easy for you to understand and deploy our application. The ``core`` folder contains the actual API objects you will need to deploy a fully functional COGNIY.AI system. The other two folders within the root of this repository contain additional software that can be installed side-by-side with COGNIGY.AI. Those additional tools will not be functional without a working COGNIGY.AI deployment.

## Setup
This section will guide you through the process of setting up your COGNIGY.AI installation. You will no longer need it, once your initial deployment succeeded.

### Config map
---
COGNIGY.AI uses K8s' concept of config-maps for its full configuration. The directory ``core/config-maps`` contains all config-maps which you can edit and modify to your needs.

You will definitely need to modifye the BASE_URL entries to match the URLS you use in your application (e.g. ui.company.com).

After you are done with your modifications, deploy those within your K8s cluster:
```
kubectl apply -f core/config-maps
```

### Image Pull Secret
---
In order to pull the required ``docker images`` from Cognigy's production registry, you need to define a so-called ``image pull secret``. Kubernetes will use this secret in order to authenticate against our docker registry in order to retrieve docker images for a release.

We will provide the necessary ``username:password`` pair when you obtain your COGNIGY.AI license.

In order to create the image pull secret, issue the following command:
```
kubectl create secret docker-registry cognigy-registry-token \
    --docker-server=docker.cognigy.com:5000 \
    --docker-username=<username> \
    --docker-password=<password>
```

### Secrets
---
COGNIGY.AI heavily uses secrets, a K8s API object that allows you to store secret information in a secure way within your cluster. Secrets will be made accessible to our software during runtime.

The ``core/secrets.dist`` folder contains empty API objects you can use to prepare your correct secrets and finally deploy them. The first step within preparing your secrets is to rename the ``secrets.dist`` folder to ``secrets``. We have a ``.gitignore`` file within this repository which avoids that you accidentally push your final secrets to the public world.

Let's have a look at one of these files:
```
apiVersion: v1
kind: Secret
metadata:
    name: cognigy-facebook
type: Opaque
data:
    # -> base64 encoded
    fb-verify-token: 
```

The secret has some meta-data and a data part which contains the actual secret. In this example, the key is ``fb-verify-token`` and the actual value was not created. You can now create a random value for this using OpenSSL:
```
openssl rand -hex 32
```

This will genreate 32 bytes of random values encoded as HEX - these values are safe to use them as passwords, secrets and tokens. The secret API objects contain suggestions on what length should be used for best performance/security.

Store the raw-value (plain text values) of your generated secrets somewhere in a safe place. We use a password-manager like KeyPassX. In order to fill the actual secret API objects, your secret values now need to get ``base64`` encoded. You can do this with:
```
echo <your-secret> -n | base64 -w0
```

Take the value that was created and store it within your secret API object as followed:
```
apiVersion: v1
kind: Secret
metadata:
    name: cognigy-facebook
type: Opaque
data:
    # -> base64 encoded
    fb-verify-token: MWNmOTVlMmI5Nzg0YjQ0MWUyNTkxNTMyOGZiMzYzZjk4MzY3Nzc3YTg2MjI0ZjY3ZDI1YzQ1ZDM4Mjc1NjVlOSAtbgo=
```

You can now use the same procedure for all files within your ``core/secrets`` directory and finally deploy those API objects into your cluster issuing:
```
kubectl apply -f core/secrets
```

### Storage
#### Introduction
---
COGNIGY.AI needs to persist data in order to work properly. The simplest way to persist data is by using NFS as a volume-provider within your Kubernetes cluster.

Please bare in mind, that the performance of your NFS is crucial in order to have a high performance COGNIGY.AI installation! Don't put your NFS onto a slow network!

We usually setup a server and create the following directories in which NFS can store the actual data:
```
sudo mkdir -p /var/opt/cognigy/data/models
sudo mkdir /var/opt/cognigy/data/config
sudo mkdir /var/opt/cognigy/data/flow-modules
```

If your NFS is fast, you can also utilize it for MongoDB and Redis (persistent). We don't do this usually, cause databases suffer from bad disk-performance. Instead, we are using local volumes that will give you the full speed of your SSD storage.

Please also create the following additional directories where MongoDB and Redis (persistent) can store their data:
```
sudo mkdir -p /var/opt/cognigy/mongo
sudo mkdir -p /var/opt/cognigy/redis-persistent
```

#### Persistent Volumes
---
Persistent volumes (PVs) allow containerized applications to store their state (files, configuration, cache) in a persistent way either on the host or somewhere in the cloud. In order to run COGNIGY.AI you need to create persistent volumes so pods (defined by deployments) can mount persistent volumes by issuing so-called ``persistent volume claims``.

Have a look at the contents of ``core/volumes.dist`` and inspect all of the following PVs:
- flow-modules-nfs.yaml
- nlp-config-nfs.yaml
- nlp-models-nfs.yaml
- redis-persistent.yaml
- stateful-mongo-server.yaml

The reason why we named this directory ``.dist`` is the fact, that you definitely need to change these files before you apply them to your cluster. You should not update those files in the future when Cognigy provides new files for a new release. Simply create a copy of the folder and name it ``volumes``. Do all of your modifications within the ``volumes`` folder and leave ``volumes.dist`` untouched.

We essentially have grouped them and are using ``NFS`` for some of those volumes (they have the ``-nfs`` prefix in their names). The other volumes for ``redis-persistent`` and ``mongo`` use local volumes that are persistent on one of the hosts directly.

Be sure to exchange the ``server/path`` pair within the ``-nfs``-prefixed files as well as the ``machine-name`` (#change me!) for your local volumes.

If you finished all of your adjustments, deploy the PV objects into your cluster:
```
kubectl apply -f core/volumes/
```

#### Persistent Volume Claims
Persistent volume claims (PVCs) are the actual way how Pods express their needs for storage. They claim a part of a PV. Kubernetes will reserve this block of the whole PV specifically for them!

All of the PVC objects are within ``core/volume-claims``. You might want to adjust the ``storage`` request value in case you also modified the PV objects. Please ensure that the storage request is actually lower than what your PVs offer.

If you are done with your modifications, deploy the objects using:
```
kubectl apply -f core/volume-claims/
```

#### Stateful deployments
COGNIGY.AI is dependent on MongoDB, Redis, a second Redis which persists data and RabbitMQ. Those components need to be available in order so COGNIGY.AI can be deployed and started.

To make it easier to separate our software from those dependencies, we created the ``core/stateful-deployments`` which only contain deployment objects for databases and the message broker. Feel free to have a look at those files.

If you are done, deploy those objects using:
```
kubectl apply -f core/stateful-deployments/
```

#### Initializing MongoDB
After you have deployed all of the stateful deployments, wait until all of your Pods are up and running.

We now need to create various databases within MongoDB and create users for the individual databases. In order to create those databases and users, let's connect to our running MongoDB by executing the ``mongo`` (client) command within the MongoDB pod. In order to do that, first get the name/id of your MongoDB pod:
```
kubectl get po
```

Find the actual Pod name of your MongoDB copy and copy it. Then execute the following command and replace ``<mongo-pod-name>`` as well as ``<mongo-admin-password>``:
```
kubectl exec -it <mongo-pod-name> mongo -u admin -p <mongo-admin-password> --authenticaitonDatabase admin
```

You should now see the mongo client prompt. In order to create all databases and users, execute the following command for all of the databases you see within the list below:
```
use service-profiles
db.createUser({
    user: "service-profiles",
    pwd: "zC8RPdCWrbMQkauZfQutgJ86JNyJPXajMMtqk7XTv74Q6w6sqwKD2LyhLL7ZELB7aFcTPUHWuGcSaVfT5gqQfc276sMPFwE6bJrCQJZnZq5E52bbCCa6UcxD5f8GXmLk",
    roles: [
        { role: "readWrite", db: "service-profiles" }
    ]
})
```

Be sure to replace 'service-profiles' with the actual name from the list below and also replace the password (the part after 'pwd') with the actual clear-type (not the base64 encoded one) password you have generated earlier in the Secrets step (the ones you might have stored in your password manager).

An example for ``service-ai`` from the list would look like:
```
use service-ai
db.createUser({
    user: "service-ai",
    pwd: "my-super-secure-service-ai-password",
    roles: [
        { role: "readWrite", db: "service-ai" }
    ]
})
```

Execute the command above for every element within the following list:

| database/user name | Optional notes |
| ------------------ | ----- |
| service-ai | - |
| service-alexa-management | - |
| service-analytics-collector-provider | Used by ``service-analytics-collector``, ``service-analytics-odata`` as well as ``service-analytics-reporter`` |
| service-analytics-conversation-collector-provider | Used by ``service-analytics-conversation-collector`` as well as ``service-analytics-conversation-provider`` |
| service-api | - |
| service-custom-modules | - |
| service-database-connections | - |
| service-endpoints | - |
| service-files | - |
| service-flows | - |
| service-forms | - |
| service-lexicons | - |
| service-logs | - |
| service-nlp | - |
| service-nlp-connectors | - |
| service-playbooks | - |
| service-profiles | - |
| service-projects | - |
| service-secrets | - |
| service-security | - |
| service-settings | - |

If you want to use the additional ``livechat`` product, you also need to create the database/user for:

| database/user name | Optional notes |
| ------------------ | ----- |
| service-handover | - |

After you created all of the databases within MongoDB, quite the shell by typing ``exit``. You can now apply our main ``deployment`` objects by executing:
```
kubectl apply -f core/services/
kubectl apply -f core/deployments/
```

### Availability of your installation
COGNIGY.AI exposes several software components which need to be available from outside of your installation. These are services such as ``endpoint``, ``ui`` and ``api``. Our customers use these services to build new Conversational AIs (ui, api) and make them available to other platforms like ``Facebook Messenger``.

In the folder ``core/reverse-proxy.dist``, you will find the necessary API objects to expose COGNIGY.AI. You should therefore rename the folder to ``reverse-proxy``, since we need to make some changes.

We use a modern ``Ingress Controller`` which allows external web-traffic to reach certain software components within your cluster - it's called Traefik. If you do not have your own reverse proxy set up, then you can deploy Traefik. Do this by first changing the external IP address in the file ``core/reverse-proxy/services/traefik.yaml`` to be the external IP of your server, and afterwards, you can run the two commands below to deploy Traefik:

```
kubectl apply -f core/reverse-proxy/services/
kubectl apply -f core/reverse-proxy/deployments/
```

The Ingress objects stored within ``core/reverse-proxy/ingress`` control how the reverse proxy can reach your internal services and can make them accessible. If you use Traefik as the reverse proxy, then you can simply apply these files to make COGNIGY.AI accessible. If you are using a different reverse proxy, then you should modify them to work with this reverse proxy (e.g. modify the ingress annotation class).

In order to deploy all of your ingress configurations execute:

```
kubectl apply -f core/ingress/
```