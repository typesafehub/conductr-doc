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

Note that by default, bundles have a maximum size of 100MB. This can be altered via the `akka.http.server.parsing.max-content-length` setting.

Use `conduct info` command to list all loaded bundles. You should see that the Visualizer is replicated but not running (note that the example below shows 3 replications - you'll only get that if you have 3 or more nodes as the bundle cannot replicate beyond the cluster size).

```bash
conduct info --ip 172.17.0.1
```
...will yield something like:

```bash
ID       NAME          TAG  #REP  #STR  #RUN  ROLES
6cc7dbc  visualizer  2.0.0     3     0     0  web
```

Run the Visualizer by executing:

```bash
conduct run --ip 172.17.0.1 visualizer
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


## Built-in bundle resolvers

The CLI comes with built-in URI and Bintray resolvers which comes into play when `conduct load` command is invoked.

### URI resolver

The URI resolver accepts local file system path as well as HTTP URL, e.g.

```
conduct load /tmp/downloads/reactive-maps-frontend-1.0.0-023f9da2243a0751c2e231b452aa3ed32fbc35351c543fbd536eea7ec457cfe2.zip
conduct load http://192.168.0.1/files/reactive-maps-frontend-1.0.0-023f9da2243a0751c2e231b452aa3ed32fbc35351c543fbd536eea7ec457cfe2.zip
```

### Bintray resolver

The Bintray resolver accepts a [bundle shorthand expression](#Bundle-shorthand-expression) which is translated to a Bintray download URL.

```
conduct load reactive-maps-frontend
```

Bundles can be published to Bintray using the [sbt-bintray-bundle](https://github.com/sbt/sbt-bintray-bundle) plugin.


## Bundle shorthand expression

The shorthand bundle expression has the following format:

```
[ "urn:x-bundle:" ] , [ organization , "/" ] , [ repository , "/" ] , package , [ ":" , tag [ "-" , digest ] ]
```

The usage of the shorthand expression is best illustrated with the following example.

Shorthand Expression                                                                       | Resolved to
-------------------------------------------------------------------------------------------|------------
reactive-maps-frontend                                                                     | Latest version of the `reactive-maps-frontend` bundle hosted within `organization` called `typesafe` and `repository` called `bundle`.
reactive-maps-frontend:1.0.0                                                               | Latest version of the `reactive-maps-frontend` bundle having `tag` of `1.0.0` hosted within `organization` called `typesafe` and `repository` called `bundle`.
reactive-maps-frontend:1.0.0.beta.1-023f9da2243a0751c2e231b452aa3ed32fbc35351c543fbd536eea7ec457cfe2 | `reactive-maps-frontend` bundle having `tag` of `1.0.0-beta.1` and `digest` of `023f9da2243a0751c2e231b452aa3ed32fbc35351c543fbd536eea7ec457cfe2` hosted within `organization` called `typesafe` and `repository` called `bundle`.
my-company/secret-repo/super-bundle:0.1.0                                                     | Latest version of the `super-bundle` bundle having `tag` of `0.1.0` hosted within `organization` called `my-company` and `repository` called `secret-repo`.


## Implementing your own custom resolver

The CLI tool is written in Python to support Python 3 and above, and hence the custom resolver must be written in Python 3.

Here are the steps to implement a custom resolver:

1. Create the custom resolver file in `~/.conductr/plugins/my_resolver.py`
2. Create the custom resolver as such:

```
from conductr_cli.resolvers import uri_resolver
# other imports here...


def resolve_bundle(cache_dir, uri):
    actual_http_url_or_file_path = do_convert(uri)
    return uri_resolver.resolve_bundle(cache_dir, actual_http_url_or_file_path)


def load_bundle_from_cache(cache_dir, uri):
    actual_http_url_or_file_path = do_convert(uri)
    return uri_resolver.load_bundle_from_cache(cache_dir, actual_http_url_or_file_path)


def resolve_bundle_configuration(cache_dir, uri):
    actual_http_url_or_file_path = do_convert(uri)
    return uri_resolver.resolve_bundle_configuration(cache_dir, actual_http_url_or_file_path)


def load_bundle_configuration_from_cache(cache_dir, uri):
    actual_http_url_or_file_path = do_convert(uri)
    return uri_resolver.load_bundle_configuration_from_cache(cache_dir, actual_http_url_or_file_path)


def do_convert(uri):
    # Use own logic to convert uri string into actual http url or file path
    ...
```

In the example above, once the supplied `uri` is resolved to either HTTP URL or actual file path, `conductr_cli.resolvers.uri_resolver` is used to continue processing as it has logic to handle HTTP URL or file path input.

3. Configure the custom resolver in the `~/.conductr/settings.conf`

```
resolvers = [
  my_resolver,
  conductr_cli.resolvers.bintray_resolver,
  conductr_cli.resolvers.uri_resolver
]
```

The resolution will follow the sequence of resolvers declared in `~/.conductr/settings.conf`. Based on the example above, if `my_resolver` returns a bundle and bundle configuration (either from cache or a new download), then the remaining resolvers will not be invoked.

**Do not use `print`** - use built-in Python logging library instead. This will allow correct output when `-q` is supplied to `conduct load` command. Using `print` instead of Python logging library will break [orchestration](#Using-CLI-to-orchestrate-bundle-deployments) of bundle deployments.
