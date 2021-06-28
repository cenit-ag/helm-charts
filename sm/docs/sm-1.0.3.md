# 1. Helm Chart for Enterprise System Monitor / Service Monitor

<!-- TOC -->

- [1. Helm Chart for Enterprise System Monitor / Service Monitor](#1-helm-chart-for-enterprise-system-monitor--service-monitor)
- [2. Prerequisites](#2-prerequisites)
    - [2.1. Compatibility](#21-compatibility)
    - [2.2. RBAC](#22-rbac)
    - [2.3. Admin Password Secret](#23-admin-password-secret)
- [3. Installation](#3-installation)
    - [3.1. Default Installation (SM Server & SM Agent)](#31-default-installation-sm-server--sm-agent)
    - [3.2. SM Server Only Installation](#32-sm-server-only-installation)
    - [3.3. SM Agent Only Installation](#33-sm-agent-only-installation)
    - [3.4. Ingress Access](#34-ingress-access)
    - [3.5. Cleanup](#35-cleanup)
        - [3.5.1. Uninstall Release](#351-uninstall-release)
        - [3.5.2. Delete Persistent Volumes](#352-delete-persistent-volumes)
        - [3.5.3. Delete Secrets & ConfigMaps](#353-delete-secrets--configmaps)
    - [3.6. Upgrades](#36-upgrades)
    - [3.7. Installation on OpenShift](#37-installation-on-openshift)
        - [3.7.1. Push Images to IBM Container Registry](#371-push-images-to-ibm-container-registry)
        - [3.7.2. Cluster Login](#372-cluster-login)
        - [3.7.3. Prepare a Project](#373-prepare-a-project)
        - [3.7.4. Create Secret for Admin Password](#374-create-secret-for-admin-password)
        - [3.7.5. Create Image Pull Secret](#375-create-image-pull-secret)
        - [3.7.6. Create RBAC Resources](#376-create-rbac-resources)
        - [3.7.7. Configure Chart](#377-configure-chart)
        - [3.7.8. Install & Verify](#378-install--verify)
- [4. Configuration Reference](#4-configuration-reference)
- [5. Configuration Examples](#5-configuration-examples)
    - [5.1. SM Server Persistence](#51-sm-server-persistence)
        - [5.1.1. Via H2 Database on a Persistent Volume](#511-via-h2-database-on-a-persistent-volume)
        - [5.1.2. Via External Database Services](#512-via-external-database-services)
    - [5.2. Dependency Injection](#52-dependency-injection)
        - [5.2.1. Option: Extended SM Container Images](#521-option-extended-sm-container-images)
        - [5.2.2. Option: Shared Volume](#522-option-shared-volume)
    - [5.3. Restricted Pod Permissions](#53-restricted-pod-permissions)
        - [5.3.1. Option A: RBAC Resources Created by the Chart](#531-option-a-rbac-resources-created-by-the-chart)
        - [5.3.2. Option B: RBAC Resources Created by the User](#532-option-b-rbac-resources-created-by-the-user)
- [6. Troubleshooting](#6-troubleshooting)
    - [6.1. Entering the SM Server Pod](#61-entering-the-sm-server-pod)
    - [6.2. Client-side Chart Testing](#62-client-side-chart-testing)
    - [6.3. Failing Probes on Startup](#63-failing-probes-on-startup)
    - [6.4. Job "mysm-smserver-set-password" Fails on Fresh Installations](#64-job-mysm-smserver-set-password-fails-on-fresh-installations)
- [7. Advanced](#7-advanced)
    - [7.1. Container Image Preloading](#71-container-image-preloading)
        - [7.1.1. On Docker Runtime](#711-on-docker-runtime)
        - [7.1.2. On Containerd Runtime](#712-on-containerd-runtime)
        - [7.1.3. On CRI-O Runtime](#713-on-cri-o-runtime)
        - [7.1.4. Into a k3s Cluster managed by k3d](#714-into-a-k3s-cluster-managed-by-k3d)
    - [7.2. In-Cluster Db2 Database for Testing Scenarios](#72-in-cluster-db2-database-for-testing-scenarios)
        - [7.2.1. Caveats](#721-caveats)
        - [7.2.2. Activation](#722-activation)
    - [7.3. Exposing SM Server to External Agents With NGINX Ingress](#73-exposing-sm-server-to-external-agents-with-nginx-ingress)

<!-- /TOC -->


# 2. Prerequisites

## 2.1. Compatibility

* Helm 3 is required. Helm 2 is __not__ supported.
* Kubernetes 1.15.x and higher.
* OpenShift 4.x.
* The chart can be installed multiple times per namespace.

## 2.2. RBAC

* The installing user (i.e. invoking `kubectl`) requires role `cluster-admin`. Alternatively, required RBAC resources can be created manually beforehand with an administrative user (see more [details](#restricted-pod-permissions)).
* On OpenShift it is required to run the containers in privileged mode (see more [details](#installation-on-openshift)).

## 2.3. Admin Password Secret

To secure the built-in `admin` user, its default password (set to `admin`) has to be set to a secure password. To set a password on installation, create a secret containing the new password before installing or upgrading:
```
kubectl create secret generic my-sm-admin-pw \
  --from-literal=password='s3cr3t'
```
Then refer to this password using key `server.adminUserSecret` either in your custom `values.yaml` file or using the `--set` argument.

It is also possible to set the password in the SM UI. Please note, that setting a password in the UI is only persistent in case you have configured [Persistent Storage for the SM Server](#persistent-storage-for-sm-server).

# 3. Installation

The notation in this document's installation commands refers to the chart packaged as `.tgz` file. To access the example YAML files mentioned in this documentation extract the archive:
```
tar -xzvf sm-5.5.5-0-003.tgz
```

## 3.1. Default Installation (SM Server & SM Agent)

When using the default values of the chart, a SM Server with one SM Agent will be installed, but most features are disabled that would require configuration specific to each cluster (in example Persistent Storage and Ingress). For this variety the only mandatory configuration are references to the container images.

A `values_custom.yaml` file could look like this:
```
server:
  image:
    repository: someregistry.example.com/sm/smserver
    tag: 5.5.5.0-002
agent:
  image:
    repository: someregistry.example.com/sm/smagent
    tag: 5.5.5.0-002
```

The following command installs a release called `mysm` using the Helm Chart in archive `sm-5.5.5-0-003.tgz` and above configuration file `values_custom.yaml`:
```
helm install mysm sm-5.5.5-0-003.tgz -f <path-to-your-values-file>/values_custom.yaml
```

If the chart has been extracted from the `.tgz` file, use the chart's folder:
```
helm install mysm sm/ -f <path-to-your-values-file>/values_custom.yaml
```

There are some example values files you can use after adjusting them to your needs, i.e.:
```
helm install mysm sm-5.5.5-0-003.tgz -f sm/examples/values/default_persistent.yaml
```

## 3.2. SM Server Only Installation

To only install the SM Server without a default SM Agent, set `agent.enabled` to `false`. The setting `server.enabled` is `true` by default, but set in the example anyway for more clarity.

Disable the SM Agent using a `values_custom.yaml` file:
```
[...]
server:
  enabled: true
[...]
agent:
  enabled: false
[...]
```
(Please note that `[...]` are only placeholders for this documentation to make the snippet easier to read.)

Apply this configuration using:
```
helm install mysm sm-5.5.5-0-003.tgz -f <path-to-your-values-file>/values_custom.yaml
```

Instead of disabling the SM Agent in a values file, another option is to pass the setting via a `--set` argument to Helm:
```
helm install mysm sm-5.5.5-0-003.tgz -f <path-to-your-values-file>/values_custom.yaml \
    --set server.enabled=true \
    --set agent.enabled=false
```
(Values set via `--set` take precedence over any values mentioned in a values file passed with `-f`.)

If you want to add a SM Agent to this release at a later point of time you can:
* ... [change](#upgrades) the existing release using `helm upgrade`.
* ... install a SM Agent in a [separate Helm release](#sm-agent-only-installation) and connect it to the existing SM Server.


## 3.3. SM Agent Only Installation

You can install a SM Agent for an existing SM Server. It doesn't matter whether the SM Server was previously installed with or without a SM Agent enabled. Below command installs a separate release containing a SM Agent only using `--set`.
```
helm install mysm-2nd-agent sm-5.5.5-0-003.tgz -f <path-to-your-values-file> \
    --set server.enabled=false \
    --set agent.enabled=true \
    --set server.hostname=mysm-smserver
```
The parameter `server.hostname` is mandatory in this scenario and has to point to an existing SM Server. In this example, it points to the existing SM Server installed with Helm release `mysm` within the same namespace. The pattern for the hostname is: `<release name of SM Server>-smserver`

If the SM Agent shall be installed in a namespace different from the SM Server use a pattern like this: `<release name of SM Server>-smserver.<namespace>`. Example: `mysm-smserver.some-namespace`

## 3.4. Ingress Access

The sample values file `default_persistent` shows an example configuration for NGINX Ingress:
```
server:
  ingress:
    enabled: true
    hosts:
       - paths: 
          - /sm(/|$)(.*)
    annotations:
      nginx.ingress.kubernetes.io/rewrite-target: /$2
```
With this Ingress configuration you will be able to access the cluster using: `https://<cluster-domain>/sm/index.html`. With the current version of SM, the `/index.html` postfix is mandatory.

## 3.5. Cleanup

### 3.5.1. Uninstall Release
```
helm uninstall mysm
```

### 3.5.2. Delete Persistent Volumes

Required volumes are created automatically by the Helm Chart in case persistence is enabled for SM Server using `volumeClaimTemplates`. Helm [cannot automatically delete](https://github.com/helm/helm/issues/3313) such volumes on uninstallation of a release. To manually delete the SM Server volume run:
```
kubectl delete pvc data-mysm-smserver-0
```
__Important:__ This deletes any state persisted to the volume by the SM Server like monitoring data and manually accomplished configuration steps.


### 3.5.3. Delete Secrets & ConfigMaps

In addition you may want to remove manually created Secrets and ConfigMaps.

## 3.6. Upgrades

To update a release, instad of uninstalling and re-installing it with a new configuration, it can be updated in place.
```
helm upgrade --install [...]
```
For more information on the command's syntax, please refer to the [Helm documentation](https://helm.sh/docs/).



## 3.7. Installation on OpenShift

The following steps show the installation procedure on an RedHat OpenShift Cluster on IBM Cloud ([ROKS](https://www.openshift.com/products/openshift-ibm-cloud)).

### 3.7.1. Push Images to IBM Container Registry

When using RedHat OpenShift on IBM Cloud, the SM container images can be provided to the cluster through the [IBM Container Registry](https://cloud.ibm.com/registry/catalog). 

1) Setup access to the registry follow the related [quickstart documentation](https://cloud.ibm.com/registry/start).

2) Import compressed or uncompressed images:
```
docker load --input smserver.tgz
docker load --input smagent.tgz
```

3) Tag images for remote registry:
```
docker tag smserver:5.5.5.0-002 de.icr.io/<your-namespace>/smserver:5.5.5.0-002
docker tag smagent:5.5.5.0-002 de.icr.io/<your-namespace>/smagent:5.5.5.0-002
```

4) Login to IBM Cloud:
```
ibmcloud login -a https://cloud.ibm.com -r <your-region> -c <your-account-id> -g <your-resource-group>
```

5) Login to the Container Registry:
```
ibmcloud cr login
```

6) Push the tagged images:
```
docker push de.icr.io/cenit/smserver:5.5.5.0-002
docker push de.icr.io/cenit/smagent:5.5.5.0-002
```

### 3.7.2. Cluster Login

Login to the OCP cluster:
```
oc login --token=<your-token> --server=<API-connection-point>
```

### 3.7.3. Prepare a Project

Create a separate project and switch into it:
```
oc new-project sm-test
```

If the project already exists, switch into it:
```
oc project sm-test
```

### 3.7.4. Create Secret for Admin Password

Create a secret to be used for the built-in admin account of SM Server:
```
oc create secret generic my-sm-admin-pw \
  --from-literal=password='s3cr3t'
```

See chapter [Admin Password Secret](#admin-password-secret) for more details.

### 3.7.5. Create Image Pull Secret

The `default` project already contains an Image Pull Secret to allow access to the Container Registry. For customly created projects this is [not the case](https://cloud.ibm.com/docs/openshift?topic=openshift-registry#other_registry_accounts).

Copy the existing secret to the custom project/namespace (in this example we are using namespace/project `sm-test`):
```
oc get secret all-icr-io -n default -o yaml | sed 's/default/sm-test/g' | oc create -n sm-test -f -
```

### 3.7.6. Create RBAC Resources

In current versions the SM container images partially still require root access during execution. As privileged execution of containers in disabled by default on OpenShift clusters, we need to enable this using a Service Account.

Create a Service Account:
```
oc create serviceaccount sm-svc-acc
```

Add the created account to the `privileged` Security Context Constraints 
```
oc adm policy add-scc-to-user privileged -z sm-svc-acc
```

### 3.7.7. Configure Chart

To deploy SM on OpenShift instead of Kubernetes several adjustments are required to the configuration values:

* `platform` must be set to `ocp`.
* `server.serviceAccountName` must be set to the service account created in [Create RBAC Resources](#create-RBAC-Resources)
* `agent.serviceAccountName` must be set to the service account created in [Create RBAC Resources](#create-RBAC-Resources)
* `server.image.repository` needs to point to the provided trough the IBM Container Registry
* `agent.image.repository` needs to point to the provided trough the IBM Container Registry
* `imagePullSecrets` has to include a pull secret working for the IBM Container Registry (i.e. `all-icr-io`)
* `server.persistence.storageClass` has to be set to a valid Storage Class providing block storage (i.e. `ibmc-block-bronze` - see [official reference](https://cloud.ibm.com/docs/containers?topic=containers-block_storage) for storage classes on IBM Cloud).
* `server.persistence.size` has to be set to a valid minimum size for the selected Storage Class (i.e. for `ibmc-block-bronze` the minimum is `20Gi`).

Enable access to the SM Server UI with a Route. Following configuration works for ROKS:
```
[...]
server:
[...]
  route:
    enabled: true
    tls:
      insecureEdgeTerminationPolicy: Redirect
      termination: passthrough
    useHttpsEndpoint: true
[...]
```
With this Route configuration the SM UI will be accessible with an URL similar to this: `https://<release-name>-smserver-<namespace/project>.<cluster-domain>`

An example values file is provided in `sm/examples/values/roks_persistent.yaml`.

### 3.7.8. Install & Verify

Install using Helm:
```
helm install mysm sm-5.5.5-0-003.tgz -f sm/examples/values/roks_persistent.yaml
```

Access SM UI via Route configured in the chart:
```
TODO
```


# 4. Configuration Reference

The following table lists the configurable parameters of the SM chart and their default values.

The following table lists the configurable parameters of the Sm chart and their default values.


| Parameter                | Description             | Default        |
| ------------------------ | ----------------------- | -------------- |
| `platform` | Installs for platform Kubernetes ('k8s') by default. Set to 'ocp' for OpenShift. | `"k8s"` |
| `createRbacResources` | Creates RBAC resources required if the default policies for Pods do not allow a rule 'runAsUser' = 'RunAsAny'. | `false` |
| `imagePullSecrets` | Optionally specify an array of imagePullSecrets for pulling images from a private container registry. Secrets must be manually created in the namespace. | `[]` |
| `chartHookDeletionPolicy` | Control what happens with finished or failed jobs. Example: "before-hook-creation,hook-succeeded" | `"before-hook-creation"` |
| `nameOverride` |  | `""` |
| `fullnameOverride` |  | `""` |
| `server.enabled` | Creates a StatefulSet for a SM Server (enabled by default). | `true` |
| `server.image.repository` |  | `"smserver"` |
| `server.image.tag` |  | `"5.5.5.0-003"` |
| `server.image.pullPolicy` |  | `"IfNotPresent"` |
| `server.service.type` |  | `"ClusterIP"` |
| `server.service.httpPort` |  | `8080` |
| `server.service.httpsPort` |  | `8443` |
| `server.service.mqttPort` |  | `1883` |
| `server.service.mqttsPort` |  | `8883` |
| `server.mqttLoadBalancer.enabled` |  | `false` |
| `server.mqttLoadBalancer.loadBalancerIP` |  | `""` |
| `server.mqttLoadBalancer.ports` |  | `[{"name": "mqtt", "protocol": "TCP", "port": 1883, "targetPort": 1883}, {"name": "mqtts", "protocol": "TCP", "port": 8883, "targetPort": 8883}]` |
| `server.serviceAccountName` | Name of exisiting service account. If unset, the namespace's default service account is used. | `""` |
| `server.logLevel` | Supported Log4J log levels: https://logging.apache.org/log4j/2.x/manual/customloglevels.html | `"ERROR"` |
| `server.adminPasswordSecret` | Existing secret containing the password for user 'admin' (key 'password' must contain password string). | `""` |
| `server.terminationGracePeriodSeconds` | Waiting period before a terminating pod is forcefully ended. Default value is 30 (for 30 seconds). | `30` |
| `server.resources` | No requests or limits set by default. | `{}` |
| `server.readinessProbe.initialDelaySeconds` |  | `120` |
| `server.readinessProbe.periodSeconds` |  | `20` |
| `server.readinessProbe.failureThreshold` |  | `9` |
| `server.readinessProbe.successThreshold` |  | `1` |
| `server.livenessProbe.initialDelaySeconds` |  | `300` |
| `server.livenessProbe.periodSeconds` |  | `10` |
| `server.livenessProbe.failureThreshold` |  | `3` |
| `server.livenessProbe.successThreshold` |  | `1` |
| `server.nodeSelector` |  | `{}` |
| `server.tolerations` |  | `[]` |
| `server.affinity` |  | `{}` |
| `server.podAnnotations` |  | `{}` |
| `server.podSecurityContext` |  | `{}` |
| `server.securityContext` |  | `{}` |
| `server.ingress.enabled` |  | `false` |
| `server.ingress.annotations` |  | `{}` |
| `server.ingress.hosts` |  | `[]` |
| `server.ingress.tls` |  | `[]` |
| `server.ingress.useHttpsEndpoint` | If true (default), Ingress will connect to the secure (HTTPS) endpoint on the SM Server Pod. True by default. False will also disable the HTTPS endpoint of the SM Server container. | `true` |
| `server.route.enabled` |  | `false` |
| `server.route.tls` |  | `{}` |
| `server.route.useHttpsEndpoint` | If true (default), Ingress will connect to the secure (HTTPS) endpoint on the SM Server Pod. True by default. | `true` |
| `server.persistence.existingClaim` | If 'existingClaim' is defined, a PVC must be created manually before volume will be bound | `""` |
| `server.persistence.enabled` |  | `false` |
| `server.persistence.storageClass` |  | `"-"` |
| `server.persistence.accessModes` |  | `["ReadWriteOnce"]` |
| `server.persistence.size` |  | `"8Gi"` |
| `server.persistence.annotations` |  | `{}` |
| `server.persistence.volumePermissions.enabled` |  | `false` |
| `server.persistence.volumePermissions.image.repository` |  | `""` |
| `server.persistence.volumePermissions.image.tag` |  | `""` |
| `server.persistence.volumePermissions.image.pullPolicy` |  | `"IfNotPresent"` |
| `server.persistence.volumePermissions.resources` |  | `{}` |
| `server.extraVolumes` |  | `[]` |
| `server.extraVolumeMounts` |  | `[]` |
| `server.extraInitContainers` |  | `[]` |
| `server.extraEnv` |  | `[]` |
| `server.externalConfigurationDatabase.enabled` | Enables usage of an external database for configuration data (DB2, SQL Server). If disabled, a local H2 database instance is used. | `false` |
| `server.externalConfigurationDatabase.jdbcUrl` |  | `""` |
| `server.externalConfigurationDatabase.jdbcDriverClass` |  | `""` |
| `server.externalConfigurationDatabase.jdbcSecret` |  | `""` |
| `server.externalMonitoringDatabase.enabled` | Enables usage of an external database for monitoring data (DB2, SQL Server). If disabled, a local H2 database instance is used. | `false` |
| `server.externalMonitoringDatabase.jdbcUrl` |  | `""` |
| `server.externalMonitoringDatabase.jdbcDriverClass` |  | `""` |
| `server.externalMonitoringDatabase.jdbcSecret` |  | `""` |
| `server.mountSharedVolume.enabled` | Mounts a existing RWX volume to karaf/deploy (i.e. to provide JDBC drivers). | `false` |
| `server.mountSharedVolume.claimName` |  | `"sm-shared-pvc"` |
| `agent.enabled` | Creates a Deployment with a single SM Agent Pod (enabled by default). | `true` |
| `agent.image.repository` |  | `"smagent"` |
| `agent.image.tag` |  | `"5.5.5.0-003"` |
| `agent.image.pullPolicy` |  | `"IfNotPresent"` |
| `agent.serviceAccountName` | Name of exisiting service account. If unset, the namespace's default service account is used. | `""` |
| `agent.logLevel` | Supported Log4J log levels: https://logging.apache.org/log4j/2.x/manual/customloglevels.html | `"ERROR"` |
| `agent.podSecurityContext` |  | `{}` |
| `agent.securityContext` |  | `{}` |
| `agent.terminationGracePeriodSeconds` |  | `30` |
| `agent.resources` | No requests or limits set by default. | `{}` |
| `agent.readinessProbe.initialDelaySeconds` |  | `120` |
| `agent.readinessProbe.periodSeconds` |  | `20` |
| `agent.readinessProbe.failureThreshold` |  | `9` |
| `agent.readinessProbe.successThreshold` |  | `1` |
| `agent.livenessProbe.initialDelaySeconds` |  | `300` |
| `agent.livenessProbe.periodSeconds` |  | `10` |
| `agent.livenessProbe.failureThreshold` |  | `3` |
| `agent.livenessProbe.successThreshold` |  | `1` |
| `agent.nodeSelector` |  | `{}` |
| `agent.tolerations` |  | `[]` |
| `agent.affinity` |  | `{}` |
| `agent.podAnnotations` |  | `{}` |
| `agent.extraVolumes` |  | `[]` |
| `agent.extraVolumeMounts` |  | `[]` |
| `agent.extraInitContainers` |  | `[]` |
| `agent.extraEnv` |  | `[]` |
| `agent.mountSharedVolume.enabled` | Mounts a existing RWX volume to karaf/deploy (i.e. to provide JDBC drivers). | `false` |
| `agent.mountSharedVolume.claimName` |  | `"sm-shared-pvc"` |
| `db2TestInstance.enabled` | Installs a single replica Db2 container with default settings. IMPORTANT: It is not intended for production use, but only for testing and training. | `false` |
| `db2TestInstance.image.repository` |  | `"docker.io/ibmcom/db2"` |
| `db2TestInstance.image.tag` |  | `"11.5.5.0"` |
| `db2TestInstance.image.pullPolicy` |  | `"IfNotPresent"` |
| `db2TestInstance.persistence.existingClaim` |  | `""` |
| `db2TestInstance.persistence.enabled` |  | `false` |
| `db2TestInstance.persistence.storageClass` |  | `"-"` |
| `db2TestInstance.persistence.accessModes` |  | `["ReadWriteOnce"]` |
| `db2TestInstance.persistence.size` |  | `"8Gi"` |
| `db2TestInstance.persistence.annotations` |  | `{}` |


# 5. Configuration Examples

## 5.1. SM Server Persistence

### 5.1.1. Via H2 Database on a Persistent Volume

SM Server manages the monitoring and the configuration database in local H2 databases by default. SM Server can use a Persistent Volume Claim to persist data such as configuration data and historical event data. If persistence is disabled, data will be written on the local node on a volume of type [`emptyDir`](https://kubernetes.io/docs/concepts/storage/volumes/#emptydir) to allow the data to survive Pod restarts. However, with `emptyDir` all data will vanish if a Pod is removed.

In order to persist both of these databases onto a Persistent Volume Claim, use the `server.persistence` configuration of the chart. Example:
```
[...]
server:
[...]
  persistence:
    enabled: true
    storageClass: "my-storage-class"
    accessModes:
      - ReadWriteOnce
    size: 16Gi
[...]
```

It is also possible to use an existing volume created beforehand. This useful if the required StorageClass does not allow auto-provisioning through Volume Claim Templates. Example:
```
[...]
server:
[...]
  persistence:
    existingClaim: "my-pvc-123"
[...]
```

__Notice on High Availability:__ Using a Persistent Volume with StorageClass `hostPath` or a similar StorageClass will tie the SM Server Pod to the node it is scheduled to for the first time after its installation. If this node is removed from the cluster or if the node goes down, the SM Server Pod cannot be scheduled to another node, because its persisted data still sits on the unreachable node. To make your setup resilient against this failure situation you can:
* use StorageClasses providing automated replication
* use StorageClasses backed by a central storage system (like a Network File Systems)
* use an external database for persistence (see [guidance in this document](#via-external-database-services))


### 5.1.2. Via External Database Services

Instead of persisting data on mounted volumes, an external database service can be used. Two separate databases are used for data persistence whereat each of them is configured individually.
* Configuration Database
* Monitoring Database

Create secrets containing user and password for database access. Below examples make use a Db2 test instance (see [dedicated chapter](#install-a-db2-database-for-testing-scenarios)):
```
# Credentials for configuration database
kubectl create secret generic creds-configuration-db \
  --from-literal=user='db2inst1' \
  --from-literal=password='s3cr3t'  

# Credentials for monitoring database
kubectl create secret generic creds-monitoring-db \
  --from-literal=user='db2inst1' \
  --from-literal=password='s3cr3t'
```

Then add the required configuration parameter to the settings:
```
[...]
  externalConfigDatabase:
    enabled: true
    jdbcUrl: "jdbc:db2://mydb2:50000/smdb"
    jdbcDriverClass: "com.ibm.db2.jcc.DB2Driver"
    jdbcSecret: creds-configuration-db
    
  externalMonitoringDatabase:
    enabled: true
    jdbcUrl: "jdbc:db2://mydb2:50000/smdb"
    jdbcDriverClass: "com.ibm.db2.jcc.DB2Driver"
    jdbcSecret: creds-monitoring-db
[...]
```

## 5.2. Dependency Injection

In some scenarios external dependencies have to be made available in the SM containers. Examples:
* JDBC drivers for monitoring specific databases
* JDBC drivers for using a specific databases as SM Server backend
* Libraries required certain SM Probes (i.e. for monitoring Content Process Engine)

As containers are of ephemeral nature, the Helm Chart supports multiple options to inject file dependencies into SM containers:
* __Extended SM Container Images:__ Extend the official SM images (SM Server & SM Agent).
* __Shared Volume:__ Provide files on a volume used by multiple Pods.

Below examples work with a Db2 JDBC Driver, which can be downloaded from the IBM website with an IBM account:
* Version: [Db2 11.5 GA](https://www.ibm.com/support/pages/node/387577)
* Package: [IBM Data Server Driver for JDBC and SQLJ (JCC Driver)](https://epwt-www.mybluemix.net/software/support/trial/cst/welcomepage.wss?siteId=854&tabId=1958&w=1&_ga=2.54861677.791151574.1612524773-1645088533.1612524773)

Extract the downloaded package:
```
# Extract zip file containing the actual driver from tgz file
tar -zxvf db2_db2driver_for_jdbc_sqlj_v11.5.tar.gz jdbc_sqlj/db2_db2driver_for_jdbc_sqlj.zip

# Extract jar file from zip file
unzip -j "jdbc_sqlj/*.zip" "db2jcc4.jar" -d "."
```

You are now prepared to use one of the two subsequently proposed options.

### 5.2.1. Option: Extended SM Container Images
Derive a new image based on the existing standard SM container images by adding additional files.

Create a Dockerfile similar to the following example:
```
# Define the required base image
FROM smserver:5.5.5.0-002

# Make sure to perform further actions with the correct user id
USER sm

# Copy the local JDBC driver file into the correct directory of the SM Server container
COPY ./db2jcc4.jar /opt/sm/server/karaf/deploy/db2jcc4.jar
```

Run the build:
```
docker build -t smserver:5.5.5.0-002-jdbc .
```


### 5.2.2. Option: Shared Volume

This option leverages a Persistent Volume of type `ReadWriteMany` (`RWX`) which can be mounted by all SM container in the cluster to access the same set of dependencies. `Please note:` RWX volumes requires a StorageClass to be present in the cluster which supports `ReadWriteMany` access (i.e. `azurefile` on Azure, `efs` on AWS or OpenShift Container Storage for OCP).

As a first step we need to create a `RWX` volume. Before applying the manifest, set a `storageClassName` available on the cluster at hand:
```
kubectl apply -f sm/examples/shared_volume/rwx_pvc.yaml
```

Next, we will start an auxiliary Pod which mounts this volume. Before applying the manifest, set the `image` property to a container image accesible for your cluster (i.e. the `smserver` or `smagent` image):
```
kubectl apply -f sm/examples/shared_volume/aux_pod.yaml
```

Now we use the auxiliary Pod to copy any dependencies from our local system (such as a JDBC driver) onto the volume.
```
kubectl cp drivers/db2jcc4.jar sm-shared-pvc-aux:/deploy
```

Delete the auxiliary Pod (not needed anymore): 
```
kubectl delete pod sm-shared-pvc-aux
```

To make the SM containers use this prepared volume a few settings have to be enablled before installing or upgrading the chart:

For the SM Server:
* Set `server.mountSharedVolume.enabled` to `true` (default: `false`)
* Set `server.mountSharedVolume.claimName` if a custom name was used (default: `sm-shared-pvc`)

For the SM Agent:
* Set `agent.mountSharedVolume.enabled` to `true` (default: `false`)
* Set `agent.mountSharedVolume.claimName` if a custom name was used (default: `sm-shared-pvc`)

This installation command enabled usage of the shared volume for both server and agent:
```
helm install mysm sm-5.5.5-0-003.tgz -f sm/examples/values/default_persistent.yaml \
  --set server.mountSharedVolume.enabled=true \
  --set agent.mountSharedVolume.enabled=true
```

## 5.3. Restricted Pod Permissions

Containers of SM versions above 5.5.6.0 are running in non-root mode. This means that they can work with a strict PodSecurityPolicy like the one below (`very-restrictive-policy`).

__IMPORTANT:__ For version 5.5.6.0 and below it is required to create the appropriate RBAC resources, in case the PodSecurityPolicy Admission Plugin is active for the Kubernetes cluster at hand.

```
apiVersion: policy/v1beta1
kind: PodSecurityPolicy
metadata:
  name: very-restrictive-policy
spec:
  privileged: false
  allowPrivilegeEscalation: false
  requiredDropCapabilities:
    - ALL
  volumes:
    - 'configMap'
    - 'emptyDir'
    - 'projected'
    - 'secret'
    - 'downwardAPI'
    - 'persistentVolumeClaim'
  hostNetwork: false
  hostIPC: false
  hostPID: false
  runAsUser:
    rule: 'MustRunAsNonRoot'
  seLinux:
    rule: 'MustRunAs'
    # Range 1 - 65535 ensures that root cannot be used (uid 0)
    ranges:
      - min: 1
        max: 65535
  supplementalGroups:
    rule: 'MustRunAs'
    ranges:
      - min: 1
        max: 65535
  fsGroup:
    rule: 'MustRunAs'
    ranges:
      - min: 1
        max: 65535
  readOnlyRootFilesystem: false
```

### 5.3.1. Option A: RBAC Resources Created by the Chart

Set the charts's property `createRbacResources` to `true`. This will create the required RBAC resources for you automatically on installation. __Attention:__ This option requires that the user you are operating with (i.e. run `helm` and `kubectl` commands) is a `cluster-admin`.

```
[...]
createRbacResources: true
[...]
```

### 5.3.2. Option B: RBAC Resources Created by the User

If the user id you are using to work with the cluster does not have the role `cluster-admin` bound, ask the Cluster's Administrator to create the required RBAC resources for you. Examples for the required RBAC resources can be found in `sm/examples/rbac`.

Among the resources to be created is a `ServiceAccount`. As soon as the RBAC resources are available in the namespace dedicated for the installation, add the respective `ServiceAccount` to the Chart settings:
```
[...]
createRbacResources: false
[...]

server:
[...]
  serviceAccountName: your-account
[...]

agent:
[...]
  serviceAccountName: your-account
[...]
```


# 6. Troubleshooting

## 6.1. Entering the SM Server Pod

```
kubectl exec -it mysm-smserver-0 bash
```

## 6.2. Client-side Chart Testing

Use flags `--dry-run=client` and `--debug` for testing purposes.

## 6.3. Failing Probes on Startup

If the SM Server Pod does not have enough CPU resources allocated the startup process time may increase to a time period so long, that the default `readinessProbe` prematurely diagnoses a failed startup and consequently restarts the Pod for another try. In this case you can:
* Increase the allocated CPU resources in `server.resources.limits.cpu` to accelerate the startup process.
* Increase the values `server.readinessProbe.initialDelaySeconds` and `server.livenessProbe.initialDelaySeconds` to delay the probes and allow a longer startup time.

## 6.4. Job "mysm-smserver-set-password" Fails on Fresh Installations

`helm uninstall` does not remove existing Persistent Volume Claims used by the SM Server. If a SM Server has already been installed in the cluster with persistence activated at an earlier point in time, its volume is possibly still present. Ensure that after uninstalling the volume `data-mysm-smserver-0` is also deleted before starting a new installation.



# 7. Advanced

## 7.1. Container Image Preloading

Clusters for development and testing purposes may not have access to a container image registry providing the SM container images. In these environments the images can be loaded into the container runtime manually.

### 7.1.1. On Docker Runtime

Docker is a supported runtime on:
* OpenShift 4.x on RHEL 7
* Many Kubernetes distributions

1) Import compressed or uncompressed images:
```
docker load --input smserver.tgz
docker load --input smagent.tgz
```

2) Check if images are available:
```
docker images
```

### 7.1.2. On Containerd Runtime

Containerd is a supported runtime on:
* Many Kubernetes distributions (i.e. `k3s`)

This specific example was specifically tested with `k3s`.

1) Extract compressed images:
```
gunzip -c smserver.tgz > smserver.tar
gunzip -c smagent.tgz > smagent.tar
```

2) Set variable for shorter 'ctr' commands:
```
CTR="sudo ctr -n k8s.io -a /run/k3s/containerd/containerd.sock"
```

3) Import uncompressed images:
```
$CTR images import smserver.tar
$CTR images import smagent.tar
```

4) List imported images:
```
$CTR images ls | grep -E 'smserver|smagent'
```

### 7.1.3. On CRI-O Runtime

CRI-O is the [default container runtime](https://www.redhat.com/en/blog/red-hat-openshift-container-platform-4-now-defaults-cri-o-underlying-container-engine) for OpenShift 4.x on RHEL 8.

__Notice:__ If CRI-O is running as non-root, `remove` sudo from the commands shown below.


1) Extract compressed images:
```
gunzip -c smserver.tgz > smserver.tar
gunzip -c smagent.tgz > smagent.tar
```

2) Import uncompressed images:
```
cat smserver.tar | sudo podman load
cat smagent.tar | sudo podman load
```

3) List imported images:
```
sudo crictl images | grep -E 'smserver|smagent'
```

### 7.1.4. Into a k3s Cluster managed by k3d
Steps to import images into a k3s cluster created and managed by [k3d](https://github.com/rancher/k3d):

1) Extract compressed images:
```
gunzip -c smserver.tgz > smserver.tar
gunzip -c smagent.tgz > smagent.tar
```

2) Import extracted images:
```
k3d image import -c <your-cluster-name> smserver.tar smagent.tar
```


## 7.2. In-Cluster Db2 Database for Testing Scenarios

Using an external relational database system can be tested by deploying a Db2 database container in the same cluster as SM. Be advised, that this is __not recommended for production use!__ In production environments, either persist the built-in H2 database onto a Persistent volume, or use an external database system outside of the cluster.

### 7.2.1. Caveats
For testing and learning purposes this Helm Chart contains a template for installing [Db2](https://hub.docker.com/r/ibmcom/db2) as a a __single-replica Pod__. Be aware of the following caveats of this DB2 test deployment:

* Data is persisted onto an `emptyDir` volume by default. If no other type of volumes are configured, the __Db2 database state will be lost if the Pod is removed__.
* The Container Security Context is configured with `privileged: true` and the Pod Security Context is configured with `fsGroup: 1000` ([see more details](https://www.ibm.com/cloud/blog/how-to-running-ha-ibm-db2-on-kubernetes)). This cannot be changed and __without these evelated permissions the Db2 container will not work__.
* As installing the container takes a while, the __probes of SM Server will be delayed for 5 minutes__ before checking the SM Server container's health to avoid timeouts.

### 7.2.2. Activation
To enable the deployment of the Db2 test instance, use following settings:
```
[...]
db2TestInstance:
  enabled: true
[...]
```
Refer to the Configuration Reference to see parameters to control the used container image.

Since this Db2 setup is intended for testing purposes only, the database's access credentials are hardcoded in this chart. Database and user id are created automatically:
* Database name: `smdb`
* User (instance admin): `db2inst1`
* Password (instance admin): `s3cr3t`
* Hostname: `db2`
* Port: `50000`

To use this database to install a SM Server for testing and educational purposes, refer to chapter the [respective guidance in this document](#via-external-database-services).


## 7.3. Exposing SM Server to External Agents With NGINX Ingress

A service of type `LoadBalancer` is created if `server.loadBalancer.enabled to` is set to `true` (default).

__When do I need this?__ In some scenarios it may be required to install SM Agents in another Kubernetes cluster than the SM Server or for example directly on a Linux/AIX/Windows instance. To make it possible for these external agents to connect to an SM Server within a Kubernetes cluster, additional measures have to be taken. While connecting into the cluster using the HTTP protocol (i.e. in order to use the SM Web UI) can be covered with an Ingress, SM Agents use the MQTT protocol to communicate with the SM Server. As MQTT is a TCP-based protocol, standard Ingress resources are not applicable for this use case, but Services of type `LoadBalancer` are the appropriate approach.

Verify the existence and binding of the new service (replace `mysm` with the name of your release): `kubectl get service mysm-smserver-mqtt-loadbalancer`
```
NAMESPACE      NAME                                TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)                               AGE
default        mysm-smserver-mqtt-loadbalancer     LoadBalancer   10.43.160.30    172.18.0.2    1883:31791/TCP,8883:30122/TCP         4m29s
```
Depending on the type of Load Balancer the output looks different. If an External IP is displayed, you should be able to connect with an Agent from outside the Kubernetes cluster.

If you are working with a local development cluster (like Minikube, K3S, K3D, or Microk8s) a simple SM Agent Docker container can be used to test an external connection:
```
docker run -d --name extagent01 \
    --net=host \
    -e SERVER_HOSTNAME='127.0.0.1' \
    -e SERVER_PORT=1883 \
    -e CLIENT_ID='extagent01' \
    smagent:5.5.5.0-003
```
For connections through a Load Balancer from an external network, use the External IP displayed with `kubectl get service mysm-smserver-mqtt-loadbalancer`.