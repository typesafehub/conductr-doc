# ConductR Migration Guide

This document describes what is required to move between version 2.0 and 2.1.

## Cluster incompatibility

ConductR 2.1's cluster protocol is not compatible with 2.0 or 1.1. As such, you will need to install ConductR 2.1 on to a new set of machines.

## Production Suite Licensing

A license is now required to run ConductR 2.1 onwards. Not having a license will restrict you to just one ConductR agent.

### Obtain license

To obtain a license please download the latest CLI, type `conduct load-license` and follow the instructions it provides. Once authenticated at lightbend.com, `load-license` will download your license, connect to the ConductR cluster via either by setting the environment variable `CONDUCTR_HOST` or by providing the CLI option `--host` and then upload the license. You can invoke `conduct load-license` any number of times, changing the ConductR IP address to upload to accordingly.

### Verify license

To see the license you have, please type the `conduct info` command. This command has been upgraded to display license information. The command will first connect with your ConductR cluster via either the `CONDUCTR_HOST` or `--host`, and then display your license.

## Bundle endpoints

Bundle endpoints using `services` instead of `acls` are no longer accepted. `services` were deprecated in 2.0. Please ref to the [ACL documentation](AclConfiguration) for reference on how to express ACLs.
