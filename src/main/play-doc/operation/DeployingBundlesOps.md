# Deploying Bundles

The standard bundle lifecycle in the ConductR is:

1. Load. Bundle is loaded into a ConductR cluster and replicated among cluster nodes.
2. Run. Bundle is started by running one or more instances of all defined bundle components.
3. Stop. Bundle is stopped by stopping all instances of all defined bundle components.
4. Unload. Bundle is unloaded from a ConductR cluster and all bundle replicas are removed from cluster nodes.

We use `conduct` command provided by the CLI to load a Visualizer bundle together with configuration to the ConductR and then run it. Every `conduct` command that is communicating with a ConductR needs to be given `--host` and `--port` parameters (these can be omitted if ConductR is running on the same machine and on the default port). Alternatively CLI can use `CONDUCTR_HOST` and `CONDUCTR_PORT` environment variables for corresponding parameters.

This data is used by ConductR when making bundle scheduling decisions. Load the Visualizer bundle by executing:

```bash
conduct load --host 172.17.0.1 visualizer
```

> Substitute `172.17.0.1` with the address of a ConductR core on your cluster. `192.168.10.1` is the default address when using the developer sandbox.

Note that by default, bundles have a maximum size of 1GB. This can be altered via the `akka.http.server.parsing.max-content-length` setting.

Use `conduct info` command to list all loaded bundles. You should see that the Visualizer is replicated but not running (note that the example below shows 3 replications - you'll only get that if you have 3 or more nodes as the bundle cannot replicate beyond the cluster size).

```bash
conduct info --host 172.17.0.1
```
...will yield something like:

```bash
ID       NAME          TAG  #REP  #STR  #RUN  ROLES
6cc7dbc  visualizer  2.0.0     3     0     0  web
```

Run the Visualizer by executing:

```bash
conduct run --host 172.17.0.1 visualizer
```

Whenever you need to refer to a bundle you can use a prefix of or a full bundle id/name. Run `conduct info` once again, to see that the bundle has been successfully started.

```bash
ID       NAME          TAG  #REP  #STR  #RUN  ROLES
6cc7dbc  visualizer  2.0.0     3     0     1  web
```

> Pro tip: if you get bored of typing `--host` you can set the `CONDUCTR_HOST` environment variable instead.

## Using CLI to orchestrate bundle deployments

The CLI commands `conduct load`, `conduct run`, `conduct stop`, and `conduct unload` waits for an expected event to occur. For example `conduct load` will wait for bundle to be installed, and `conduct run` will wait for the number of scale requested to be achieved.

This behaviour makes it possible to use the CLI for orchestration of bundle deployment, e.g.

```bash
#!/bin/bash

set -e

backend_region_bundle_id=$(conduct load -q reactive-maps-backend-region)
backend_summary_bundle_id=$(conduct load -q reactive-maps-backend-summary)
frontend_bundle_id=$(conduct load -q reactive-maps-frontend)

conduct run ${backend_region_bundle_id}
conduct run ${backend_summary_bundle_id} --affinity ${backend_region_bundle_id}
conduct run ${frontend_bundle_id}
```

The script above performs the following:

* Loads all bundles required by [Reactive Maps](http://www.lightbend.com/activator/template/reactive-maps) in the following order: Backend Region, Backend Summary, and Frontend.
* Start the bundles with the following order: Backend Region, Backend Summary, and Frontend.
* The Backend Summary will be run in the same host as Backend Region due to `--affinity` switch.

Note the `-q` switch applied to the `conduct load` command in the example above. The `-q` switch will ensure only the `bundle_id` of the loaded bundle is displayed on the standard output. The `bundle_id` then can be referenced from the subsequent commands.

In the example above the bundle shorthand expression `reactive-maps-backend-region`, `reactive-maps-backend-summary`, and `reactive-maps-frontend` will be resolved against the [Reactive Maps](http://www.lightbend.com/activator/template/reactive-maps) bundle published to Bintray.

## Bundle resolution process

When `conduct load` or `conduct deploy` is invoked, bundle and the optional configuration are supplied as the input.

As part of the resolution process, a list of resolvers will be selected to resolve the given bundle input. This list of resolvers is called resolver chain. Both `conduct load` or `conduct deploy` has a default resolver chain which is used in most situations, as well as an offline resolver chain. When working during `--offline` mode, the offline resolver chain will be used. It is possible to override the default resolver chain, and this is discussed as part of implementing [custom resolver](#Implementing-your-own-custom-resolver).

Each resolver declares a list of supported URI schemes. For example, `bintray_resolver` only accepts [bundle shorthand expression](#Bundle-shorthand-expression) and hence `bundle` scheme, while `uri_resolver` accepts `file`, `http`, and `https` scheme.

The following steps are taken during bundle resolution process.

First, a resolver chain is selected. The resolver chain can be either default, offline, or custom.

Next, a list of possible URI schemes is deduced from the input URI. For example, if `my-app` is given as the input, then the the `bundle` scheme applies to the URI because `my-app` is a valid [bundle shorthand expression](#Bundle-shorthand-expression). If `my-app` also exists in the local path, then the `file` scheme is also applicable. So in the above example, the schemes `bundle` and `file` apply to `my-app`.

The list of possible URI schemes is then compared to the list of URI schemes supported by each resolver in the selected chain. Only resolvers which can accept the selected URI schemes are used during the bundle resolution process.

The input is then passed into each remaining resolver in the chain until a successful resolution can be found. The result of a successful resolution is the bundle artifact which will then be cached. If none of the remaining resolvers in the chain can resolve the input, an error is raised.

Once the resolution is complete for a bundle, this process will be repeated for a bundle configuration, if a configuration has been supplied as part of the input.

### Default resolver chain

The following resolvers are declared as part of default resolver chain. The resolution will go through the resolvers in the following order.

* `stdin_resolver`
* `uri_resolver`
* `bintray_resolver`
* `docker_resolver`
* `s3_resolver`

### Offline resolver chain

During the `--offline` mode, both `conduct load` or `conduct deploy` will not be making calls to external network resources, i.e. Bintray or Docker Hub.

The following resolvers are declared as part of offline resolver chain. The resolution will go through the resolvers in the following order.

* `stdin_resolver`
* `offline_resolver`
* `docker_offline_resolver`

### Supported schemes

The URI schemes which are supported by default are declared in `conductr_cli.resolvers.scheme`.

```
SCHEME_BUNDLE = 'urn:x-bundle'
SCHEME_STDIN = 'stdin'
SCHEME_FILE = 'file'
SCHEME_HTTP = 'http'
SCHEME_HTTPS = 'https'
SCHEME_S3 = 's3'
```


Scheme         | Supporting Resolver(s)                | Description
---------------|---------------------------------------|------------
Bundle         | `bintray_resolver`, `docker_resolver` | [bundle shorthand expression](#Bundle-shorthand-expression) which are translated to Bintray download and Docker image download by `bintray_resolver` and `docker_resolver` respectively.
Stdin          | `stdin_resolver`                      | Invoked during `stdin` input. If `stdin` input is detected, no other schemes will be applied to the input URI.
File           | `uri_resolver`                        | Actual path to the file system. Can be declared as standard file path, i.e. `/tmp/my-bundle.zip`, or declared as file URI, i.e. `file:///tmp/my-bundle.zip`
HTTP and HTTPS | `uri_resolver`                        | HTTP or HTTPS address where artifact can be downloaded.
S3             | `s3_resolver`                         | S3 address where artifact can be downloaded.

### Note on stdin

If `stdin` input is detected, no other schemes will be applied to the input URI.

## Built-in bundle resolvers

The CLI comes with built-in URI and Bintray resolvers which comes into play when `conduct load` command is invoked.

### URI resolver

Located at the [conductr_cli.resolvers.uri_resolver](https://github.com/typesafehub/conductr-cli/blob/master/conductr_cli/resolvers/uri_resolver.py) Python package.

Supported schemes:

* File
* HTTP
* HTTPS

As such, the URI resolver accepts local file system path as well as HTTP URL, e.g.

```
conduct load /tmp/downloads/reactive-maps-frontend-1.0.0-023f9da2243a0751c2e231b452aa3ed32fbc35351c543fbd536eea7ec457cfe2.zip
conduct load http://192.168.0.1/files/reactive-maps-frontend-1.0.0-023f9da2243a0751c2e231b452aa3ed32fbc35351c543fbd536eea7ec457cfe2.zip
```

### Bintray resolver

Located at the [conductr_cli.resolvers.bintray_resolver](https://github.com/typesafehub/conductr-cli/blob/master/conductr_cli/resolvers/bintray_resolver.py) Python package.

Supported scheme:

* Bundle

The Bintray resolver accepts a [bundle shorthand expression](#Bundle-shorthand-expression) which is translated to a Bintray download URL.

```
conduct load reactive-maps-frontend
```

Bundles can be published to Bintray using the [sbt-bintray-bundle](https://github.com/sbt/sbt-bintray-bundle) plugin.

### Docker resolver

Located at the [conductr_cli.resolvers.docker_resolver](https://github.com/typesafehub/conductr-cli/blob/master/conductr_cli/resolvers/docker_resolver.py) Python package.

Supported scheme:

* Bundle

The Docker resolver accepts the name and optional tag of a docker image. It allows you to load images from a Docker registry directly into ConductR.

```
conduct load my-docker-image:my-tag
```

### Docker offline resolver

Located at the [conductr_cli.resolvers.docker_offline_resolver](https://github.com/typesafehub/conductr-cli/blob/master/conductr_cli/resolvers/docker_resolver.py) Python package.

Supported scheme:

* Bundle

The Docker resolver accepts the name and optional tag of a docker image. It allows loading a cached Docker image into ConductR.

```
conduct load my-docker-image:my-tag
```

As such, no new Docker image can be loaded during `--offline` mode as it requires network connectivity to Docker Hub.

### stdin resolver

Located at the [conductr_cli.resolvers.stdin_resolver](https://github.com/typesafehub/conductr-cli/blob/master/conductr_cli/resolvers/stdin_resolver.py) Python package.

Supported scheme:

* Stdin

The stdin resolver can be useful for composing programs together in UNIX-based fashion. For example, the following command uses `docker save` and `conduct load` to export your local Docker image into ConductR.

```
docker save my-local-docker-image:my-tag | conduct load
```

### Offline resolver

Located at the [conductr_cli.resolvers.offline_resolver](https://github.com/typesafehub/conductr-cli/blob/master/conductr_cli/resolvers/offline_resolver.py) Python package.

Supported scheme:

* Bundle
* File

The `offline_resolver` is used when `--offline` mode is specified. The resolver will attempt to load locally cached artefact into ConductR, i.e.

```
conduct load --offline visualizer
```

The resolver also accepts local file system path.

```
conduct load --offline /tmp/my-bundle.zip
```

### S3 resolver

Located at the [conductr_cli.resolvers.s3_resolver](https://github.com/typesafehub/conductr-cli/blob/master/conductr_cli/resolvers/s3_resolver.py) Python package.

Supported scheme:

* S3

The S3 resolver allows resolving bundle from Amazon S3. The S3 resolver accepts an S3 URL as the input.

```
conduct load \
  s3://my-org-bucket/files/bundle/my-bundle.zip \
  s3://my-org-bucket/files/bundle-configuration/test-configuration.zip
```

The S3 URL has the following format.

```
s3://<bucket-name>/<key to bundle or configuration zip file>
```

Based on the example above, the bundle input S3 URL `s3://my-org-bucket/files/bundle/my-bundle.zip` has `my-org-bucket` as the S3 bucket name, and `files/bundle/my-bundle.zip` as the S3 key. Similarly, the configuration input `s3://my-org-bucket/files/bundle-configuration/test-configuration.zip` has `my-org-bucket` as the bucket name and `files/bundle-configuration/test-configuration.zip` as the key.

The S3 resolver uses AWS Python client library `Boto 3` to authenticate and access the S3 resources. To access protected S3 resources, the `Boto 3` library requires authentication to be configured.

Create the credentials file at `~/.aws/credentials`. Populate the credentials file:

```
[default]
aws_access_key_id = AWS_ACCESS_KEY
aws_secret_access_key = AWS_SECRET_KEY

[conductr]
aws_access_key_id = AWS_ACCESS_KEY
aws_secret_access_key = AWS_SECRET_KEY
```

The S3 resolver will attempt to use credentials assigned to the `conductr` profile. The `conductr` profile is not defined, the S3 resolver will fallback to `default` profile.

Refer to [Boto 3 documentation](http://boto3.readthedocs.io/en/latest/guide/quickstart.html#configuration) for more details on the credentials configuration.

The `AWS_ACCESS_KEY` can be obtained from the [IAM Console](https://console.aws.amazon.com/iam/home). `AWS_SECRET_KEY` is shown only when `AWS_ACCESS_KEY` is created, as such a new key needs to be generated if the `AWS_SECRET_KEY` is not known.


## Bundle shorthand expression

The shorthand bundle expression has the following format:

```
[ "urn:x-bundle:" ] , [ organization , "/" ] , [ repository , "/" ] , package , [ ":" , tag [ "-" , digest ] ]
```

The usage of the shorthand expression is best illustrated with the following example.

Shorthand Expression                                                                                 | Resolved to
-----------------------------------------------------------------------------------------------------|------------
reactive-maps-frontend                                                                               | Latest version of the `reactive-maps-frontend` bundle hosted within `organization` called `typesafe` and `repository` called `bundle`.
reactive-maps-frontend:1.0.0                                                                         | Latest version of the `reactive-maps-frontend` bundle having `tag` of `1.0.0` hosted within `organization` called `typesafe` and `repository` called `bundle`.
reactive-maps-frontend:1.0.0.beta.1-023f9da2243a0751c2e231b452aa3ed32fbc35351c543fbd536eea7ec457cfe2 | `reactive-maps-frontend` bundle having `tag` of `1.0.0-beta.1` and `digest` of `023f9da2243a0751c2e231b452aa3ed32fbc35351c543fbd536eea7ec457cfe2` hosted within `organization` called `typesafe` and `repository` called `bundle`.
my-company/secret-repo/super-bundle:0.1.0                                                            | Latest version of the `super-bundle` bundle having `tag` of `0.1.0` hosted within `organization` called `my-company` and `repository` called `secret-repo`.

## Implementing your own custom resolver

The CLI tool is written in Python to support Python 3 and above, and hence the custom resolver must be written in Python 3.

Let us assume we want to provide a custom resolver to resolve the following custom URL.

```
acme://<team-name>/<artefact-name>
```

Here are the steps to implement a custom resolver:

1. Create the custom resolver file in `~/.conductr/plugins/my_resolver.py`
2. Create the custom resolver as such:

```
from conductr_cli.resolvers import uri_resolver
# other imports here...


def supported_schemes():
    return ['acme']

def resolve_bundle(cache_dir, uri):
    team_name, artefact_name = extract_team_and_artefact_name(uri)
    actual_http_url_or_file_path = do_convert(team_name, artefact_name)
    return uri_resolver.resolve_bundle(cache_dir, actual_http_url_or_file_path)


def load_bundle_from_cache(cache_dir, uri):
    team_name, artefact_name = extract_team_and_artefact_name(uri)
    actual_http_url_or_file_path = do_convert(team_name, artefact_name)
    return uri_resolver.load_bundle_from_cache(cache_dir, actual_http_url_or_file_path)


def resolve_bundle_configuration(cache_dir, uri):
    team_name, artefact_name = extract_team_and_artefact_name(uri)
    actual_http_url_or_file_path = do_convert(team_name, artefact_name)
    return uri_resolver.resolve_bundle_configuration(cache_dir, actual_http_url_or_file_path)


def load_bundle_configuration_from_cache(cache_dir, uri):
    team_name, artefact_name = extract_team_and_artefact_name(uri)
    actual_http_url_or_file_path = do_convert(team_name, artefact_name)
    return uri_resolver.load_bundle_configuration_from_cache(cache_dir, actual_http_url_or_file_path)


def extract_team_and_artefact_name(uri):
    # Use own logic to extract `uri` string into team name and artefact name
    ...


def do_convert(team_name, artefact_name):
    # Use own logic to convert `team_name` and `artefact_name` into actual http url or file path
    ...
```

Looking at the `supported_schemes` method, our custom resolver supports `acme` as the URI scheme. If the URL `acme://my-team/super-artefact` is given as the input, our custom resolver will be used as it supports the `acme` scheme.

Default schemes are declared as constants, and can be referenced as such if you wish to do so in your custom resolver.

```
from conductr_cli.resolvers.schemes import SCHEME_BUNDLE, SCHEME_FILE, \
    SCHEME_HTTP, SCHEME_HTTPS, SCHEME_S3, SCHEME_STDIN
```

In the example above, once the supplied `uri` is eventually resolved to either HTTP URL or actual file path, `conductr_cli.resolvers.uri_resolver` is used to continue processing as it has logic to handle HTTP URL or file path input.

3. Configure the custom resolver in the `~/.conductr/settings.conf`

```
resolvers = [
  my_resolver,
  conductr_cli.resolvers.stdin_resolver,
  conductr_cli.resolvers.uri_resolver,
  conductr_cli.resolvers.bintray_resolver,
  conductr_cli.resolvers.docker_resolver,
  conductr_cli.resolvers.s3_resolver
]
```

The resolution will follow the sequence of resolvers declared in `~/.conductr/settings.conf`. Based on the example above, if `my_resolver` returns a bundle and bundle configuration (either from cache or a new download), then the remaining resolvers will not be invoked.

**Do not use `print`** - use built-in Python logging library instead. This will allow correct output when `-q` is supplied to `conduct load` command. Using `print` instead of Python logging library will break [orchestration](#Using-CLI-to-orchestrate-bundle-deployments) of bundle deployments.
