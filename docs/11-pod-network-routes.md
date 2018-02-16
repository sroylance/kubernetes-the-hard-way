# Provisioning Pod Network Routes

Pods scheduled to a node receive an IP address from the node's Pod CIDR range. At this point pods can not communicate with other pods running on different nodes due to missing network [routes](https://cloud.google.com/compute/docs/vpc/routes).

In this lab you will install flannel to manage pod networking.

> There are [other ways](https://kubernetes.io/docs/concepts/cluster-administration/networking/#how-to-achieve-this) to implement the Kubernetes networking model.

## POD cidr assignments
```
kubectl patch node ip-10-251-0-20.ec2.internal -p '{"spec":{"podCIDR":"100.64.0.0/24"}}'
kubectl patch node ip-10-251-4-20.ec2.internal -p '{"spec":{"podCIDR":"100.64.1.0/24"}}'
kubectl patch node ip-10-251-8-20.ec2.internal -p '{"spec":{"podCIDR":"100.64.2.0/24"}}'
```

## Flannel

```
kubectl apply -f https://raw.githubusercontent.com/sroylance/kubernetes-the-hard-way/aws-ify/deployments/kube-flannel.yml
```

## Verification
```
kubectl get pods --all-namespaces
```

output
```
NAMESPACE     NAME                    READY     STATUS    RESTARTS   AGE
kube-system   kube-flannel-ds-hn49s   1/1       Running   0          22s
kube-system   kube-flannel-ds-wmgsv   1/1       Running   0          22s
kube-system   kube-flannel-ds-xgndr   1/1       Running   0          22s

```


Next: [Deploying the DNS Cluster Add-on](12-dns-addon.md)
