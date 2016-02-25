# ConductR release notes


## 1.1.1

1. When a load or a scale fails due to no resources being available then ConductR now reports this. For loading this means that the initial http load request is responded with a `Forbidden` http status`, along with a message. For all other scenarios, an error event is raised on the bundle and `conduct events` can be used to discover what happened.
2. Bundle replication would sometimes fail due to there being insufficient resources. In this scenario CPU, memory and disk space were being checked. However replication only requires a check in terms of disk space, and so this is now the case. Replication should therefore be more reliable in scenarios where there is low CPU or memory availability.

## 1.1.0

Initial release.
