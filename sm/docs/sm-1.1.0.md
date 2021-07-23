# 1. Helm Chart for Enterprise System Monitor / Service Monitor

<!-- TOC -->

- [1. Helm Chart for Enterprise System Monitor / Service Monitor](#1-helm-chart-for-enterprise-system-monitor--service-monitor)
- [2. Prerequisites](#2-prerequisites)
    - [2.1. Compatibility](#21-compatibility)
    - [2.2. RBAC](#22-rbac)
- [3. Installation](#3-installation)
    - [3.1. Default Installation (SM Server & SM Agent)](#31-default-installation-sm-server--sm-agent)
    - [3.2. SM Server Only Installation](#32-sm-server-only-installation)
    - [3.3. SM Agent Only Installation](#33-sm-agent-only-installation)
    - [3.4. Access to SM UI via Ingress or Route](#34-access-to-sm-ui-via-ingress-or-route)
    - [3.5. Cleanup](#35-cleanup)
        - [3.5.1. Uninstall Release](#351-uninstall-release)
        - [3.5.2. Delete Persistent Volumes](#352-delete-persistent-volumes)
        - [3.5.3. Delete Secrets & ConfigMaps](#353-delete-secrets--configmaps)
    - [3.6. Upgrades](#36-upgrades)
    - [3.7. Installation on OpenShift](#37-installation-on-openshift)
        - [3.7.1. Push Images to IBM Container Registry](#371-push-images-to-ibm-container-registry)
        - [3.7.2. Cluster Login](#372-cluster-login)
        - [3.7.3. Prepare a Project](#373-prepare-a-project)
        - [3.7.4. Create Image Pull Secret](#374-create-image-pull-secret)
        - [3.7.5. Create RBAC Resources](#375-create-rbac-resources)
        - [3.7.6. Configure Chart](#376-configure-chart)
        - [3.7.7. Install & Verify](#377-install--verify)
- [4. Configuration Reference](#4-configuration-reference)
- [5. Configuration Examples](#5-configuration-examples)
    - [5.1. SM Server Persistence](#51-sm-server-persistence)
        - [5.1.1. Via H2 Database on a Persistent Volume](#511-via-h2-database-on-a-persistent-volume)
        - [5.1.2. Via External Database Services](#512-via-external-database-services)
    - [5.2. Dependency Injection](#52-dependency-injection)
        - [5.2.1. Option: Auto-Download on Installation](#521-option-auto-download-on-installation)
        - [5.2.2. Option: Extended SM Container Images](#522-option-extended-sm-container-images)
        - [5.2.3. Option: Shared Volume](#523-option-shared-volume)
    - [5.3. Restricted Pod Permissions](#53-restricted-pod-permissions)
        - [5.3.1. Option A: RBAC Resources Created by the Chart](#531-option-a-rbac-resources-created-by-the-chart)
        - [5.3.2. Option B: RBAC Resources Created by the User](#532-option-b-rbac-resources-created-by-the-user)
        - [5.3.3. Example of a Minimum Policy](#533-example-of-a-minimum-policy)
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
    - [7.2. In-Cluster Db2 or SQL Server Database for Testing Scenarios](#72-in-cluster-db2-or-sql-server-database-for-testing-scenarios)
    - [7.3. Exposing SM Server to External Agents With NGINX Ingress](#73-exposing-sm-server-to-external-agents-with-nginx-ingress)
    - [7.4. Set an Admin Password on Chart Installation (Optional)](#74-set-an-admin-password-on-chart-installation-optional)

<!-- /TOC -->


# 2. Prerequisites

## 2.1. Compatibility

* Helm 3 is required. Helm 2 is __not__ supported.
* Kubernetes 1.15.x and higher.
* OpenShift 4.x.
* The chart can be installed multiple times per namespace.

## 2.2. RBAC

* For versions below 5.5.5.1-001 it is required to create the appropriate RBAC resources, in case the PodSecurityPolicy Admission Plugin is active for the Kubernetes cluster at hand (see more [details](#restricted-pod-permissions)). On OpenShift it is additionally required to run the containers in privileged mode for SM version lower than 5.5.5.1-001 (see more [details](#installation-on-openshift))
* For SM version 5.5.5.1-001 and above no specific RBAC resources have to be created.

# 3. Installation

Add this chart's repository and run a repository update:
```
helm repo add cenit https://cenit-ag.github.io/helm-charts
helm repo update
```

## 3.1. Default Installation (SM Server & SM Agent)

When using the default values of the chart, a SM Server with one SM Agent will be installed, but most features are disabled that would require configuration specific to each cluster (in example Persistent Storage and Ingress). For this variety the only mandatory configuration are references to the container images.

A `values_custom.yaml` file could look like this:
```
server:
  image:
    repository: someregistry.example.com/sm/smserver
    tag: 5.5.5.1-001
agent:
  image:
    repository: someregistry.example.com/sm/smagent
    tag: 5.5.5.1-001
```

The following command installs a Helm release called `mysm` using the Helm chart `sm` from registry `cenit` and uses installation parameters from `values_custom.yaml`:
```
helm install mysm cenit/sm -f values_custom.yaml
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
helm install mysm cenit/sm -f values_custom.yaml
```

Instead of disabling the SM Agent in a values file, another option is to pass the setting via a `--set` argument to Helm:
```
helm install mysm cenit/sm -f values_custom.yaml \
    --set server.enabled=true \
    --set agent.enabled=false
```
__NOTICE:__ Values set via `--set` take precedence over any values mentioned in a values file passed with `-f`.

If you want to add a SM Agent to this release at a later point of time you can:
* ... [change](#upgrades) the existing release using `helm upgrade`.
* ... install a SM Agent in a [separate Helm release](#sm-agent-only-installation) and connect it to the existing SM Server.


## 3.3. SM Agent Only Installation

You can install a SM Agent for an existing SM Server. It doesn't matter whether the SM Server was previously installed with or without a SM Agent enabled. Below command installs a separate release containing a SM Agent only using `--set`.
```
helm install mysm-2nd-agent cenit/sm -f values_custom.yaml \
    --set server.enabled=false \
    --set agent.enabled=true \
    --set server.hostname=mysm-smserver
```
The parameter `server.hostname` is mandatory in this scenario and has to point to an existing SM Server. In this example, it points to the existing SM Server installed with Helm release `mysm` within the same namespace. The pattern for the hostname is: `<release name of SM Server>-smserver`

If the SM Agent shall be installed in a namespace different from the SM Server use a pattern like this: `<release name of SM Server>-smserver.<namespace>`. Example: `mysm-smserver.some-namespace`

## 3.4. Access to SM UI via Ingress or Route

By default the chart does not create Kubernetes Ingresses or OpenSift Routes. To enable access from outside the cluster, use one of the following:
* Set `server.ingress.enabled` to `true`
* Set `server.route.enabled` to `true`

With this configuration you will be able to access the cluster using: `https://<route-or-ingress-domain>/`

## 3.5. Cleanup

### 3.5.1. Uninstall Release

```
helm uninstall mysm
```

### 3.5.2. Delete Persistent Volumes

Required volumes are created automatically by the Helm Chart in case persistence is enabled for SM Server using `volumeClaimTemplates` or `existingClaim`. Helm [cannot automatically delete](https://github.com/helm/helm/issues/3313) such volumes on uninstallation of a release. To manually delete the SM Server volume run:
```
# Replace `mysm` with the name of your release.
kubectl delete pvc -l smInstance=mysm
```
__IMPORTANT:__ This deletes any state persisted to the volumes by the SM Server like monitoring data and manually accomplished configuration steps.


### 3.5.3. Delete Secrets & ConfigMaps

In addition you may want to remove remaining created Secrets and ConfigMaps.

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
docker tag smserver:5.5.5.1-001 de.icr.io/<your-namespace>/smserver:5.5.5.1-001
docker tag smagent:5.5.5.1-001 de.icr.io/<your-namespace>/smagent:5.5.5.1-001
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
docker push de.icr.io/cenit/smserver:5.5.5.1-001
docker push de.icr.io/cenit/smagent:5.5.5.1-001
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

### 3.7.4. Create Image Pull Secret

The `default` project already contains an Image Pull Secret to allow access to the Container Registry. For customly created projects this is [not the case](https://cloud.ibm.com/docs/openshift?topic=openshift-registry#other_registry_accounts).

Copy the existing secret to the custom project/namespace (in this example we are using namespace/project `sm-test`):
```
oc get secret all-icr-io -n default -o yaml | sed 's/default/sm-test/g' | oc create -n sm-test -f -
```

### 3.7.5. Create RBAC Resources

For versions below 5.5.5.1-001  the SM container images partially still require root access during execution. As privileged execution of containers in disabled by default on OpenShift clusters, we need to enable this using a Service Account.

Create a Service Account:
```
oc create serviceaccount sm-svc-acc
```

Add the created account to the `privileged` Security Context Constraints 
```
oc adm policy add-scc-to-user privileged -z sm-svc-acc
```

### 3.7.6. Configure Chart

To deploy SM on OpenShift instead of Kubernetes several adjustments are required to the configuration values:

* `platform` must be set to `ocp`.
* `server.serviceAccountName` must be set to the service account created in [Create RBAC Resources](#create-RBAC-Resources)
* `agent.serviceAccountName` must be set to the service account created in [Create RBAC Resources](#create-RBAC-Resources)
* `server.image.repository` needs to point to the provided trough the IBM Container Registry
* `agent.image.repository` needs to point to the provided trough the IBM Container Registry
* `imagePullSecrets` has to include a pull secret working for the IBM Container Registry (i.e. `all-icr-io`)
* `server.persistence.storageClass` has to be set to a valid Storage Class providing block storage (i.e. `ibmc-block-bronze` - see [official reference](https://cloud.ibm.com/docs/containers?topic=containers-block_storage) for storage classes on IBM Cloud).
* `server.persistence.size` has to be set to a valid minimum size for the selected Storage Class (i.e. for `ibmc-block-bronze` the minimum is `20Gi`).
* `server.route.*` has to be configured (see below example)

The final values file should look like this (`roks.yaml`):
```
platform: ocp

imagePullSecrets:
  - name: all-icr-io

server:

  image:
    repository: someregistry.example.com/sm/smserver
    tag: 5.5.5.1-001

  serviceAccountName: sm-svc-acc
  
  persistence:
    enabled: true
    storageClass: ibmc-block-bronze
    accessModes:
      - ReadWriteOnce
    size: 20Gi
  
  route:
    enabled: true
    tls:
      insecureEdgeTerminationPolicy: Redirect
      termination: passthrough
    useHttpsEndpoint: true

agent:

  image:
    repository: someregistry.example.com/sm/smagent
    tag: 5.5.5.1-001

  serviceAccountName: sm-svc-acc
```

### 3.7.7. Install & Verify

Install using Helm:
```
helm install mysm cenit/sm -f roks.yaml
```

With this configuration the SM UI will be accessible with an URL similar to this: `https://<release-name>-smserver-<namespace/project>.<cluster-domain>`

# 4. Configuration Reference

The following table lists the configurable parameters of the SM chart and their default values.

The following table lists the configurable parameters of the Sm chart and their default values.


| Parameter                | Description             | Default        |
| ------------------------ | ----------------------- | -------------- |
| `createRbacResources` | Creates RBAC resources required if the default policies for Pods do not allow a rule 'runAsUser' = 'RunAsAny'. | `false` |
| `imagePullSecrets` | Optionally specify an array of imagePullSecrets for pulling images from a private container registry. Secrets must be manually created in the namespace. | `[]` |
| `serviceAccountName` | Name of exisiting service account. Overridden by 'serviceAccountName' explicitly defined for worksload (like i.e. 'server.serviceAccountName'). | `""` |
| `runPrivilegedContainers` | If true, all main containers managed by this chart will be executed in privileged mode. Overridden by 'runPrivilegedContainer' explicitly defined for worksload (like i.e. 'server.runPrivilegedContainer'). | `false` |
| `chartHookDeletionPolicy` | Control what happens with finished or failed jobs. Example: "before-hook-creation,hook-succeeded" | `"before-hook-creation"` |
| `nameOverride` |  | `""` |
| `fullnameOverride` |  | `""` |
| `server.enabled` | Creates a StatefulSet for a SM Server (enabled by default). | `true` |
| `server.runPrivilegedContainer` | Required for SM version =< 5.5.5.1-000 if Pod Security Policies do not allow root containers. | `false` |
| `server.image.repository` | Image repository, i.e. 'smserver' | `""` |
| `server.image.tag` | Image tag, i.e. '5.5.5.1-001' | `""` |
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
| `server.setPasswordJobEnabled` | If enabled, a job will automatically set a password for user 'admin' as defined in 'adminPasswordSecret'. | `false` |
| `server.adminPasswordSecret` | Existing secret containing the password for user 'admin' (key 'password' must contain password string). | `""` |
| `server.resourceDownload.enabled` | If enabled, the files in list resourceDownload.files will be downloaded and placed into '/opt/sm/server/karaf/deploy'. | `false` |
| `server.resourceDownload.files` | A list of files to be downloaded and provided to the container. If a file already exists, it will not be downloaded. | `["https://repo1.maven.org/maven2/com/ibm/db2/jcc/11.5.5.0/jcc-11.5.5.0.jar", "https://repo1.maven.org/maven2/com/microsoft/sqlserver/mssql-jdbc/9.2.1.jre8/mssql-jdbc-9.2.1.jre8.jar"]` |
| `server.terminationGracePeriodSeconds` | Waiting period before a terminating pod is forcefully ended. Default value is 30 (for 30 seconds). | `30` |
| `server.resources` | No requests or limits set by default. | `{}` |
| `server.readinessProbe.initialDelaySeconds` |  | `180` |
| `server.readinessProbe.periodSeconds` |  | `20` |
| `server.readinessProbe.failureThreshold` |  | `9` |
| `server.readinessProbe.successThreshold` |  | `1` |
| `server.readinessProbe.timeoutSeconds` |  | `3` |
| `server.livenessProbe.initialDelaySeconds` |  | `360` |
| `server.livenessProbe.periodSeconds` |  | `10` |
| `server.livenessProbe.failureThreshold` |  | `3` |
| `server.livenessProbe.successThreshold` |  | `1` |
| `server.livenessProbe.timeoutSeconds` |  | `3` |
| `server.nodeSelector` |  | `{}` |
| `server.tolerations` |  | `[]` |
| `server.affinity` |  | `{}` |
| `server.podAnnotations` |  | `{}` |
| `server.podSecurityContext` |  | `{}` |
| `server.securityContext` |  | `{}` |
| `server.ingress.enabled` |  | `false` |
| `server.ingress.annotations` |  | `{}` |
| `server.ingress.hosts` |  | `[{"host": "*", "paths": ["/"]}]` |
| `server.ingress.tls` |  | `[]` |
| `server.ingress.useHttpsEndpoint` | If true (default), Ingress will connect to the secure (HTTPS) endpoint on the SM Server Pod. True by default. False will also disable the HTTPS endpoint of the SM Server container. | `true` |
| `server.route.enabled` |  | `false` |
| `server.route.tls` |  | `{}` |
| `server.route.useHttpsEndpoint` | If true (default), Ingress will connect to the secure (HTTPS) endpoint on the SM Server Pod. True by default. | `true` |
| `server.persistence.existingClaim` | If 'existingClaim' is defined, a PVC must be created manually before volume will be bound | `""` |
| `server.persistence.enabled` |  | `false` |
| `server.persistence.storageClass` |  | `""` |
| `server.persistence.accessModes` |  | `["ReadWriteOnce"]` |
| `server.persistence.size` |  | `"8Gi"` |
| `server.persistence.annotations` |  | `{}` |
| `server.persistence.volumePermissions.enabled` |  | `false` |
| `server.extraVolumes` |  | `[]` |
| `server.extraVolumeMounts` |  | `[]` |
| `server.extraInitContainers` |  | `[]` |
| `server.extraEnv` |  | `[]` |
| `server.externalConfigurationDatabase.enabled` | Enables usage of an external database for configuration data (DB2, SQL Server). If disabled, a local H2 database instance is used. | `false` |
| `server.externalConfigurationDatabase.type` | Valid values: 'db2' or 'sqlserver' | `""` |
| `server.externalConfigurationDatabase.jdbcUrl` |  | `""` |
| `server.externalConfigurationDatabase.jdbcDriverClass` |  | `""` |
| `server.externalConfigurationDatabase.jdbcSecret` |  | `""` |
| `server.externalMonitoringDatabase.enabled` | Enables usage of an external database for monitoring data (DB2, SQL Server). If disabled, a local H2 database instance is used. | `false` |
| `server.externalMonitoringDatabase.type` | Valid values: 'db2' or 'sqlserver' | `""` |
| `server.externalMonitoringDatabase.jdbcUrl` |  | `""` |
| `server.externalMonitoringDatabase.jdbcDriverClass` |  | `""` |
| `server.externalMonitoringDatabase.jdbcSecret` |  | `""` |
| `server.externalDependenciesVolume.persistence.existingClaim` |  | `""` |
| `server.externalDependenciesVolume.persistence.enabled` |  | `false` |
| `server.externalDependenciesVolume.persistence.storageClass` |  | `""` |
| `server.externalDependenciesVolume.persistence.accessModes` |  | `["ReadWriteOnce"]` |
| `server.externalDependenciesVolume.persistence.size` |  | `"1Gi"` |
| `server.externalDependenciesVolume.persistence.annotations` |  | `{}` |
| `agent.enabled` | Creates a Deployment with a single SM Agent Pod (enabled by default). | `true` |
| `agent.runPrivilegedContainer` | Required for SM version =< 5.5.5.1-000 if Pod Security Policies do not allow root containers. | `false` |
| `agent.image.repository` | Image repository, i.e. 'smagent' | `""` |
| `agent.image.tag` | Image tag, i.e. '5.5.5.1-001' | `""` |
| `agent.image.pullPolicy` |  | `"IfNotPresent"` |
| `agent.serviceAccountName` | Name of exisiting service account. If unset, the namespace's default service account is used. | `""` |
| `agent.logLevel` | Supported Log4J log levels: https://logging.apache.org/log4j/2.x/manual/customloglevels.html | `"ERROR"` |
| `agent.podSecurityContext` |  | `{}` |
| `agent.securityContext` |  | `{}` |
| `agent.terminationGracePeriodSeconds` |  | `30` |
| `agent.resources` | No requests or limits set by default. | `{}` |
| `agent.readinessProbe.initialDelaySeconds` |  | `180` |
| `agent.readinessProbe.periodSeconds` |  | `20` |
| `agent.readinessProbe.failureThreshold` |  | `9` |
| `agent.readinessProbe.successThreshold` |  | `1` |
| `agent.readinessProbe.timeoutSeconds` |  | `3` |
| `agent.livenessProbe.initialDelaySeconds` |  | `360` |
| `agent.livenessProbe.periodSeconds` |  | `10` |
| `agent.livenessProbe.failureThreshold` |  | `3` |
| `agent.livenessProbe.successThreshold` |  | `1` |
| `agent.livenessProbe.timeoutSeconds` |  | `3` |
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
| `db2TestInstance.runPrivilegedContainer` | The Db2 container only runs if executed in privileged mode. Therefore 'true' by default. | `true` |
| `db2TestInstance.serviceAccountName` | Name of exisiting service account. If unset, the namespace's default service account is used. | `""` |
| `db2TestInstance.image.repository` |  | `"docker.io/ibmcom/db2"` |
| `db2TestInstance.image.tag` |  | `"11.5.5.0"` |
| `db2TestInstance.image.pullPolicy` |  | `"IfNotPresent"` |
| `db2TestInstance.persistence.existingClaim` |  | `""` |
| `db2TestInstance.persistence.enabled` |  | `false` |
| `db2TestInstance.persistence.storageClass` |  | `""` |
| `db2TestInstance.persistence.accessModes` |  | `["ReadWriteOnce"]` |
| `db2TestInstance.persistence.size` |  | `"8Gi"` |
| `db2TestInstance.persistence.annotations` |  | `{}` |
| `db2TestInstance.resources` | No requests or limits set by default. | `{}` |
| `db2TestInstance.securityContext` |  | `{}` |
| `sqlServerTestInstance.enabled` | Installs a single replica SQL Server container with default settings. IMPORTANT: It is not intended for production use, but only for testing and training. | `false` |
| `sqlServerTestInstance.serviceAccountName` | Name of exisiting service account. Overridden by 'serviceAccountName' explicitly defined for worksload (like i.e. 'server.serviceAccountName'). | `""` |
| `sqlServerTestInstance.image.repository` |  | `"mcr.microsoft.com/mssql/server"` |
| `sqlServerTestInstance.image.tag` | Get a list of available tags from: https://mcr.microsoft.com/v2/mssql/server/tags/list | `"2019-latest"` |
| `sqlServerTestInstance.image.pullPolicy` |  | `"IfNotPresent"` |
| `sqlServerTestInstance.persistence.existingClaim` |  | `""` |
| `sqlServerTestInstance.persistence.enabled` |  | `false` |
| `sqlServerTestInstance.persistence.storageClass` |  | `""` |
| `sqlServerTestInstance.persistence.accessModes` |  | `["ReadWriteOnce"]` |
| `sqlServerTestInstance.persistence.size` |  | `"8Gi"` |
| `sqlServerTestInstance.persistence.annotations` |  | `{}` |
| `sqlServerTestInstance.resources` | No requests or limits set by default. | `{}` |
| `sqlServerTestInstance.securityContext` |  | `{}` |


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

Configuration example for Db2:
```
[...]
  externalConfigDatabase:
    enabled: true
    jdbcUrl: "jdbc:db2://mydb2:50000/smconfdb"
    jdbcDriverClass: "com.ibm.db2.jcc.DB2Driver"
    jdbcSecret: creds-configuration-db
    
  externalMonitoringDatabase:
    enabled: true
    jdbcUrl: "jdbc:db2://mydb2:50000/smmondb"
    jdbcDriverClass: "com.ibm.db2.jcc.DB2Driver"
    jdbcSecret: creds-monitoring-db
[...]
```

Configuration example for SQL Server:
```
[...]
  externalConfigDatabase:
    enabled: true
    jdbcUrl: "jdbc:sqlserver://mysqlserver:1433;databaseName=smconfdb"
    jdbcDriverClass: "com.microsoft.sqlserver.jdbc.SQLServerDriver"
    jdbcSecret: creds-configuration-db
    
  externalMonitoringDatabase:
    enabled: true
    jdbcUrl: "jdbc:sqlserver://mysqlserver:1433;databaseName=smmondb"
    jdbcDriverClass: "com.microsoft.sqlserver.jdbc.SQLServerDriver"
    jdbcSecret: creds-monitoring-db
[...]
```

## 5.2. Dependency Injection

In some scenarios external dependencies have to be made available in the SM containers. Examples:
* JDBC drivers for monitoring specific databases
* JDBC drivers for using a specific databases as SM Server backend
* Libraries required by certain SM Probes (i.e. for monitoring Content Process Engine)

As containers are of ephemeral nature, the Helm Chart supports multiple options to inject file dependencies into SM containers.

### 5.2.1. Option: Auto-Download on Installation

If your Kubernetes or OpenShift cluster has internet access, you can have the SM Server's Init Container download required files before the server starts. The download can take place onto a Persistent Volume, so that the download ideally only happens on the first start of the SM Server container. The volume is mounted on the path `/opt/sm/server/karaf/deploy`. If files like JDBC drivers reside in this directory, they are automatically loaded on start of SM Server. If the download procedure is enabled, the chart will by default download required JDBC drivers for SQL Server and Db2. 

Modify your chart configuration like this to enable the automated download:
```
server:
  resourceDownload:
    enabled: true
  externalDependenciesVolume:
    persistence: enabled
```
Refer to the Configuration Reference to see parameters to control the used container image.

### 5.2.2. Option: Extended SM Container Images

Derive a new image based on the existing standard SM container images by adding additional files.

Create a Dockerfile similar to the following example:
```
# Define the required base image
FROM smserver:5.5.5.1-001

# Make sure to perform further actions with the correct user id
USER sm

# Copy the local JDBC driver file into the correct directory of the SM Server container
COPY ./db2jcc4.jar /opt/sm/server/karaf/deploy/db2jcc4.jar
```

Run the build:
```
docker build -t smserver:5.5.5.1-001-jdbc .
```


### 5.2.3. Option: Shared Volume

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

### 5.3.3. Example of a Minimum Policy

The following PodSecurityPolicy has the minimum set of permissions defined required for the SM containers to work. Use this policy if the cluster has special policies in place that prevent SM from being deployed.

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


## 7.2. In-Cluster Db2 or SQL Server Database for Testing Scenarios

You can have the chart deploy and auto-configure either a Db2 or a SQL Server test instance. The database containers can have persistence enabled, but will run as single instances only.

Check the Configuration Reference for more details (`db2TestInstance`, `sqlServerTestInstance`).

__IMPORTANT:__ Due to license agreements and the nature of the deployment this must not be used in Production environment!

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
    smagent:5.5.5.1-001
```
For connections through a Load Balancer from an external network, use the External IP displayed with `kubectl get service mysm-smserver-mqtt-loadbalancer`.

## 7.4. Set an Admin Password on Chart Installation (Optional)

To secure the built-in `admin` user, it is recommended to set its password (default value is `admin`) to a secure password. This can be done after the first login to the UI using the System Administration settings. Please note, that setting a password in the UI is only persistent in case you have configured [Persistent Storage for the SM Server](#persistent-storage-for-sm-server).

If the password should be set automatically on installation of SM Server, create a secret containing the new password before installing or upgrading:
```
kubectl create secret generic my-sm-admin-pw \
  --from-literal=password='s3cr3t'
```
To enable the automatic setting of the password:
* refer to this password using key `server.adminPasswordSecret` .
* enable the responsible job by setting `server.setPasswordJobEnabled` to `true`.
These changes can be made either in your custom `values.yaml` file or using Helm's `--set` argument.