# ConductR release notes

### 1.1.3

1. The Resource Provider now outputs its view of what resources it believes are available so that operators can better diagnose failure to obtain suitable resource offers.

2. The service locator should always remove the first path component when replying with a redirect and only use the first path component to identify the service name in a redirect style scenario, enabling seamless usage with HTTP clients.

3. Get and load bundles endpoint should only respond to `/bundles` or `/bundles/` paths, and should not response to any other subpaths underneath.
Prior this release, calling HTTP get on `/bundles/some/random/path` will return the bundles response. This is incorrect as 404 should be returned instead

4. Introduces Eslite, a basic Elasticsearch behaving bundle that doesn't require a GiB of memory to run. Eslite is ideal for localhost testing and will be used for the default logging option by the developer sandbox.

5. Updates ConductR's use of the Reactive Platform to rp16s01p05.

6. Syslog connectivity was broken. This has now been repaired.

### 1.1.2

1. Preserve the first component of a path that is rewritten at the proxy.

2. Only retrieve bundle info after the events connection is established to prevent potential loss of events.

3. Introduce new conductr-haproxy integration tests.

4. Integrate StateEvents and StateQueries actors to provide a consistent view of bundle state between the two to prevent possible race condition.

## 1.1.1

1. When a load or a scale fails due to no resources being available then ConductR now reports this. For loading this means that the initial http load request is responded with a `Forbidden` http status`, along with a message. For all other scenarios, an error event is raised on the bundle and `conduct events` can be used to discover what happened.
2. Bundle replication would sometimes fail due to there being insufficient resources. In this scenario CPU, memory and disk space were being checked. However replication only requires a check in terms of disk space, and so this is now the case. Replication should therefore be more reliable in scenarios where there is low CPU or memory availability.

## 1.1.0

Initial release.
