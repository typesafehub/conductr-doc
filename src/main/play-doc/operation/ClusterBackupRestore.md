# Backup ConductR Cluster

To backup a ConductR cluster use the backup command. The backup command can be used to backup an entire cluster or individual bundles.

## Backing up an entire cluster

The backup includes all the bundles along with their configuration. The backup also includes member and agent configurations.

The following command enables the backup of an entire cluster. The output flag (`-output/-o`) can be used to specify the path to the backup.

```bash
conduct backup -o backup.zip
```

The output from the `backup` command can also be written to stdout.
```bash
conduct backup > backup.zip
```

## Backing up a bundle

The backup command can also be used to backup a single bundle.

```bash
conduct backup visualizer > backup.zip
```

The backup command can take the bundle name or bundle id as input. Both long and short bundle ids are supported.
If there is ambiguity in bundle name/id, it will result in error similar to other conduct cli commands.

Once the backup is completed, it can be optionally combined with the restore command.
