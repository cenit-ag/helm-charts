Release Notes for `cenit-ag/helm-charts/sm`
===

<!-- TOC -->

- [v1.1.3](#v113)
- [v1.1.2](#v112)

<!-- /TOC -->

# v1.1.3
- Set environment variable LOG4J_FORMAT_MSG_NO_LOOKUPS for Server and Agent by default to mitigate CVE-2021-44228. this can be configured using `server.log4jFormatMsgNoLookups` and `agent.log4jFormatMsgNoLookups`, but it is strongly recommended to keep the default value of `true`.
- Disable MV Storage Engine for H2 configuration database by default. __IMPORTANT:__ If you want to keep MV Store, set ... to `true`. Otherwise export the Server's configuration (see "Administration" panel in the UI) and re-import the configuration after the upgrade.
- Add parameter `server.ingress.ingressClassName` with a default value `nginx`.

# v1.1.2
- Add `server.caCerts` list to allow import of certificates into `cacerts` trust store of SM Server JVM