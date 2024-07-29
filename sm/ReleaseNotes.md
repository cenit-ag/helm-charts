Release Notes for `cenit-ag/helm-charts/sm`
===

<!-- TOC -->

- [Release Notes for `cenit-ag/helm-charts/sm`](#release-notes-for-cenit-aghelm-chartssm)
- [v1.2.0](#v120)
- [v1.1.8](#v118)
- [v1.1.6](#v116)
- [v1.1.5](#v115)
- [v1.1.4](#v114)
- [v1.1.3](#v113)
- [v1.1.2](#v112)

<!-- /TOC -->

# v1.2.0

- H2 updated from 2.1.x to 2.2.x
  - **Important** If you are using H2 read the Release Notes before updating!
- New operating UI based upon Angular
- New features: Maintenance mode and tag manager. Both only accessible in the new UI.
- User management now also available in the new UI.

# v1.1.8

- Use this version for 5.5.11.0-003 and 5.5.11.0-004.
- IBM Java 11 JRE updated
- Apache Karaf updated
- H2 updated.
- Security fixes
  - These fixes also have an effect on the URL to be used in the readiness and liveness probes.

# v1.1.6

- Fix issue with mounting and importing CA certificates
- Add notice regarding mounted log volumes in documentation

# v1.1.5

- Remove default value of `0` for `agent.podSecurityContext.runAsGroup` to avoid issues on many clusters
- Add warning in documentation regarding the defaut `reclaimPolicy` being set to `Delete` by default for most `StorageClasses`
- __BREAKING CHANGE:__ Fix CA certificate import for multiple cases in which multiple certificates are mentioned from the same Kubernetes secret. Check the default `values.yaml` file of the chart for the new configuration format.

# v1.1.4
- Annotations for IBM License Service added.

# v1.1.3
- Set environment variable LOG4J_FORMAT_MSG_NO_LOOKUPS for Server and Agent by default to mitigate CVE-2021-44228. this can be configured using `server.log4jFormatMsgNoLookups` and `agent.log4jFormatMsgNoLookups`, but it is strongly recommended to keep the default value of `true`.
- Disable MV Storage Engine for H2 configuration database by default. __IMPORTANT:__ If you want to keep MV Store, set ... to `true`. Otherwise export the Server's configuration (see "Administration" panel in the UI) and re-import the configuration after the upgrade.
- Add parameter `server.ingress.ingressClassName` with a default value `nginx`.

# v1.1.2
- Add `server.caCerts` list to allow import of certificates into `cacerts` trust store of SM Server JVM
