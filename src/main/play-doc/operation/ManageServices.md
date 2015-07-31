# Managing ConductR services

During installation ConductR registers a linux service named `conductr`. If HAProxy is installed, the service `conductr-haproxy` is registered as well. Both services are started automatically during boot-up.

## Service user
The `conductr` service runs as the daemon user `conductr` in the user group `conductr`. When the service is started the first time it creates the user and group itself.

The `conductr` user executes the commands in the background without any shell. You can specify additional environment variables in `/etc/default/conductr`. This file will be sourced before the actual service gets started.


## Change service state

In order to start, stop or restart ConductR on one node change the state of the service.

**sysvinit**

```bash
sudo service conductr start
sudo service conductr stop
sudo service conductr restart
```
