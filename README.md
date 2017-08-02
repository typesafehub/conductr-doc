# ConductR Documentation

Documentation for conductr.lightbend.com

Feel free to contribute and correct!

## How to run ConductR doc locally

You will require `sass` or `compass` on the `PATH`. Refer to [compass installation](http://compass-style.org/install/) or [sass installation](http://sass-lang.com/install) for the actual installation step for your platform.

Clone the `project-doc`:

```bash
$ git clone https://github.com/typesafehub/project-doc
```

`cd` into the `project-doc` directory:

```bash
$ cd project-doc
```

Modify the `app/modules/ConductRDocRendererModule.scala` file to point to your PR change:
* For example we are looking at [PR 405](https://github.com/typesafehub/conductr-doc/pull/405).
* This PR is based off the `seglo` repo, under the branch called `licensing`.
* Modify `ConductRDocRendererProvider21` to point to the PR repo and branch:

```
@Singleton
class ConductRDocRendererProvider21 @Inject()(actorSystem: ActorSystem, wsClient: WSClient)
  extends ConductRDocRendererProvider(
    actorSystem,
    wsClient,
    new URI("https://github.com/seglo/conductr-doc/archive/licensing.zip"),
    "2.1.x",
    versions
  )
```

Once this is done, run the `project-doc`:

```bash
$ sbt run
```

You will need to restart the `project-doc` process to immediately see the changes pushed to the PR branch.

&copy; Lightbend Inc., 2015-2016
