# Dynamic proxy configuration

The dynamic proxy configuration feature allows operations to provide a customized proxy configuration that would be updated automatically according to the applications that are running within the ConductR cluster.

ConductR provides a default configuration that can be overridden should the need arise.  For example, HAProxy can provide HTTP Basic Auth for staging deployments enabling protected preview prior to promoting to production. This is accomplished by deploying a template `haproxy.cfg` file as part of the configuration of the HAProxy bundle.

This document is a guide and reference for using dynamic HAProxy configuration templates.


## Assumptions

ConductR uses [HAProxy](http://www.haproxy.org/) as its proxying solution. It is assumed the reader has some familiarity with HAProxy installation and configuration.

ConductR makes use of [Handlebars](http://handlebarsjs.com/) as the templating solution to generate the customised HAProxy configuration. It is assumed the reader has some degree familiarity in working with other templating solutions.

## Requirement

ConductR makes the assumption that an endpoint must be uniquely identified by its name given a particular system and system version.

The endpoints that failed to satisfy this requirement will be considered as duplicate, and the duplication is assumed to be a result of configuration error. Because of this, duplicated endpoints will not be exposed via proxy configuration.

## Creating and deploying custom HAProxy configuration

It is possible to provide custom HAProxy configuration by supplying configuration overrides to the ConductR HAProxy bundle.

[[ConductR CLI|CLI]] is required to perform the following steps, and as such it needs to be installed beforehand.

These are the steps to deploying custom HAProxy configuration.

### Setup

Create a directory where the custom HAProxy configuration will be placed, e.g.

```lang-none
mkdir -p /tmp/custom-haproxy-conf
```

### Populate the custom HAProxy configuration template

Create the custom HAProxy template, e.g.

```lang-none
touch /tmp/custom-haproxy-conf/haproxy-override.cfg
```

The file can be given any name as long as it's within the directory created from the previous step.

In this example, we'll populate the file with the following custom template.

```
# HAProxy Specific Configuration
global
	log /dev/log	local0
	log /dev/log	local1 notice
	chroot /var/lib/haproxy
	stats socket /run/haproxy/admin.sock mode 660 level admin
	stats timeout 30s
	user haproxy
	group haproxy
	daemon

defaults
	log	global
	mode	http
	option	httplog
	option	dontlognull
    timeout connect 5000
    timeout client  50000
    timeout server  50000


# Custom Endpoint Configuration
{{#eachAcls bundles defaultHttpPort=9443}}
  {{#ifAcl 'public-site' '1' 'main'}}
frontend my_www_frontend
  bind {{haproxyHost}}:80
  # Uncomment the following line if SSL is used
  # bind {{haproxyHost}}:443 ssl crt {{overrideConfigDir}}/test-cert.pem
  mode http
  acl my_www_host_match hdr(host) -i www.acme.com
  use_backend my_www_backend if my_www_host_match

backend my_www_backend
  mode http
    {{#eachBackendServer}}
  server {{serverName}} {{host}}:{{port}} maxconn 1024
    {{/eachBackendServer}}
  {{/ifAcl}}
{{/eachAcls}}
```

Here are the explanation for the example template above:

* *HAProxy Specific Configuration* section is declared prior *Custom Endpoint Configuration* section.
* *HAProxy Specific Configuration* section is populated with operating system specific setup. In the example above it is populated with a cut-down version of Ubuntu's configuration.
* *Custom Endpoint Configuration* section contains application specific proxy configuration.
* The endpoint that matches the following criteria has a custom configuration declared within the [ifAcl](#ifAcl) block:
    * `system` is `public-site`
    * `system version` is `1`
    * `endpoint name` is `main`
* HAProxy frontend `my_www_frontend` is configured to listen on HTTP port `80` and will relay request for `www.acme.com` to `my_www_backend`.
* HAProxy backend `my_www_backend` server is generated based on the backend server information contained within the [eachBackendServer](#eachBackendServer) helper function.
* The backend server information will be updated whenever there's changes to the endpoint, e.g. bundle instances being started or stopped the within the ConductR cluster.
* All other endpoints is for configured using ConductR default configuration. The HTTP endpoint is exposed on the port `9443`

### Deploying SSL certificate

This step is optional.

It's possible to deploy SSL cert as part of the configuration override. The SSL cert will enable HTTPS traffic which terminates at the HAProxy.

Continuing from previous example, place the SSL certificate in `/tmp/custom-haproxy-conf`, e.g.

```lang-none
cp /tmp/test-cert.pem /tmp/custom-haproxy-conf
```

In the example above, the SSL cert called `test-cert.pem` is placed in `/tmp/custom-haproxy-conf`.

The SSL cert will need to be referenced in the HAProxy configuration template. Continuing from previous example, uncomment the SSL configuration from the template, e.g.

```
...

  {{#ifAcl 'public-site' '1' 'main'}}
frontend my_www_frontend
  bind {{haproxyHost}}:80
  # Uncomment the following line if SSL is used
  bind {{haproxyHost}}:443 ssl crt {{overrideConfigDir}}/test-cert.pem

...

```


### Declare the custom HAProxy configuration template

Create a file called `runtime-config.sh` within the proxy configuration directory, e.g.

```lang-none
touch /tmp/custom-haproxy-conf/runtime-config.sh
```

Populate the file with the following entry:

```bash
#!/bin/bash

CONFIG_DIR=$( cd $( dirname "${BASH_SOURCE[0]}" ) && pwd )
export CONDUCTR_HAPROXY_CONFIG_OVERRIDE="$CONFIG_DIR/haproxy-override.cfg"
```

The `CONFIG_DIR` refers to the directory where the configuration override will be expanded. `CONDUCTR_HAPROXY_CONFIG_OVERRIDE` is the environment variable which tells ConductR where to find the custom HAProxy configuration template.


### Package the custom HAProxy configuration template

Use the CLI to package the configuration override:

```lang-none
shazar /tmp/custom-haproxy-conf
Created digested ZIP archive at ./custom-haproxy-conf-ffd0dcf76f4d565424a873022fbb39f3025d4239c87d307be3078b320988b052.zip

```

The generated file `custom-haproxy-conf-ffd0dcf76f4d565424a873022fbb39f3025d4239c87d307be3078b320988b052.zip` is the configuration override that can be loaded alongside ConductR HAProxy bundle.


### Load the custom HAProxy configuration template

Once custom configuration override is generated, it can be loaded into ConductR, e.g:

```lang-none
conduct load /tmp/conductr-haproxy-v2-0d24d10cb0d1af9bf9f7e0bf81778a61d2cc001f9393ef035cb343722da3ac87.zip /tmp/custom-haproxy-conf-ffd0dcf76f4d565424a873022fbb39f3025d4239c87d307be3078b320988b052.zip
```

In the example above, the files required are placed within the `/tmp` directory. Replace the `/tmp` with the actual path to the files.

Refer to the set of `conduct` commands described in [[Deploying bundles|DeployingBundlesOps]] for more details.

## Default configuration

ConductR provides the following HAProxy configuration as a default:

```
global
    log /dev/log    local0
    log /dev/log    local1 notice
    chroot /var/lib/haproxy
    stats socket /run/haproxy/admin.sock mode 660 level admin
    stats timeout 30s
    user haproxy
    group haproxy
    daemon

    # Default SSL material locations
    ca-base /etc/ssl/certs
    crt-base /etc/ssl/private

    # Default ciphers to use on SSL-enabled listening sockets.
    # For more information, see ciphers(1SSL). This list is from:
    #  https://hynek.me/articles/hardening-your-web-servers-ssl-ciphers/
    ssl-default-bind-ciphers ECDH+AESGCM:DH+AESGCM:ECDH+AES256:DH+AES256:ECDH+AES128:DH+AES:ECDH+3DES:DH+3DES:RSA+AESGCM:RSA+AES:RSA+3DES:!aNULL:!MD5:!DSS
    ssl-default-bind-options no-sslv3

defaults
    log global
    mode    http
    option  httplog
    option  dontlognull
        timeout connect 5000
        timeout client  50000
        timeout server  50000
    errorfile 400 /etc/haproxy/errors/400.http
    errorfile 403 /etc/haproxy/errors/403.http
    errorfile 408 /etc/haproxy/errors/408.http
    errorfile 500 /etc/haproxy/errors/500.http
    errorfile 502 /etc/haproxy/errors/502.http
    errorfile 503 /etc/haproxy/errors/503.http
    errorfile 504 /etc/haproxy/errors/504.http

{{#eachAcls bundles defaultHttpPort=9000}}

  {{#ifAcl 'conductr-kibana' '1' 'kibana'}}
# ConductR - Kibana Bundle HAProxy Configuration
frontend kibana_frontend
  bind {{haproxyHost}}:5601
  mode http
  acl kibana_context_root path_beg /
  use_backend kibana_backend if kibana_context_root

backend kibana_backend
  mode http
    {{#eachBackendServer}}
  server {{serverName}} {{host}}:{{port}} maxconn 1024
    {{/eachBackendServer}}
  {{/ifAcl}}

  {{#ifAcl 'visualizer' '1.1' 'visualizer'}}
# ConductR - Visualizer Bundle HAProxy Configuration
frontend visualizer_frontend
  bind {{haproxyHost}}:9999
  mode http
  acl visualizer_context_root path_beg /
  use_backend visualizer_backend if visualizer_context_root

backend visualizer_backend
  mode http
    {{#eachBackendServer}}
  server {{serverName}} {{host}}:{{port}} maxconn 1024
    {{/eachBackendServer}}
  {{/ifAcl}}

{{/eachAcls}}


{{#haproxyConf bundles}}
{{serviceFrontends}}
{{#unless (serviceFrontends)}}
frontend dummy
  bind 127.0.0.1:65535
{{/unless}}
{{serviceBackends}}
{{/haproxyConf}}
```

The default configuration above is written in [Handlebars](http://handlebarsjs.com/) template, and thus the template makes use of variables and helper functions made available to the template.

## Dynamic proxy configuration DSL

The variables and helper functions exposed to the template forms the DSL required to generate HAProxy configuration from the information available within the ConductR cluster.

### Variables

The following variables are available to the template:

| Name              | Type   | Description |
|-------------------|--------|-------------|
| haproxyHost       | String | Required: Host address of the proxy of which the configuration will be generated for.<br><br>Example: 10.0.2.223 |
| bundles | JSON array | Required: List of bundles returned by the ConductR bundles endpoint. This list is in the form of an arrays where each element is a bundle represented by a JSON object. The cluster-wide information of what has been deployed and what is currently running is being stored in this variable.<br><br>Example: See Query bundle state |
| overrideConfigDir | String | Optional: Physical directory location on the file system where configuration overrides has been unpacked. This variable is only defined when HAProxy custom template for ConductR HAProxy has been supplied.<br><br>Example: /tmp/0d24d10cb0d1af9bf9f7e0bf81778a61-e357d11e8a38228880aa57d976d6f83b/e357d11e8a38228880aa57d976d6f83b576fd95e21dc99e8e4d02d4b227b4be3<br><br>In the example above the content of the ConductR HAProxy configuration override zip has been unpacked into the directory `/tmp/0d24d10cb0d1af9bf9f7e0bf81778a61-e357d11e8a38228880aa57d976d6f83b/e357d11e8a38228880aa57d976d6f83b576fd95e21dc99e8e4d02d4b227b4be3`. |


### Functions

The following helper functions are available to the template:

* [eachAcls](#eachAcls)
* [ifAcl](#ifAcl)
* [eachBackendServer](#eachBackendServer)
    * [serverName](#serverName)
    * [host](#host)
    * [port](#port)
* [haproxyConf](#haproxyConf)
* [serviceFrontends](#serviceFrontends)
* [serviceBackends](#serviceBackends)


#### eachAcls

Iterates through each request ACL exposed by the endpoints given a list of bundles.

Used in conjunction with [ifAcl](#ifAcl) to provide customised configuration override given endpoint matches a particular `system`, `system version`, and `endpoint name`.

Once iteration of all bundles have been completed, generates default HAProxy configuration for the bundles without configuration override.

##### Input arguments

| Name            | Type       | Description |
|-----------------|------------|-------------|
| bundles         | JSON array | Optional: List of bundles returned by the ConductR endpoint. Refer to [bundles](#bundles). |
| defaultHttpPort | Integer    | Default HTTP port where HTTP-based endpoints will be exposed. This must be specified keyword argument - refer to the example usage below. |

##### Examples

```
{{#eachAcls bundles defaultHttpPort=9000}}
{{/eachAcls}}
```

In the example above, default HAProxy configuration will be generated. Since no overrides has been specified, so all HTTP-based endpoint will be exposed via port `9000`.

#### ifAcl

Given an endpoint has request ACL defined and it matches a particular `system`, `system version`, and `endpoint name`, the content nested within the `ifAcl` block will be rendered.

The `ifAcl` helper can be used as a stand-alone block helper, or nested within [eachAcls](#eachAcls) block.

When called within [eachAcls](#eachAcls) block, and the matching endpoint will be excluded when the default configuration is being generated.

##### Input arguments

| Name          | Type       | Description |
|---------------|------------|-------------|
| bundles       | JSON array | Optional: List of bundles returned by the ConductR endpoint. Refer to [bundles](#bundles). |
| system        | String     | Match criteria for a particular `system`. |
| systemVersion | String     | Match criteria for a particular `system version`. |
| endpointLabel | String     | Match criteria for a particular `endpoint name`. |

##### Examples

_Standalone_

```
frontend http_frontend
  bind {{haproxyHost}}:80
  mode http
{{#ifAcl bundles 'typesafe-website' '1' 'www'}}
  # Forward to www.lightbend.com backend
  acl lightbend_www hdr(host) -i www.lightbend.com
  use_backend lightbend_www_backend if lightbend_www
{{/ifAcl}}
```

In the example above, the `ifAcl` helper looks looks for the endpoint matches the following criteria within the `bundle` input:

* `system` is `typesafe-website`
* `system version` is `1`
* `endpoint name` is `www`

Should an endpoint that matches the above criteria is deployed and running, the content nested within the `ifAcl` block will be generated. In this case, it will perform a header match against `www.lightbend.com` and will forward the request to a backend called `lightbend_www_backend`.

The content above contains additional DSLs such as [haproxyHost](#haproxyHost) which is explained in their respective sections.


_Within [eachAcls](#eachAcls) block_


```
{{#eachAcls bundles defaultHttpPort=9000}}
  {{#ifAcl 'visualizer' '1.1' 'visualizer'}}
# ConductR - Visualizer Bundle HAProxy Configuration
frontend visualizer_frontend
  bind {{haproxyHost}}:9999
  mode http
  acl visualizer_context_root path_beg /
  use_backend visualizer_backend if visualizer_context_root

backend visualizer_backend
  mode http
    {{#eachBackendServer}}
  server {{serverName}} {{host}}:{{port}} maxconn 1024
    {{/eachBackendServer}}
  {{/ifAcl}}
{{/eachAcls}}

```

In the example above, the `ifAcl` helper looks for the endpoint matches the following criteria:
* `system` is `visualizer`
* `system version` is `1`
* `endpoint name` is `visualizer`

Should an endpoint that matches the above criteria is deployed and running, the content nested within the `ifAcl` block will be generated.

The content above contains additional DSLs such as [haproxyHost](#haproxyHost), [eachBackendServer](#eachBackendServer), [serverName](#serverName), [host](#host), and [port](#port) which are explained in their respective sections.

#### eachBackendServer

Iterate through the backend server given a particular endpoint. The content declared within the `eachBackendServer` block will be generated given each backend server, and will be concatenated in the end.

Can only be used within [ifAcl](#ifAcl) block.

##### Input arguments

The `eachBackendServer` exposes the following variables. They are only scoped within `eachBackendServer` block:

| Name       | Type    | Description |
|------------|---------|-------------|
| serverName | String  | A server name unique to a particular backend server. Only visible within [eachBackendServer](#eachBackendServer) block. |
| host       | String  | The host address of a particular backend server. Only visible within [eachBackendServer](#eachBackendServer) block. |
| port       | Integer | The port of a particular backend server. Only visible within [eachBackendServer](#eachBackendServer) block. |

##### Examples

```
{{#ifAcl bundles 'typesafe-website' '1' 'www'}}
# Lightbend website
backend lightbend_www_backend
  mode http
  {{#eachBackendServer}}
  server {{serverName}} {{host}}:{{port}} maxconn 1024
  {{/eachBackendServer}}
{{/ifAcl}}
```

In the example above we are iterating through the backend server for the endpoint matching the following criteria:

* `system` is `typesafe-website`
* `system version` is `1`
* `endpoint name` is `www`

#### haproxyConf

Generates HAProxy configuration given bundle endpoints which are declared with `services` URI (as opposed to request acls).

##### Input arguments

| Name          | Type       | Description |
|---------------|------------|-------------|
| bundles       | JSON array | Optional: List of bundles returned by the ConductR endpoint. Refer to [bundles](#bundles). |
| haproxyConf   | String     | Works in conjunction with [serviceFrontends](#serviceFrontends) and [serviceBackends](#serviceBackends) below. |

#### serviceFrontends

Generates HAProxy frontend configuration given bundle endpoints which are declared with `services` URI (as opposed to request acls).

#### serviceBackends

Generates HAProxy backend configuration given bundle endpoints which are declared with `services` URI (as opposed to request acls).

##### Examples

The functions [haproxyConf](#haproxyConf), [serviceFrontends](#serviceFrontends), and [serviceBackends](#serviceBackends) are used in conjunction:

```
{{#haproxyConf bundles}}
{{serviceFrontends}}
{{#unless (serviceFrontends)}}
frontend dummy
  bind 127.0.0.1:65535
{{/unless}}
{{serviceBackends}}
{{/haproxyConf}}
```

In the example above HAProxy configuration will be generated for endpoints declared with `services` URI given the `bundles` input. If there are no `serviceFrontends` to be generated, the frontend called `dummy` will be rendered instead.

## Example Use Cases

Here are some common use cases for the use of customized HAProxy configuration template.

### Collecting multiple backends under a single frontend

An example of this use case would be:
* Collect all public facing HTTP port 80/443 endpoints under the same frontend.
* All other endpoints are exposed via separate port exposed only to the internal network.

```
# Catch-all HTTP frontend coming at port 80
frontend http_frontend
  bind {{haproxyHost}}:80
  mode http
{{#ifAcl bundles 'typesafe-website' '1' 'www'}}
  # Forward to www.lightbend.com backend
  acl lightbend_www hdr(host) -i www.lightbend.com
  use_backend lightbend_www_backend if lightbend_www
{{/ifAcl}}
{{#ifAcl bundles 'reactive-summit' '1' 'web'}}
  # Forward to www.reactivesummit.org
  acl reactive_summit_www hdr(host) -i www.reactivesummit.org
  use_backend reactive_summit_www_backend if reactive_summit_www
{{/ifAcl}}

{{#eachAcls bundles defaultHttpPort=5443}}
  {{#ifAcl 'typesafe-website' '1' 'www'}}
# Lightbend website
backend lightbend_www_backend
  mode http
    {{#eachBackendServer}}
  server {{serverName}} {{host}}:{{port}} maxconn 1024
    {{/eachBackendServer}}
  {{/ifAcl}}
	{{#ifAcl 'reactive-summit' '1' 'web'}}
# Reactive Summit website
backend reactive_summit_www_backend
  mode http
    {{#eachBackendServer}}
  server {{serverName}} {{host}}:{{port}} maxconn 1024
    {{/eachBackendServer}}
  {{/ifAcl}}
{{/eachAcls}}
```

Here are the explanation for the example template above:
* The first two [ifAcl](#ifAcl) instructions collects the `typesafe-website` and `reactive-summit` endpoints under same HTTP frontend port `80`.
* The [eachAcls](#eachAcls) instruction exposes the remaining endpoints to port `5443` which is declared as the default HTTP port.
* The [ifAcl](#ifAcl) instructions nested within the [eachAcls](#eachAcls) will render the backend for `typesafe-website` and `reactive-summit` endpoints respectively.

### HTTP Basic Auth

Here's an example of enabling HTTP Basic Auth.

```
{{#eachAcls bundles defaultHttpPort=5443}}
  {{#ifAcl 'conductr-kibana' '1' 'kibana'}}
# ConductR - Kibana Bundle HAProxy Configuration
userlist kibana_users
  user kibana insecure-password k1b4n4

frontend kibana_frontend
  bind {{haproxyHost}}:5601
  mode http
  acl kibana_basic_auth http_auth(kibana_users)
  http-request allow if kibana_basic_auth
  http-request auth realm kibana_users if !kibana_basic_auth
  acl kibana_context_root path_beg /
  use_backend kibana_backend if kibana_context_root

backend kibana_backend
  mode http
    {{#eachBackendServer}}
  server {{serverName}} {{host}}:{{port}} maxconn 1024
    {{/eachBackendServer}}
  {{/ifAcl}}
{{/eachAcls}}
```

Here are the explanation for the example template above:
* The endpoint that matches the following criteria has a custom configuration declared within the [ifAcl](#ifAcl) block:
  * `system` is `conductr-kibana`
  * `system version` is `1`
  * `endpoint name` is `kibana`
* The `userlist` provides list of user groups and `user` directive declares each user within the group.
* The `kibana_basic_auth` acl checks if the HTTP basic auth is supplied with the request
* The `http-request auth realm kibana_users if !kibana_basic_auth` will trigger the HTTP basic auth prompt if the HTTP basic auth is not supplied with the request.
* The example has list of users declared with insecure password. In the actual production scenario, consider using a `password` HAProxy directive which declares a secure password - refer to the [HAProxy](http://www.haproxy.org/) documentation for further details.

## Troubleshooting

Here are the common troubleshooting steps when there's a problem exposing bundle endpoints.

### Check if bundle is running

The proxy will only be configured for endpoints from the bundles which are running. Use the command `conduct info` to ensure that your bundle is in the running state.

### Ensure ACL is exposed

Review the ACL configuration of the endpoints deployed in the ConductR cluster. Use the command `conduct acls http` and `conduct acls tcp` to display request ACLs for HTTP and TCP traffic respectively.

The command will provide a warning message if duplicate endpoints are detected. Refer to [Requirement](#Requirement) section for more details on duplicate endpoints.

### Check the logs from ConductR HAProxy bundle

Check the logs from ConductR HAProxy bundle for possible errors.

If there is a problem with the supplied custom HAProxy configuration template, they will be logged as part of ConductR HAProxy bundle.

If consolidated logging with Elasticsearch is enabled, the command `conduct logs conductr-haproxy` will display logs from the ConductR HAProxy bundle.

Refer to [Logging](Logging) for further details on Elasticsearch bundle installation and setup.

### Check the generated HAProxy configuration

The ConductR HAProxy bundle will generate HAProxy configuration in `/etc/haproxy/haproxy.cfg` - check if the generated HAProxy configuration is correct.