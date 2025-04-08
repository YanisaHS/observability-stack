# Troubleshooting `Gateway address unavailable`

Whenever Traefik is used to ingress your Kubernetes workloads, you might in some specific
cases encounter a "Gateway Address Unavailable" message. In this article, we'll go through 
what you can do to remediate it.

```{caution}
In this article, we will assume that you are running MicroK8s on either a bare-metal 
or virtual machine. If your setup differs from this, parts of the how-to may still 
apply, although you will need to tailor the exact steps and commands to your setup.
```

## Checklist

- You have run `juju trust traefik --scope=cluster`
- The [MetalLB MicroK8s add-on](https://microk8s.io/docs/addon-metallb) is enabled.
- Traefik's service type is ``LoadBalancer``.
- An external IP address is assigned to Traefik.

## Possible causes

### The MetalLB add-on isn't enabled

Check with:

```bash 
$ microk8s status -a metallb
```

If it is disabled, you can enable it with:

```bash 
$ IPADDR=$(ip -4 -j route get 2.2.2.2 | jq -r '.[] | .prefsrc')
$ microk8s enable metallb:$IPADDR-$IPADDR
```

This command will fetch the IPv4 address assigned to your host, and hand it to MetalLB 
as an assignable IP. If the address range you want to hand to MetalLB differs from your 
host IP, alter the `$IPADDR` variable to instead specify the range you want to assign, 
for instance `IPADDR=10.0.0.1-10.0.0.100`.

### No external IP address is assigned to the Traefik service

Does the Traefik service have an external IP assigned to it? Check with:

```bash
$ JUJU_APP_NAME="traefik"
$ kubectl get svc -A -o wide | grep -E "^NAMESPACE|$JUJU_APP_NAME"
```

### No available IP in address pool

This can happen when: 
- MetalLB has only one IP in its range but you deployed two instances of Traefik, 
  or when Traefik is forcefully removed (`--force --no-wait`) and a new Traefik 
  app is deployed immediately after.
- The [ingress](https://microk8s.io/docs/ingress) add-on is enabled. It's possible
  that Nginx from the ingress add-on has claimed the `ExternalIP`. Disable Nginx and 
  re-enable MetalLB.

Check with:

```bash
$ kubectl get ipaddresspool -n metallb-system -o yaml && kubectl get all -n metallb-system
```

You could add more IPs to the range:

```bash
$ FROM_IP="..." TO_IP="..."
$ microk8s enable metallb:$FROM_IP-$TO_IP
```

### The Load Balancer service type reverted to `ClusterIP`

Juju controller cycling may cause the type to revert from `LoadBalancer` back to 
`ClusterIP`. 

Check with:

```bash
$ kubectl get svc -A -o wide | grep -E "^NAMESPACE|LoadBalancer"
```

If Traefik isn't listed (it's not `LoadBalancer`), then recreate the pod to have it 
re-trigger the assignment of the external IP with `kubectl delete` . It should be `LoadBalancer` 
when Kubernetes brings it back.

### Integration tests pass locally but fail on GitHub runners

This used to happen when the github runners were at peak usage, making the already small 2cpu7gb 
runners run even slower. As much of a bad answer as this is, the best response may be to increase 
timeouts or try to move CI jobs to internal runners.

## Verification

Verify that the Traefik Kubernetes service now has been assigned an external IP:

```

$ microk8s.kubectl get services -A

NAMESPACE         NAME                     TYPE           CLUSTER-IP       EXTERNAL-IP                 PORT(S)
cos               traefik                  LoadBalancer   10.152.183.130   10.70.43.245                80:32343/TCP,443:30698/TCP   4d3h
                                                                            👆 - This one!
```

Verify that Traefik is functioning correctly by trying to trigger one of your ingress paths. 
If you have COS Lite deployed, you may check that if works as expected using the Catalogue charm:

```bash 
# curl http://<TRAEFIKS_EXTERNAL_IP>/<YOUR_MODEL_NAME>-catalogue/
# for example...
$ curl http://10.70.43.245/cos-catalogue/
```

This command should return a long HTML code block if everything works as expected.