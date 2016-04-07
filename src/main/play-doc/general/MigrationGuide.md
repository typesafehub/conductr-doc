# ConductR Migration Guide


This document describes what is required to move between version 1.1 to 1.2. Here is a list of major areas to be considered when migrating:

* [Proxying](#Roles_and_toplogy)
* [Proxying](#Proxying)

## Roles and topology

ConductR now requires roles to be specified for each node where ConductR bundles run. Prior to installing ConductR consider a topology that has roles representing a DMZ, gateway services, persistance services (databases) and so forth. You can disable role checking by specifying the following in `application.ini`: 

```
-Dconductr.resource-provider.match-offer-roles=off
```

## Proxying

conductr-haproxy is now provided as a bundle with the "haproxy" role. When running conductr-haproxy you should specify a scale for the number of HAProxy services in your cluster. In addition the nodes that run the HAProxy service must have the "haproxy" role from a ConductR perspective.

### Amazon's ELB

Prior to 1.2, we recommended that you configure the ELB to poll the /bundles endpoint of the control protocol. Given conductr-haproxy you should now configured the ELB to poll the http://:9009/status. This endpoint will return an HTTP OK status when HAProxy has been correctly configured therefore leading to a more reliable configuration of the ELB. See [the cluster setup considerations document](ClusterSetupConsiderations#Cluster security considerations) for more information.

## Syslog

In additional to disabling Elasticsearch via configuration, a service lookup must now also be disabled. ConductR's default behavior is to locate the service endpoint for logging when required. Supposing that you are using RSYSLOG locally then ConductR's `application.ini` should contain the following:

```
  -Dcontrail.syslog.server.host=127.0.0.1
  -Dcontrail.syslog.server.port=514
  -Dcontrail.syslog.server.elasticsearch.enabled=off 
  -Dcontrail.syslog.server.service-locator.enabled=off
```