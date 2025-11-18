# 1. Helm Chart for Enterprise System Monitor / Service Monitor

- [1. Helm Chart for Enterprise System Monitor / Service Monitor](#1-helm-chart-for-enterprise-system-monitor--service-monitor)
- [2. Compatibility](#2-compatibility)
- [3. Installation](#3-installation)
  - [3.1. Default Installation (SM Server \& SM Agent)](#31-default-installation-sm-server--sm-agent)
  - [3.2. SM Server Only Installation](#32-sm-server-only-installation)
  - [3.3. SM Agent Only Installation](#33-sm-agent-only-installation)
  - [3.4. Access to SM UI via Ingress or Route](#34-access-to-sm-ui-via-ingress-or-route)
  - [3.5. Cleanup](#35-cleanup)
    - [3.5.1. Uninstall Release](#351-uninstall-release)
    - [3.5.2. Delete Persistent Volumes](#352-delete-persistent-volumes)
    - [3.5.3. Delete Secrets \& ConfigMaps](#353-delete-secrets--configmaps)
  - [3.6. Upgrades](#36-upgrades)
  - [3.7. Installation on OpenShift](#37-installation-on-openshift)
    - [3.7.1. Push Images to IBM Container Registry](#371-push-images-to-ibm-container-registry)
    - [3.7.2. Cluster Login](#372-cluster-login)
    - [3.7.3. Prepare a Project](#373-prepare-a-project)
    - [3.7.4. Create Image Pull Secret](#374-create-image-pull-secret)
    - [3.7.5. Configure Chart](#375-configure-chart)
    - [3.7.6. Install \& Verify](#376-install--verify)
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
  - [5.4. Provide Root Certificates to SM Server](#54-provide-root-certificates-to-sm-server)
    - [5.4.1. Provide Certificate as Secret](#541-provide-certificate-as-secret)
    - [5.4.2. Refer to Certificate Secret in Chart Configuration](#542-refer-to-certificate-secret-in-chart-configuration)
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
  - [7.5. Prepare SM for Monitoring of Mounted Logs](#75-prepare-sm-for-monitoring-of-mounted-logs)

__NOTICE:__ In the following 'SM' is short for 'System Monitor' aka 'Service Monitor'.

The following describes in detail the installation and configuration of SM Server and SM Agent in a cloud environment like
Kubernetes and OpenShift.

# 2. Compatibility

* Helm 3 is mandatory. Helm 2 is __not__ supported.
* Kubernetes 1.30 and higher is supported. Tested with 1.32 in IBM Cloud.
* OpenShift 4.15 and higher is supported. Tested with 4.19 in IBM Cloud.
* The chart can be installed multiple times per namespace.

# 3. Chart Installation

Add this chart's repository and run a repository update:
```
helm repo add cenit https://cenit-ag.github.io/helm-charts
helm repo update
```

## 3.1. Default Installation (SM Server & SM Agent)

When using the default values of the chart, an SM Server with one SM Agent will be
installed, but most features are disabled that would require configuration specific to
each cluster (for example Persistent Storage and Ingress). For this variety, the only
mandatory settings are references to the container images.

A `values_custom.yaml` file could look like this:
```
server:
  image:
    repository: someregistry.example.com/sm/smserver
    tag: 5.7.0.0-001
agent:
  image:
    repository: someregistry.example.com/sm/smagent
    tag: 5.7.0.0-001
```

The following command installs a Helm release called `mysm` using the Helm chart `sm` from
registry `cenit` and uses installation parameters from `values_custom.yaml`:
```
helm install mysm cenit/sm -f values_custom.yaml
```

## 3.2. SM Server Only Installation

To only install the SM Server without an SM Agent, set `agent.enabled` to `false`.
The setting `server.enabled` is `true` by default, but set in the example anyway for more
clarity.

Exmaple: Disable the SM Agent using a `values_custom.yaml` file:
```
[...]
server:
  enabled: true
[...]
agent:
  enabled: false
[...]
```
__NOTICE:__ `[...]` are only placeholders for this documentation to make the snippets easier to read.

Apply this configuration by executing:
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

If you want to add an SM Agent to this release at a later point of time you can:
* ... [change](#upgrades) the existing release using `helm upgrade`.
* ... install an SM Agent in a [separate Helm release](#sm-agent-only-installation) and connect it to the existing SM Server.


## 3.3. SM Agent Only Installation

You can install an SM Agent for an existing SM Server. It doesn't matter whether the SM
Server was previously installed with or without an SM Agent enabled. Below command
installs a separate release containing an SM Agent only using `--set`.
```
helm install mysm-2nd-agent cenit/sm -f values_custom.yaml \
    --set server.enabled=false \
    --set agent.enabled=true \
    --set server.hostname=mysm-smserver
```
The parameter `server.hostname` is mandatory in this scenario and has to point to an
existing SM Server. In this example, it points to the SM Server installed with Helm
release `mysm` within the same namespace. The pattern for the hostname is: `<release name of SM Server>-smserver`

If the SM Agent shall be installed in a namespace different from the SM Server use a
pattern like this: `<release name of SM Server>-smserver.<namespace>`. You can think of the
namespace as a domain specifier. Example: `mysm-smserver.some-namespace`.


## 3.4. Access to SM UI via Ingress or Route

__NOTICE:__ How to create an ingress or a route is not in the scope of this document.

By default the chart does not create Kubernetes Ingresses or OpenSift Routes. To enable
access from outside the cluster, use one of the following:
* Set `server.ingress.enabled` to `true`
* Set `server.route.enabled` to `true`

With this configuration you will be able to access the cluster using
`https://<route-or-ingress-domain>/` if the ingress or route is configured correctly.


## 3.5. Cleanup

### 3.5.1. Uninstall Release

```
helm uninstall mysm
```

### 3.5.2. Delete Persistent Volumes

__IMPORTANT:__ The following deletes any state persisted to the volumes by the SM Server like monitoring data and manually completed
configuration steps.

Required volumes are created automatically by the Helm Chart in case persistence is
enabled for SM Server using `volumeClaimTemplates` or `existingClaim`.
Helm [does not automatically delete](https://github.com/helm/helm/issues/3313) such volumes
on uninstallation of a release.

To manually delete the SM Server volume run:
```
kubectl delete pvc -l smInstance=mysm
```
Replace `mysm` with the name of your release.

### 3.5.3. Delete Secrets & ConfigMaps

In addition you may want to remove remaining created Secrets and ConfigMaps.


## 3.6. Upgrades

To update a release, instad of uninstalling and re-installing it with a new configuration,
it can be updated in place.
```
helm upgrade --install [...]
```
For more information on the command's syntax, please refer to the [Helm documentation](https://helm.sh/docs/).



## 3.7. Installation on OpenShift

The following steps show the installation procedure on an RedHat OpenShift Cluster on IBM
Cloud ([ROKS](https://www.openshift.com/products/openshift-ibm-cloud)).

__NOTICE:__ In the following, we assume basic knowledge of using the commands `docker`, `ibmcloud` and `oc` on the command line.

### 3.7.1. Push Images to IBM Container Registry

When using RedHat OpenShift on IBM Cloud, the SM container images can be provided to the
cluster through the [IBM Container Registry](https://cloud.ibm.com/registry/catalog).

Here we only show a quick list of steps how to push the SM images into the IBM Container
registry. For a deeper and better understanding, we recommend reading the
[Container Registry Product Guide](https://cloud.ibm.com/media/docs/pdf/Registry/Registry.pdf).

1) Setup access to the registry by following the related [quickstart documentation](https://cloud.ibm.com/registry/start) (login required).

2) Import compressed (tgz / tar.gz) or uncompressed (tar) images:
```
docker load --input smserver.tgz
docker load --input smagent.tgz
```

3) Tag images for remote registry:
```
docker tag smserver:5.7.0.0-001 de.icr.io/<your-namespace>/smserver:5.7.0.0-001
docker tag smagent:5.7.0.0-001 de.icr.io/<your-namespace>/smagent:5.7.0.0-001
```

4) Login to IBM Cloud:
```
ibmcloud login -a https://cloud.ibm.com -r <your-region> -c <your-account-id> -g <your-resource-group>
```

5) Login to the Container Registry:
__NOTICE:__ If you are using 'podman' instead of 'docker', replace `docker` with `podman` when logging in.
```
ibmcloud cr login --client docker
```

6) Push the tagged images:
```
docker push <your-region-prefix>.icr.io/cenit/smserver:5.7.0.0-001
docker push <your-region-prefix>.icr.io/cenit/smagent:5.7.0.0-001
```
Replace `<your-region-prefix>` with the correct value of your region, eg. `uk` or `de`.

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

The `default` project already contains an Image Pull Secret to allow access to the
Container Registry. For customly created projects this is [not the
case](https://cloud.ibm.com/docs/openshift?topic=openshift-registry#other_registry_accounts).

Copy the existing secret to the custom project/namespace (in this example we are using
namespace/project `sm-test`):
```
oc get secret all-icr-io -n default -o yaml | sed 's/default/sm-test/g' | oc create -n sm-test -f -
```

### 3.7.5. Configure Chart

To deploy SM on OpenShift instead of Kubernetes, some adjustments are required to the configuration values:

* `platform` must be set to `ocp` to allow OCP specific setting for the route.
* `server.image.repository` needs to point to the IBM Container Registry.
* `agent.image.repository` needs to point to the IBM Container Registry.
* `imagePullSecrets` has to include a pull secret working for the IBM Container Registry (i.e. `all-icr-io`)
* `server.persistence.storageClass` has to be set to a valid Storage Class providing block storage (i.e. `ibmc-block-bronze` - see [official reference](https://cloud.ibm.com/docs/containers?topic=containers-block_storage) for storage classes on IBM Cloud).
* `server.persistence.size` has to be set to a valid minimum size for the selected Storage Class (i.e. for `ibmc-block-bronze` the minimum is `20Gi`).
* `server.route.*` has to be configured (see below example). This is why we need to set platform to `ocp`.

The final values file should look like this (`roks.yaml`):
```
platform: ocp

imagePullSecrets:
  - name: all-icr-io

server:
  image:
    repository: someregistry.example.com/sm/smserver
    tag: 5.7.0.0-001

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
    tag: 5.7.0.0-001

  serviceAccountName: sm-svc-acc
```

### 3.7.6. Install & Verify

Install using Helm:
```
helm install mysm cenit/sm -f roks.yaml
```

With this configuration the SM UI will be accessible with a URL similar to this: `https://<release-name>-smserver-<namespace/project>.<cluster-domain>`.


# 4. Configuration Reference

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
| `server.copyDataAccessJar` | Loads the file 'dataaccess.jar' required for connecting to some Db2 instances. If Db2 is used as an external database, this is enabled by default. | `false` |
| `server.resourceDownload.enabled` | If enabled, the files in list resourceDownload.files will be downloaded and placed into '/opt/sm/server/karaf/deploy'. | `false` |
| `server.resourceDownload.files` | A list of files to be downloaded and provided to the container. If a file already exists, it will not be downloaded. | `["https://repo1.maven.org/maven2/com/ibm/db2/jcc/11.5.5.0/jcc-11.5.5.0.jar", "https://repo1.maven.org/maven2/com/microsoft/sqlserver/mssql-jdbc/9.2.1.jre8/mssql-jdbc-9.2.1.jre8.jar"]` |
| `server.log4jFormatMsgNoLookups` | This should be true by default to avoid exploitation of CVE-2021-44228. System Monitor itself does use a log4j2 version that already includes fixes for CVE-2021-44228. | `true` |
| `server.image.repository` | Image repository, e.g. 'de.icr.io' | `""` |
| `server.image.tag` | Image tag, e.g. '5.7.0.0-001' | `""` |
| `server.image.pullPolicy` |  | `"IfNotPresent"` |
| `server.service.type` |  | `"ClusterIP"` |
| `server.service.httpPort` |  | `8080` |
| `server.service.httpsPort` |  | `8443` |
| `server.service.mqttPort` |  | `1883` |
| `server.service.mqttsPort` |  | `8883` |
| `server.mqttLoadBalancer.enabled` |  | `false` |
| `server.mqttLoadBalancer.loadBalancerIP` |  | `""` |
| `server.mqttLoadBalancer.ports` |  | `[{"name": "mqtt", "protocol": "TCP", "port": 1883, "targetPort": 1883}, {"name": "mqtts", "protocol": "TCP", "port": 8883, "targetPort": 8883}]` |
| `server.serviceAccountName` | Name of exisiting service account. If unset, the namespace's default service account is used, which is normally fine. | `""` |
| `server.logLevel` | For supported Log4J log levels see  https://logging.apache.org/log4j/2.x/manual/customloglevels.html. Intensive logging (e.g. "DEBUG" or "TRACE") should only be used temporarily for trouble shooting. In production, "ERROR" should be fine. | `"ERROR"` |
| `server.setPasswordJobEnabled` | Optionally overwrite the default/existing password for the standard admin user at deployment time. If enabled, a job will automatically set a password for user 'admin' as defined in 'adminPasswordSecret'. Of course the password can also be set/changed via the server's web-UI. | `false` |
| `server.adminPasswordSecret` | Existing secret containing the password for user 'admin' (key 'password' must contain password string). | `""` |
| `server.terminationGracePeriodSeconds` | Waiting period before a terminating pod is forcefully ended. To avoid data loss, we recommend using a value greater or equal to 30 seconds. | `30` |
| `server.resources.requests.cpu` |  | `"500m"` |
| `server.resources.requests.memory` |  | `"1536Mi"` |
| `server.readinessProbe.initialDelaySeconds` |  | `180` |
| `server.readinessProbe.periodSeconds` |  | `30` |
| `server.readinessProbe.failureThreshold` |  | `9` |
| `server.readinessProbe.successThreshold` |  | `1` |
| `server.readinessProbe.timeoutSeconds` |  | `5` |
| `server.livenessProbe.initialDelaySeconds` |  | `360` |
| `server.livenessProbe.periodSeconds` |  | `10` |
| `server.livenessProbe.failureThreshold` |  | `3` |
| `server.livenessProbe.successThreshold` |  | `1` |
| `server.livenessProbe.timeoutSeconds` |  | `3` |
| `server.nodeSelector` |  | `{}` |
| `server.tolerations` |  | `[]` |
| `server.affinity` |  | `{}` |
| `server.podAnnotations` |  | `{}` |
| `server.podSecurityContext` | This defines the security context for the pod. The server's pod contains only one container. So normally the 'securityContext' is sufficient to be used, see below. If needed, you can e.g. set a GID for 'fsGroup' here. | `{}` |
| `server.securityContext.runAsNonRoot` | Container security context, see also 'podSecurityContext'. The server normally never needs to run as root. | `true` |
| `server.securityContext.allowPrivilegeEscalation` | Container security context, see also 'podSecurityContext'. Privilege escalation should never be needed. | `false` |
| `server.securityContext.priviledged` | Container security context, see also 'podSecurityContext'. The server needs no privileged execution rights. | `false` |
| `server.securityContext.capabilities.drop` | Container security context, see also 'podSecurityContext'. Dropping all linux container capabilities is strongly recommended. | ["ALL"] |
| `server.securityContext.runAsGroup` | Container security context, see also 'podSecurityContext'. Not needed in OpenShift with SCCs enabled. In Kubernetes you should use '1001'. GID of 'smgrp' in the container is '1001'. | `1001` |
| `server.ingress.enabled` | With an ingress, the server can be accessed from outside of the cluster. Disabled per default. In OpenShift, we recommend to use a 'route', see below. One of these should be used to expose the server's web UI. | `false` |
| `server.ingress.annotations` |  | `{}` |
| `server.ingress.hosts` |  | `[{"host": "*", "paths": ["/"]}]` |
| `server.ingress.tls` |  | `[]` |
| `server.ingress.useHttpsEndpoint` | Will connect to the secure (HTTPS) endpoint on the Server pod if set to 'true', which is the default. 'false' will also disable the HTTPS endpoint of the server container. | `true` |
| `server.ingress.ingressClassName` | Set correct Ingress class if default class 'nginx' is not valid for the present cluster. | `"nginx"` |
| `server.route.enabled` | In OpenShift, the server can be accessed from outside of the cluster. Disabled per default. Recommended in OpenShift instead of ingress. | `false` |
| `server.route.tls` |  | `{}` |
| `server.route.useHttpsEndpoint` | Will connect to the secure (HTTPS) endpoint on the Server pod if set to 'true', which is the default. 'false' will also disable the HTTPS endpoint of the server container. | `true` |
| `server.persistence.existingClaim` | If 'existingClaim' is defined, a PVC must be created manually before the volume can be bound | `""` |
| `server.persistence.enabled` |  | `false` |
| `server.persistence.storageClass` |  | `""` |
| `server.persistence.accessModes` |  | `["ReadWriteOnce"]` |
| `server.persistence.size` |  | `"8Gi"` |
| `server.persistence.annotations` |  | `{}` |
| `server.persistence.volumePermissions.enabled` | Enabling 'volumePermissions' may lead to problems in OpenShift due to SCCs. That is why it is disabled per default. | `false` |
| `server.caCerts` | A list of Secrets containing a TLS certificates which should be imported into SM Server's cacerts file. For each element in `caCerts` the name defined by `secretName` must be unique. If you want to use multiple keys from the same Secret, mention them all in list `secretKeys`. | `[]` |
| `server.extraVolumes` |  | `[]` |
| `server.extraVolumeMounts` |  | `[]` |
| `server.extraInitContainers` |  | `[]` |
| `server.extraEnv` |  | `[]` |
| `server.internalConfigurationDatabase` | See comment in values.yaml for details. | `null` |
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
| `server.externalDependenciesVolume.persistence.existingClaim` | Persistence for an external data volume. If 'existingClaim' is defined, a PVC must be created manually before the volume will be bound. | `""` |
| `server.externalDependenciesVolume.persistence.enabled` |  | `false` |
| `server.externalDependenciesVolume.persistence.storageClass` |  | `""` |
| `server.externalDependenciesVolume.persistence.accessModes` |  | `["ReadWriteOnce"]` |
| `server.externalDependenciesVolume.persistence.size` |  | `"1Gi"` |
| `server.externalDependenciesVolume.persistence.annotations` |  | `{}` |
| `agent.enabled` | Creates a Deployment (dpl) for one SM Agent Pod. Enabled by default. The minimum useful set for monitoring is one server and one agent. | `true` |
| `agent.runPrivilegedContainer` | Required for SM version =< 5.5.5.1-000 if Pod Security Policies do not allow root containers. | `false` |
| `agent.copyDataAccessJar` | Loads the file 'dataaccess.jar' required for connecting to some Db2 instances. If DB2 is used as an external database, this is enabled by default. | `false` |
| `agent.resourceDownload.enabled` | If enabled, the files in list resourceDownload.files will be downloaded and placed into '/opt/sm/agent/karaf/deploy'. | `false` |
| `agent.resourceDownload.files` | A list of files to be downloaded and provided to the container. If a file already exists, it will not be downloaded. | `["https://repo1.maven.org/maven2/com/ibm/db2/jcc/11.5.5.0/jcc-11.5.5.0.jar", "https://repo1.maven.org/maven2/com/microsoft/sqlserver/mssql-jdbc/9.2.1.jre8/mssql-jdbc-9.2.1.jre8.jar"]` |
| `agent.log4jFormatMsgNoLookups` | This should be true by default to avoid exploitation of CVE-2021-44228 in general. System Monitor itself does use a log4j2 version that already includes the fixes for CVE-2021-44228. | `true` |
| `agent.image.repository` | Image repository, e.g. 'de.ic.io/...' | `""` |
| `agent.image.tag` | Image tag, i.e. '5.5.5.1-001' | `""` |
| `agent.image.pullPolicy` |  | `"IfNotPresent"` |
| `agent.serviceAccountName` | Name of exisiting service account. If unset, the namespace's default service account is used, which is normally fine. | `""` |
| `agent.logLevel` | For supported Log4J log levels see: https://logging.apache.org/log4j/2.x/manual/customloglevels.html. Intensive logging (e.g. 'DEBUG' or 'TRACE') should be used only temporarily for trouble shooting. In production 'ERROR' is fine. | `"ERROR"` |
| `agent.terminationGracePeriodSeconds` | Waiting period in seconds before a terminating pod is forcefully ended. To avoid dataloss, we recommend using a value greater or equal to 30 seconds. | `30` |
| `agent.resources.requests.cpu` |  | `"500m"` |
| `agent.resources.requests.memory` |  | `"1536Mi"` |
| `agent.readinessProbe.initialDelaySeconds` | Should not be lower than 180. | `180` |
| `agent.readinessProbe.periodSeconds` | Should not be lower than 30. | `30` |
| `agent.readinessProbe.failureThreshold` |  | `9` |
| `agent.readinessProbe.successThreshold` |  | `1` |
| `agent.readinessProbe.timeoutSeconds` | Should not be lower than 5. | `5` |
| `agent.livenessProbe.initialDelaySeconds` | Should not be lower than 360. | `360` |
| `agent.livenessProbe.periodSeconds` | Should not be lower than 10. | `10` |
| `agent.livenessProbe.failureThreshold` |  | `3` |
| `agent.livenessProbe.successThreshold` |  | `1` |
| `agent.livenessProbe.timeoutSeconds` | Should not be lower than 3. | `3` |
| `agent.nodeSelector` |  | `{}` |
| `agent.tolerations` |  | `[]` |
| `agent.affinity` |  | `{}` |
| `agent.podAnnotations` |  | `{}` |
| `agent.podSecurityContext` | This defines the security context for the pod. The agent's pod contains only one container. So normally the 'securityContext' is sufficient to be used, see below. If needed, you can e.g. set a GID for 'fsGroup' here. | `null` |
| `agent.securityContext.runAsNonRoot` | Container security context, see also 'podSecurityContext'. The agent normally never needs to run as root. | `true` |
| `agent.securityContext.allowPrivilegeEscalation` | Container security context, see also 'podSecurityContext'. Privilege escalation should never be needed. | `false` |
| `agent.securityContext.priviledged` | Container security context, see also 'podSecurityContext'. The agent needs no privileged execution rights. | `false` |
| `agent.securityContext.capabilities.drop` | Container security context, see also 'podSecurityContext'. Dropping all container capabilities is strongly recommended. | ["ALL"] |
| `agent.securityContext.runAsGroup` | Container security context, see also 'podSecurityContext'. Not needed in OpenShift with SCCs enabled. In Kubernetes you should use '1001'. GID of 'smgrp' in the container is '1001'. | `1001` |
| `agent.logMonitoringClaims` | Use this to mount e.g. CPE or ICN logs into the agent for logfile monitoring; see 'values.yaml' for examples. The volumes will be mounted below '/opt/sm/agent/karaf/log-volumes/<name>'. | `[]` |
| `agent.extraVolumes` | Here you can provide syntactically complete entries for volume mounts; see 'values.yaml' for examples. | `[]` |
| `agent.extraVolumeMounts` | Here you can provide syntactically complete volume mounts; for examples see 'values.yaml'. | `[]` |
| `agent.extraInitContainers` | Extra initializations via init-containers should be added here; for an example see 'values.yaml'. | `[]` |
| `agent.extraEnv` | If needed, extra environment variable settings go here. | `[]` |
| `agent.externalDependenciesVolume.persistence.existingClaim` | Persistence for the 'externalDependenciesVolume'. Agents use Kubernetes Deployments which do not support automated provisioning of volumes through 'volumeClaimTemplates'. External dependencies are e.g. P8 API JARs or JDBC driver JARs. | `""` |
| `agent.externalDependenciesVolume.persistence.enabled` | Disabled per default. | `false` |
| `db2TestInstance.enabled` | Installs a single replica Db2 container with default settings. IMPORTANT: It is not intended for production use, but only for testing and training. | `false` |
| `db2TestInstance.runPrivilegedContainer` | The Db2 container only runs if executed in privileged mode. Therefore 'true' by default. | `true` |
| `db2TestInstance.serviceAccountName` | Name of exisiting service account. If unset, the namespace's default service account is used. | `""` |
| `db2TestInstance.image.repository` |  | `"docker.io/ibmcom/db2"` |
| `db2TestInstance.image.tag` |  | `"11.5.5.0"` |
| `db2TestInstance.image.pullPolicy` |  | `"IfNotPresent"` |
| `db2TestInstance.persistence.existingClaim` | Persistence for the Db2 data volume. If it is defined, a PVC must be created manually before the volume will be bound. | `""` |
| `db2TestInstance.persistence.enabled` |  | `false` |
| `db2TestInstance.persistence.storageClass` |  | `""` |
| `db2TestInstance.persistence.accessModes` |  | `["ReadWriteOnce"]` |
| `db2TestInstance.persistence.size` |  | `"8Gi"` |
| `db2TestInstance.persistence.annotations` |  | `{}` |
| `db2TestInstance.resources` | No requests or limits set by default. | `{}` |
| `db2TestInstance.securityContext` | Security context used by default. | `{}` |
| `sqlServerTestInstance.enabled` | Installs a single replica SQL Server container with default settings. IMPORTANT: It is not intended for production use, but only for testing and training. | `false` |
| `sqlServerTestInstance.serviceAccountName` | Name of exisiting service account. If unset, the namespace's default service account is used. | `""` |
| `sqlServerTestInstance.image.repository` |  | `"mcr.microsoft.com/mssql/server"` |
| `sqlServerTestInstance.image.tag` | Get a list of available tags from: https://mcr.microsoft.com/v2/mssql/server/tags/list | `"2019-latest"` |
| `sqlServerTestInstance.image.pullPolicy` |  | `"IfNotPresent"` |
| `sqlServerTestInstance.persistence.existingClaim` | Persistence for the SQL Server data volume. If it is defined, a PVC must be created manually before the volume will be bound. | `""` |
| `sqlServerTestInstance.persistence.enabled` |  | `false` |
| `sqlServerTestInstance.persistence.storageClass` |  | `""` |
| `sqlServerTestInstance.persistence.accessModes` |  | `["ReadWriteOnce"]` |
| `sqlServerTestInstance.persistence.size` |  | `"8Gi"` |
| `sqlServerTestInstance.persistence.annotations` |  | `{}` |
| `sqlServerTestInstance.resources` | No requests or limits set by default. | `{}` |
| `sqlServerTestInstance.securityContext` | Default security context used by default. | `{}` |


# 5. Configuration Examples

## 5.1. SM Server Persistence

### 5.1.1. Via H2 Database on a Persistent Volume

SM Server manages the monitoring and the configuration database in local H2 databases by default. SM Server can use a Persistent
Volume Claim (PVC) to persist data such as configuration data and historical event data. If persistence is disabled, data will be
written on the local node on a volume of type [`emptyDir`](https://kubernetes.io/docs/concepts/storage/volumes/#emptydir) to allow
the data to survive Pod restarts. However, with `emptyDir` all data will vanish if a Pod is removed.

In order to persist both of these databases onto a Persistent Volume Claim, use the `server.persistence` configuration of the
chart. Example:
```
[...]
server:
[...]
  persistence:
    enabled: true
    storageClass: "valid-storage-class"
    accessModes:
      - ReadWriteOnce
    size: 16Gi
[...]
```

> __IMPORTANT — Read carefully to avoid data loss:__ The SM Server uses StatefulSets with VolumeClaimtemplates to provision volumes.
It is recommended to configure the StorageClass with a `retainPolicy` of `Retain`. This way the PersistentVolume storing the
configuration and monitoring H2 databases is retained in case the PersistentVolumeClaim is deleted.

It is also possible to use an existing volume created beforehand. This useful if the required StorageClass does not allow
auto-provisioning through Volume Claim Templates. Example:
```
[...]
server:
[...]
  persistence:
    existingClaim: "my-pvc-123"
[...]
```

__Notice on High Availability:__ Using a Persistent Volume with StorageClass `hostPath` or a similar StorageClass will tie the SM
Server Pod to the node it is scheduled to for the first time after its installation. If this node is removed from the cluster or if
the node goes down, the SM Server Pod cannot be scheduled to another node, because its persisted data still sits on the unreachable
node. To make your setup resilient against this failure situation you can:

* use StorageClasses providing automated replication
* use StorageClasses backed by a central storage system (like a Network File Systems)
* use an external database for persistence (see [guidance in this document](#via-external-database-services))


### 5.1.2. Via External Database Services

Instead of persisting data on mounted volumes, an external database service can be used. Two separate databases are used for data
persistence whereat each of them is configured individually.

* Configuration Database
* Monitoring Database

Create secrets containing user and password for database access. Below examples make use a Db2 test instance (see
[dedicated chapter](#install-a-db2-database-for-testing-scenarios)):
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

If your Kubernetes or OpenShift cluster has internet access, you can have the SM Server's Init Container download required files
before the server starts. The download can take place onto a Persistent Volume, so that the download ideally only happens on the
first start of the SM Server container. The volume is mounted on the path `/opt/sm/server/karaf/deploy`. If files like JDBC drivers
reside in this directory, they are automatically loaded on start of SM Server. If the download procedure is enabled, the chart will
by default download required JDBC drivers for SQL Server and Db2. 

Modify your chart configuration like this to enable the automated download:
```
[...]
server:
  resourceDownload:
    enabled: true
    files:
      - https://<url-to-some-required-file>
      - https://<url-to-some-other-required-file>
  externalDependenciesVolume:
    persistence: enabled
[...]
agent:
  resourceDownload:
    enabled: true
    files:
      - https://<url-to-some-required-file>
      - https://<url-to-some-other-required-file>
  externalDependenciesVolume:
    persistence: enabled
[...]
```
If `server.resourceDownload.files` or `agent.resourceDownload.files` are left unspecified, a set of default libraries will be
downloaded. Refer to the Configuration Reference for more details.

### 5.2.2. Option: Extended SM Container Images

Derive a new image based on the existing standard SM container images by adding additional files. Keep in mind your new image will
have one or more layers added on top of the ones of the original image.

Create a Dockerfile similar to the following example:
```
# Define the required base image
FROM smserver:5.7.0.0-001

# Make sure to perform further actions with the correct user id
USER smusr

# Copy the local JDBC driver file into the correct directory of the SM Server container
COPY ./db2jcc4.jar /opt/sm/server/karaf/deploy/db2jcc4.jar
```

Run the build:
```
docker build -t smserver:5.7.0.0-001-jdbc .
```

### 5.2.3. Option: Shared Volume

If your cluster supports Persistent Volumes of type `ReadWriteMany` (`RWX`) you can provide all required dependencies through a
single volume.

As a first step we need to create a `RWX` volume. Adjust `storageClassName` in following manifest and create it as a file named
`rwx_pvc.yaml`:
```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: sm-shared-pvc
spec:
  accessModes:
    - ReadWriteMany
  # Use a 'storageClassName' available on the cluster
  storageClassName: "my-rwx-volume"
  resources:
    requests:
      storage: 100Mi
```

Apply the manifest:
```
kubectl apply -f rwx_pvc.yaml
```

Create an auxiliary Pod to be able to copy files to the volume in a file named `aux_pod.yaml`. Replace the `image` with a valid SM
Server container image:
```
apiVersion: v1
kind: Pod
metadata:
  name: sm-shared-pvc-aux
spec:
  terminationGracePeriodSeconds: 0
  volumes:
    - name: sm-shared-pvc
      persistentVolumeClaim:
        claimName: sm-shared-pvc
  containers:
    - name: sm-shared-pvc-aux
      # Set 'image' to a container image available to the cluster
      image: smserver:<some-version>
      resources:
        limits:
          cpu: "100m"
          memory: "100Mi"
      volumeMounts:
        - name: sm-shared-pvc
          mountPath: /deploy
      command: ['/bin/bash', '-c', 'while true; do sleep inf; done']
```

Apply the manifest:
```
kubectl apply -f aux_pod.yaml
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

* Set `server.externalDependenciesVolume.persistence.enabled` to `true`
* Set `server.externalDependenciesVolume.persistence.claimName` to `sm-shared-pvc`

For the SM Agent:

* Set `agent.externalDependenciesVolume.persistence.enabled` to `true`
* Set `agent.externalDependenciesVolume.persistence.claimName` to `sm-shared-pvc`

Run the installation using `helm install` as explained in previous sections using this configuration to mount the pre-equipped
volume.

## 5.3. Restricted Pod Permissions

We do not recommend to use dedicated RBAC permissions for the Pods anymore if not really needed. Most of the time SCCs in OpenShift,
the PSA profile in Kubernetes, the limitations of the default service account plus meaningful settings made in the agent's and
server's 'securityContext' and 'podSecurityContext' should be enough. In contrary complex RBAC settings made wrong can diminish the
security of your Pods and the cluster in general.

Keep in mind: SM Server and Agent normally do not need special priviledged containers!

### 5.3.1. Option A: RBAC Resources Created by the Chart

Set the charts's property `createRbacResources` to `true`. This will create the required RBAC resources for you automatically on
installation. __Attention:__ This option requires that the user you are operating with (i.e. run `helm` and `kubectl` commands) is a
`cluster-admin`.

```
[...]
createRbacResources: true
[...]
```

### 5.3.2. Option B: RBAC Resources Created by the User

If the user id you are using to work with the cluster does not have the role `cluster-admin` bound, ask the Cluster's Administrator
to create the required RBAC resources for you. Examples for the required RBAC resources can be found in `sm/examples/rbac`.

Among the resources to be created is a `ServiceAccount`. As soon as the RBAC resources are available in the namespace dedicated for
the installation, add the respective `ServiceAccount` to the Chart settings:
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

The following PodSecurityPolicy has the minimum set of permissions defined required for the SM containers to work. Use this policy
if the cluster has special policies in place that prevent SM from being deployed.

The following may not be working in OpenShift with SCCs or Kubernetes with a different PSA profile active. Especially the range for
UIDs and GIDs may not be needed in that case as an SCC or PSA enforces the numbers allowed. If so, remove these entries from
below policy. This may also apply to 'runAsUser', 'supplementalGroups', 'fsGroup'.

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

## 5.4. Provide Root Certificates to SM Server

In some scenarios it can be required to provide certificates to the SM Server. This can be for example domain root certificates or
self-signed certificates which are not signed by any root certificate which is delivered with the JVM of SM Server. In these cases
the specific certificates have to be provided to SM Server on installation. The following example shows adding a sample certificate
([Baltimore CyberTrust Root](https://www.digicert.com/kb/digicert-root-certificates.htm)) to the SM Server's `cacerts` trust store.

### 5.4.1. Provide Certificate as Secret

Download two sample certificates:
```
wget https://cacerts.digicert.com/BaltimoreCyberTrustRoot.crt.pem
wget https://cacerts.digicert.com/CybertrustGlobalRoot.crt
```

Create a secret using the downloaded certificate file:
```
kubectl create secret generic my-root-certs --from-file=baltimore=BaltimoreCyberTrustRoot.crt.pem --from-file=cybertrust=CybertrustGlobalRoot.crt
```

### 5.4.2. Refer to Certificate Secret in Chart Configuration

The chart configuration provides a property `server.caCerts`; see values.yaml
within the chart. It can be set to a list of references to existing secrets,
such as the one we just created:
```
[...]
server:
[...]
  caCerts:
  - alias: my-cybertrust
    secretName: my-root-certs
    secretKeys:
    - baltimore
    - cybertrust
[...]
```
Each list element has to contain the following keys:

* `alias`: The alias name which is to be used for the certificate in the trust store.
* `secretName`: The name of the previously created secret.
* `secretKeys`: A list of keys used to name the certificates in the Secret.

With this configuration an Init Container will take case of importing the listed certificates into the `cacerts` trust store.

To verify if the certificates have been imported, exec the SM Server container:
```
kubectl exec -it mysm-smserver-0 -c smserver -- bash
```

List all available CA Certificates using the following command and password `changeit`:
```
/opt/sm/server/jre/bin/keytool -list -v -keystore /opt/sm/server/jre/lib/security/cacerts | grep -E 'baltimore|cybertrust'
```

In addition, the logs of InitContainer `import-ca-certs` show detailed output in cases where the import of one of the CA
certificates in to the trust store fails.

```
kubectl logs -f mysm-smserver-0 -c import-ca-certs
```

Sample output:
```
Files to import into trust store from '/shared/certs':
total 4
drwxrwxrwt 3 root root  120 Nov 17 12:38 .
drwxr-xr-x 4 root root 4096 Nov 17 12:38 ..
drwxr-xr-x 2 root root   80 Nov 17 12:38 ..2022_11_17_12_38_35.1768412483
lrwxrwxrwx 1 root root   32 Nov 17 12:38 ..data -> ..2022_11_17_12_38_35.1768412483
lrwxrwxrwx 1 root root   16 Nov 17 12:38 baltimore -> ..data/baltimore
lrwxrwxrwx 1 root root   17 Nov 17 12:38 cybertrust -> ..data/cybertrust

Importing '/shared/certs/baltimore' using alias 'baltimore'
Warning: use -cacerts option to access cacerts keystore
Certificate was added to keystore

Importing '/shared/certs/cybertrust' using alias 'cybertrust'
Warning: use -cacerts option to access cacerts keystore
Certificate was added to keystore

Warning:
The input uses the SHA1withRSA signature algorithm which is considered a security risk. This algorithm will be disabled in a future
update.

Copying trust store to shared volume
```


# 6. Troubleshooting

## 6.1. Entering the SM Server Pod

```
kubectl exec -it mysm-smserver-0 bash
```

## 6.2. Client-side Chart Testing

Use flags `--dry-run=client` and `--debug` for testing purposes.

## 6.3. Failing Probes on Startup

If the SM Server Pod does not have enough CPU resources allocated the startup process time may increase to a time period so long,
that the default `readinessProbe` prematurely diagnoses a failed startup and consequently restarts the Pod for another try. In this
case you can:

* Increase the allocated CPU resources in `server.resources.limits.cpu` to accelerate the startup process.
* Increase the values `server.readinessProbe.initialDelaySeconds` and `server.livenessProbe.initialDelaySeconds` to delay the probes and allow a longer startup time.

## 6.4. Job "mysm-smserver-set-password" Fails on Fresh Installations

`helm uninstall` does not remove existing Persistent Volume Claims used by the SM Server. If a SM Server has already been installed
in the cluster with persistence activated at an earlier point in time, its volume is possibly still present. Ensure that after
uninstalling the volume `data-mysm-smserver-0` is also deleted before starting a new installation.


# 7. Advanced

## 7.1. Container Image Preloading

> __DISCLAIMER:__ The following are hints about container image preloading, which is not the recommended way to deploy SM Server and SM
> Agent. The tools and operating systems mentioned here may have changed since the time of writing. Check their official and
> up-to-date documentation before trying anything of the following. It is possible none of the following will work at all.

Clusters for development and testing purposes may not have access to a container image registry providing the SM container images.
In these environments the images can be loaded into the container runtime manually.

### 7.1.1. On Docker Runtime

Docker is a supported runtime on:

* OpenShift 4.x on RHEL 7+; but check the official documentation for your specific RHEL version beforehand.
* Many Kubernetes distributions.

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

CRI-O is the [default container runtime](https://www.redhat.com/en/blog/red-hat-openshift-container-platform-4-now-defaults-cri-o-underlying-container-engine)
for OpenShift 4.x on RHEL 8+.

__NOTICE:__ If CRI-O is running as non-root, `remove` sudo from the commands shown below.


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

You can have the chart deploy and auto-configure either a Db2 or a SQL Server test instance. The database containers can have
persistence enabled, but will run as single instances only.

Check the Configuration Reference for more details (`db2TestInstance`, `sqlServerTestInstance`).

__IMPORTANT:__ Due to license agreements and the nature of the deployment this must not be used in production environment!

## 7.3. Exposing SM Server to External Agents With NGINX Ingress

A service of type `LoadBalancer` is created if `server.mqttLoadBalancer.enabled` is set to `true`; default is `false`.

__When do I need this?__ In some scenarios it may be required to install SM Agents in another Kubernetes cluster than the SM Server
or for example directly on a Linux/AIX/Windows instance. To make it possible for these external agents to connect to an SM Server
within a Kubernetes cluster, additional measures have to be taken. While connecting into the cluster using the HTTP protocol (i.e.
in order to use the SM Web UI) can be covered with an Ingress, SM Agents use the MQTT protocol to communicate with the SM Server. As
MQTT is a TCP-based protocol, standard Ingress resources are not applicable for this use case, but Services of type `LoadBalancer`
are the appropriate approach.

Verify the existence and binding of the new service (replace `mysm` with the name of your release):
`kubectl get service mysm-smserver-mqtt-loadbalancer`
```
NAMESPACE      NAME                                TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)                               AGE
default        mysm-smserver-mqtt-loadbalancer     LoadBalancer   10.43.160.30    172.18.0.2    1883:31791/TCP,8883:30122/TCP         4m29s
```
Depending on the type of Load Balancer the output looks different. If an External IP is displayed, you should be able to connect
with an Agent from outside the Kubernetes cluster.

If you are working with a local development cluster (like Minikube, K3S, K3D, or Microk8s) a simple SM Agent Docker container can be
used to test an external connection:
```
docker run -d --name extagent01 \
    --net=host \
    -e SERVER_HOSTNAME='127.0.0.1' \
    -e SERVER_PORT=1883 \
    -e CLIENT_ID='extagent01' \
    smagent:5.7.0.0-001
```
For connections through a Load Balancer from an external network, use the External IP displayed with
`kubectl get service mysm-smserver-mqtt-loadbalancer`.

## 7.4. Set an Admin Password on Chart Installation (recommended)

To secure the built-in `admin` user, it is recommended to set its password (default value is `admin`) to a secure password. This can
be done after the first login to the UI using the System Administration settings. Please note, that setting a password in the UI is
only persistent in case you have configured [Persistent Storage for the SM Server](#persistent-storage-for-sm-server).

If the password should be set automatically on installation of SM Server, create a secret containing the new password before
installing or upgrading:
```
kubectl create secret generic my-sm-admin-pw --from-literal=password='s3cr3t'
```
To enable the automatic setting of the password:

* refer to this password using key `server.adminPasswordSecret` .
* enable the responsible job by setting `server.setPasswordJobEnabled` to `true`.

These changes can be made either in your custom `values.yaml` file or using Helm's `--set` argument.


## 7.5. Prepare SM for Monitoring of Mounted Logs

Some IBM products store their logs using dedicated `PersistentVolumeClaims`. These claims can be mounted in the SM agent if they are
referenced in `agent.logMonitoringClaims`:
```
agent:
  logMonitoringClaims:
  - name: some-of-our-cpe-logs
    claimName: cpe-fnlogstore
  - name: some-other-navigator-logs
    claimName: icn-logstore
```
__NOTICE:__ To make all all logs readable to the SM Agent container, it may be required to set appropriate parameters for `agent.securityContext`.

