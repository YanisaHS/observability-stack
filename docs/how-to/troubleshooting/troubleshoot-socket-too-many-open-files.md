# Troubleshooting ``socket: too many open files``

When deploying the Grafana Agent or Prometheus charms in large environments, 
you may sometimes bump into an issue where the large amount of scrape targets 
leads to the process hitting the max open files count, as set by ``ulimit``.

This issue can be identified by looking in your Grafana Agent logs, or Prometheus 
Scrape Targets in the UI, for the following kind of message:

```
Get "http://10.0.0.1:9275/metrics": dial tcp 10.0.0.1:9275: socket: too many open files
```

To resolve this, we need to increase the max open file limit of the Kubernetes 
deployment itself. For MicroK8s, this would be done by increasing the limits in 
`/var/snap/microk8s/current/args/containerd-env`.

## 1. Juju SSH into the machine

```bash
$ juju ssh uk8s/1
```

Substitute `uk8s/1` with the name of your MicroK8s unit. If you have more than 
one unit, you will need to repeat this for each of them.

## 2. Open the ``containerd-env``

You can use whatever editor you prefer for this. In this how-to, we'll use ``vim``.

```bash
$ vim /var/snap/microk8s/current/args/containerd-env
```

## 3. Increase the `ulimit`

```diff

# Attempt to change the maximum number of open file descriptors
# this get inherited to the running containers
#
- ulimit -n 1024 || true
+ ulimit -n 65536 || true

# Attempt to change the maximum locked memory limit
# this get inherited to the running containers
#
- ulimit -l 1024 || true
+ ulimit -l 16384 || true
```

## 4. Restart the MicroK8s machine

Restart the machine the MicroK8s unit is deployed on and then wait for it to come back up.

```bash 
$ sudo reboot
```

## 5. Validate

Validate that the change made it through and had the desired effect once the machine is 
back up and running.

```bash
$ juju ssh uk8s/1 cat /var/snap/microk8s/current/args/containerd-env

[...]

# Attempt to change the maximum number of open file descriptors
# this get inherited to the running containers
#
ulimit -n 65536 || true

# Attempt to change the maximum locked memory limit
# this get inherited to the running containers
#
ulimit -l 16384 || true
```