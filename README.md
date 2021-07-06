# Enonic XP on Kubernetes (Full Guide) (Unofficial)

This is a full guide, based on my 6+ months of experience trying to make this Framework work on Kubernetes. I'm not on Enonic Team, I'm just a DevOps guy that has been trying to do something new for the community and, the fact that I couldn't switch from several servers running single containers for each Enonic XP app, to a single Kubernetes cluster, was annoying me.

- [Enonic XP on Kubernetes (Full Guide) (Unofficial)](#enonic-xp-on-kubernetes-full-guide-unofficial)
  - [Enonic Operator for Kubernetes](#enonic-operator-for-kubernetes)
  - [Before starting](#before-starting)
  - [Shared directories](#shared-directories)
    - [NFS solutions](#nfs-solutions)
    - [Directories](#directories)
    - [PVCs](#pvcs)
  - [Headless service](#headless-service)
    - [Service DNS](#service-dns)
    - [Manifest](#manifest)
  - [Enonic Configuration](#enonic-configuration)
    - [com.enonic.xp.cluster.cfg](#comenonicxpclustercfg)
    - [com.enonic.xp.elasticsearch.cfg](#comenonicxpelasticsearchcfg)
    - [com.enonic.xp.hazelcast.cfg](#comenonicxphazelcastcfg)
    - [com.enonic.xp.web.sessionstore.cfg](#comenonicxpwebsessionstorecfg)
    - [com.enonic.xp.web.vhost.cfg](#comenonicxpwebvhostcfg)
    - [system.properties](#systemproperties)
  - [Applying configuration](#applying-configuration)
    - [Creating ConfigMap](#creating-configmap)
    - [Creating ConfigMap (manifest only)](#creating-configmap-manifest-only)
  - [Mixing and Deploying](#mixing-and-deploying)
    - [Manifest](#manifest-1)
    - [Explanation](#explanation)
      - [Annotations](#annotations)
      - [Liveness Probe](#liveness-probe)
      - [Readiness Probe](#readiness-probe)
      - [Termination Grace Period](#termination-grace-period)
      - [Volumes](#volumes)
      - [Volume Mounts](#volume-mounts)
  - [Installing applications](#installing-applications)
  - [Using Ingress](#using-ingress)
  - [Troubleshooting](#troubleshooting)
  - [Contribute](#contribute)
  - [Submit Feedback](#submit-feedback)

## Enonic Operator for Kubernetes

To make this possible, I made a Kubernetes Operator for performing several tasks on the ElasticSearch cluster, that is used for Enonic XP to store CMS's data indexes.

After reading this guide, please go and read [the operator documentation](https://github.com/DaviPtrs/enonic-operator-k8s) too. **THIS IS COMPLETELY NECESSARY**


## Before starting

We will deploy all our resources on another namespace, which I called `girons`, so to follow this example, create the namespace by the following command

```bash
kubectl create ns girons
```
## Shared directories

According to the ["clustering storage" section on Enonic docs](https://developer.enonic.com/docs/xp/stable/deployment/clustering#storage), Enonic XP needs a shared file system to share directories for all replicas. 

### NFS solutions

We will use NFS-based solutions, here are some options:

-   (Recommended) If you are on AWS, you can use EFS. Follow [this guide](https://www.padok.fr/en/blog/efs-provisioner-kubernetes) to learn how to use EFS volumes on Kubernetes.

-   If you are on a bare-metal cluster or Google Cloud, you need to spin up your own NFS server, then use [nfs-client-provisioner](https://github.com/rimusz/nfs-client-provisioner) to be able to create NFS volumes for Kubernetes.

### Directories

According to [this](https://developer.enonic.com/docs/xp/stable/deployment/clustering#dirs), these are the shared directories:

-   $XP_HOME/repo/blob
    
    Contains all files managed by XP.

-   $XP_HOME/snapshots
    
    Contains ElasticSearch index snapshots.

-   $XP_HOME/data
    
    Contains other data (e.g. system dumps).

-   $XP_HOME/deploy

    Contains the cluster-wide installed applications.
  
**Disclaimer**: The deploy folder usually should not be shared, but we'll take another way. The deploy folder needs to be shared so the operator can install a desired jar file inside the Deploy volume.

So for them, we will create PVCs, to request volumes to our NFS solutions and then, we will attach them on those directories.

### PVCs

These are my PVCs declaration, in my case, I'm using the storage class "managed-nfs-storage", which is related to nfs-client-provisioner, mentioned in [this section](#nfs-solutions). So you need to replace them with your desired storage class, according to which solution you gonna use.

```yaml
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: giropops-blobs
  namespace: girons
  annotations:
    volume.beta.kubernetes.io/storage-class: "managed-nfs-storage"
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 3Gi

---

kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: giropops-data
  namespace: girons
  annotations:
    volume.beta.kubernetes.io/storage-class: "managed-nfs-storage"
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 3Gi

---

kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: giropops-snapshots
  namespace: girons
  annotations:
    volume.beta.kubernetes.io/storage-class: "managed-nfs-storage"
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 3Gi

---

kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: giropops-deploy
  namespace: girons
  annotations:
    volume.beta.kubernetes.io/storage-class: "managed-nfs-storage"
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 400Mi
```

As you can see, for which directory, I created a PVC, requesting 3GBs storage from my NFS storage class, you of course might change this value according to your needs.

## Headless service

As you have to do with most StatefulSet objects, you will set a Headless Service, which is a service without a single Service IP or load-balancing. This service will be used by the cluster to communicate with its replicas. We also need to set up another service, that will be used by the ingress controller and allows the clients to access the application.

### Service DNS

The service can be used to reach all replicas by internal DNS record, this DNS follows the following pattern:

```
<service name>.<namespace>.svc.<cluster-domain>
```

`<cluster-domain>` is usually equals to `cluster.local`

In our example case, as our service will be called "giropops-intra-service" and namespace are "girons", will be:

```
giropops-intra-service.girons.svc.cluster.local
```

### Manifest

This is our services example, which will use "name: giropops" to select the application statefulset

```yaml
# ---------- Exposer service ------------
apiVersion: v1
kind: Service
metadata:
  name: giropops-service
  namespace: girons
spec:
  selector:
    name: giropops
  ports:
  - name:  xp-port
    port:  8080
    targetPort:  8080

---
# ---------- Internal service ------------
apiVersion: v1
kind: Service
metadata:
  name: giropops-intra-service
  namespace: girons
spec:
  clusterIP: None
  selector:
    name: giropops
  ports:
    - port: 5701
      name: hazelcast
    - port: 9300
      name: elasticsearch
  type: ClusterIP
  publishNotReadyAddresses: true
```

The main thing here to define our service as a headless service is setting `None` on clusterIp key

```yaml
spec:
  clusterIP: None
  ...
```

## Enonic Configuration

There're some essentials configurations to do, all others are completely optional to make the cluster works. If you want to learn more about Enonic XP Config Files, see [the official documentation](https://developer.enonic.com/docs/xp/stable/deployment/config#xp_config_files)

### com.enonic.xp.cluster.cfg

```conf
cluster.enabled = true

network.host = _eth0_

network.publish.host = _eth0_
```

### com.enonic.xp.elasticsearch.cfg

Replace with your application service DNS. If you don't know what I'm talking about, see [this section](#headless-service)

```conf
discovery.unicast.sockets = giropops-intra-service.girons.svc.cluster.local
```

### com.enonic.xp.hazelcast.cfg

Replace `network.join.kubernetes.serviceDns` with the same DNS settings as [com.enonic.xp.elasticsearch.cfg](#comenonicxpelasticsearchcfg).
You can also change `system.hazelcast.initial.min.cluster.size` according to your needs (if you don't want to deploy single replicas for each app)
```conf
clusterConfigDefaults = false

network.join.kubernetes.enabled = true

network.join.kubernetes.serviceDns = giropops-intra-service.girons.svc.cluster.local

network.join.tcpIp.enabled = false

# Initial expected cluster size to wait before node to start completely
system.hazelcast.initial.min.cluster.size = 1
```

### com.enonic.xp.web.sessionstore.cfg

For this one there are some options that maybe you want to change according to your needs. The only mandatory setting is `storeMode = replicated`, the others can be changed without affecting the cluster health. See [Enonic Config Docs](https://developer.enonic.com/docs/xp/stable/deployment/config#sessionstore) for more info.

```conf
# Required
storeMode = replicated

# Can be changed | Optional
saveOnCreate = true

# Can be changed | Optional
flushOnResponseCommit = true
```

### com.enonic.xp.web.vhost.cfg

You should also configure the vhosts, see the [official docs reference](https://developer.enonic.com/docs/xp/stable/deployment/config#vhost). You can use the following example


```conf
enabled = true

mapping.site.host = app.example.com
mapping.site.source = /
mapping.site.target = /site/default/master/example

mapping.admin.host = app.example.com
mapping.admin.source = /admin
mapping.admin.target = /admin
mapping.admin.idProvider.system = default
```

### system.properties

The operator can only reach our application (to perform stability tasks) if it knows how to authenticate to the cluster, so you must define a superuser password, which will be the same as you need to put on a secret on the Enonic operator's tutorial.


```conf
xp.suPassword = samplepass

# Optional but recommended
xp.init.adminUserCreation = false
```

## Applying configuration

Some people may try to pack the config files on container image build, turning it static, that will need to rebuild the application container image when you what to change some config entry. **PLEASE... DON'T... JUST DON'T...**

A good practice, when we're talking about Kubernetes, is use ConfigMaps! See [this official Kubernetes doc about ConfigMaps](https://kubernetes.io/docs/concepts/configuration/configmap/)

Resuming, ConfigMap can be mounted as a volume and each declared config file inside it will be a regular file inside this volume, so the application won't notice the difference (if you're using ConfigMap, packing the config files inside the container image, or even binding a host path), and of course, it can be changed in runtime, so that's no need to change the container image or even restarting the application (usually).

### Creating ConfigMap

You can create a ConfigMap using a folder with all of the configuration files inside. this following command you convert this folder to a ConfigMap

```
kubectl create configmap -n <namespace> <configmap-name> --from-file=<folder-path>
```

In our case, it will be:

```
kubectl create configmap -n girons giropops-config --from-file=config
```

### Creating ConfigMap (manifest only)

You can also create just the YAML manifest, without creating the object in your cluster. This is very useful to be used on CI scripts.

```
kubectl create configmap -n <namespace> <configmap-name> --from-file=<folder-path> \
--dry-run="client" -o yaml > configmap.yaml
```

In our case, it will be:

```
kubectl create configmap -n girons giropops-config --from-file=config \
--dry-run="client" -o yaml > configmap.yaml
```

## Mixing and Deploying

### Manifest

If you did everything right, this will be your StatefulSet YAML Manifest

```yaml
# ---------- Deploy ------------
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: giropops
  namespace: girons
  annotations:
    enonic-operator-managed: ""
spec:
  serviceName: giropops-service
  selector:
      matchLabels:
        name: giropops
  template:
    metadata:
      labels:
        name: giropops
    spec:
      containers:
        - image: app-image:tag
          name: xp-app
          volumeMounts:
            - mountPath: /enonic-xp/home/config/
              name: config-volume
            - mountPath: /enonic-xp/home/repo/blob/
              name: blobs
            - mountPath: /enonic-xp/home/snapshots/
              name: snapshots
            - mountPath: /enonic-xp/home/data/
              name: data
            - mountPath: /enonic-xp/home/deploy/
              name: deploy
          livenessProbe:
            httpGet:
              path: /cluster.manager
              port: 2609
            failureThreshold: 2
            initialDelaySeconds: 30
            periodSeconds: 10
          readinessProbe:
            httpGet:
              path: /cluster.manager
              port: 2609
            initialDelaySeconds: 30
            periodSeconds: 10
      terminationGracePeriodSeconds: 180
      volumes:
        - name: config-volume
          configMap:
            name: giropops-config
        - name: blobs
          persistentVolumeClaim:
            claimName: giropops-blobs
        - name: snapshots
          persistentVolumeClaim:
            claimName: giropops-snapshots
        - name: data
          persistentVolumeClaim:
            claimName: giropops-data
        - name: deploy
          persistentVolumeClaim:
            claimName: giropops-deploy
```

### Explanation

If you don't know how a StatefulSet on Kubernetes works, see these official guides

- https://kubernetes.io/docs/tutorials/stateful-application/basic-stateful-set/
- https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/

In the following sub-sections, I'll explain the key parts of our StatefulSet YAML, so I'll assume that you're familiar with the basic concepts.

#### Annotations

This annotation are the required to work with [Enonic Operator for k8s](https://github.com/DaviPtrs/enonic-operator-k8s)

```yaml
metadata:
  annotations:
    enonic-operator-managed: ""
```

#### Liveness Probe

This will make sure that the cluster is healthy, if it is not, the container will be restarted. If you want time to perform some operations before the cluster spins up, you can increase the settings, but **DON'T DECREASE**, believe me, I've been testing this a lot.

```yaml
          livenessProbe:
            httpGet:
              path: /cluster.elasticsearch
              port: 2609
            failureThreshold: 2
            initialDelaySeconds: 30
            periodSeconds: 10
```

#### Readiness Probe

This will make sure that the replica is ready before being added to the main service, if it is not, the container will be restarted. This prevents unready instances to be served to clients and also helps the scaling process, so this probe is critical, don't decrease those parameters unless you know exactly what you are doing.

```yaml
          readinessProbe:
            httpGet:
              path: /cluster.manager
              port: 2609
            initialDelaySeconds: 30
            periodSeconds: 10
```

#### Termination Grace Period

This is how much time your application has to be terminated after receiving a SIGTERM. Since [Enonic Operator for k8s](https://github.com/DaviPtrs/enonic-operator-k8s) does some operations before letting the application being terminated, you **MUST NOT** decrease this setting! 

```yaml
terminationGracePeriodSeconds: 180
```

#### Volumes

Here we will import all those PVCs volumes and also the ConfigMap

```yaml
      volumes:
        - name: config-volume
          configMap:
            name: giropops-config
        - name: blobs
          persistentVolumeClaim:
            claimName: giropops-blobs
        - name: snapshots
          persistentVolumeClaim:
            claimName: giropops-snapshots
        - name: data
          persistentVolumeClaim:
            claimName: giropops-data
```

#### Volume Mounts

Mounting all the shared volumes according to [this section](#directories) and the config volume according to the default Enonic XP config directory (`/enonic-xp/home/config/`)

```yaml
          volumeMounts:
            - mountPath: /enonic-xp/home/config/
              name: config-volume
            - mountPath: /enonic-xp/home/repo/blob/
              name: blobs
            - mountPath: /enonic-xp/home/snapshots/
              name: snapshots
            - mountPath: /enonic-xp/home/data/
              name: data
            - mountPath: /enonic-xp/home/deploy/
              name: deploy
```

## Installing applications

The Enonic operator comes with a custom resource that allows you to install applications by downloading their .jar files from an S3 bucket, to learn how to set up it, see [this section of the operator's doc](https://github.com/DaviPtrs/enonic-operator-k8s#installing-jar-files-from-an-s3-bucket).

For this application, we will be using the following example, which is the `jar.yaml` manifest.

```yaml
apiVersion: kopf.enonic/v1
kind: EnonicXpApp
metadata:
  name: giropops
  namespace: girons
spec:
  secret_name: bucket-secret
  pvc_name: deploy-pvc
  bucket:
    url: s3.example.com
    url_sufix: sample-project/master
  object:
    prefix: "sample-project-"
    name: sample-project-1.1.0.jar
```


## Using Ingress

Regardless of which Ingress Class you gonna use, don't forget to set Sticky Sessions/Session Affinity, or your requests will jump through different replicas, causing problems on everything that needs to persist sessions.

Here is an example using [Ingress Nginx](https://kubernetes.github.io/ingress-nginx/)

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: giropops-ingress
  namespace: girons
  annotations:
    kubernetes.io/ingress.class: "nginx"
    nginx.ingress.kubernetes.io/from-to-www-redirect: "true"
    ingress.kubernetes.io/rewrite-target: /
    nginx.ingress.kubernetes.io/proxy-body-size: 1000m
    nginx.ingress.kubernetes.io/affinity: "cookie" # Mandatory
spec:
  rules:
    - host: host.example
      http:
        paths:
          - path: /
            backend:
              serviceName: giropops-service
              servicePort: 8080
```

## Troubleshooting

As I said, I'm not from Enonic Team, so give preference to ask on their community forum https://discuss.enonic.com/

They also have many other social channels. See on https://enonic.com/resources/community

I'm on their Slack so there I can see your question too. 


## Contribute

Contributions are always welcome!
If you need some light, read some of the following guides: 
- [The beginner's guide to contributing to a GitHub project](https://akrabat.com/the-beginners-guide-to-contributing-to-a-github-project/)
- [First Contributions](https://github.com/firstcontributions/first-contributions)
- [How to contribute to open source](https://github.com/freeCodeCamp/how-to-contribute-to-open-source)
- [How to contribute to a project on Github](https://gist.github.com/MarcDiethelm/7303312)

## Submit Feedback

Be free to [open an issue](https://github.com/DaviPtrs/enonic-xp-kubernetes/issues/new/choose) telling your experience, suggesting new features or asking questions (there's no stupid questions, but make sure that yours cannot be answered by just reading the docs)

You can also find me on LinkedIn [/in/davipetris](https://www.linkedin.com/in/davipetris/)