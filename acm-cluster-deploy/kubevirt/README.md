# HCP Kubevirt

This repo contains instructions for using kubevirt with HCP.

## Wilcard subdomains

Make sure wildcard routes allowed if you want subdomains of management cluster.

```
oc patch ingresscontroller -n openshift-ingress-operator default --type=json -p '[{ "op": "add", "path": "/spec/routeAdmission", "value": {wildcardPolicy: "WildcardsAllowed"}}]'
```

## Render yaml

You can generate the yaml and secrets running one of the deploy scripts and then apply the rendered yaml.

## Troubleshooting and config

If you want to use nodeport and external LB then you can config the servicepublishing. Example below moves API server to nodeport which requires external LB config.

```
services:
  - service: APIServer
    servicePublishingStrategy:
      type: NodePort
      nodePort:
        address: api.cluster2.pharriso.co.uk
        port: 31050
  - service: Ignition
    servicePublishingStrategy:
      type: Route
```

## Create a cluster with hcp CLI

Create a cluster called pharriso

```
hcp create cluster kubevirt --name pharriso --node-pool-replicas 2 --memory 8Gi --cores 2 --etcd-storage-class=lvms-vg1 --pull-secret /root/pull-secret  --release-image quay.io/openshift-release-dev/ocp-release:4.17.9-x86_64
```

## Check deployment state


Get the HCP kubeconfig.

```
hcp create kubeconfig --name pharriso > /tmp/pharriso-kubeconfig
```

Check the cluster state

```
oc --kubeconfig /tmp/pharriso-kubeconfig get co
oc get hostedclusters.hypershift.openshift.io  -n clusters
```

In case of some arp issues and you can't connect to kube api.

```
oc get svc -n clusters-pharriso
arping <IP>
```

## MetalLB - patch ingress on the hosted cluster.

Patch the Loadbalancer. Get the ports for http and https

```
oc --kubeconfig /tmp/pharriso-kubeconfig get services -n openshift-ingress router-nodeport-default -o jsonpath='{.spec.ports[?(@.name=="https")].nodePort}'
oc --kubeconfig /tmp/pharriso-kubeconfig get services -n openshift-ingress router-nodeport-default -o jsonpath='{.spec.ports[?(@.name=="http")].nodePort}'
```

Update the lb.yaml file and apply it.


## Deploy HCP with multiple storageclass mappings:

```
hcp create cluster kubevirt \
  --name pharriso \
  --node-pool-replicas 1 \
  --control-plane-availability-policy SingleReplica \
  --pull-secret /root/pull-secret \
  --memory 6Gi \
  --cores 2 \
  --etcd-storage-class ocs-storagecluster-ceph-rbd \
  --infra-storage-class-mapping=ocs-storagecluster-ceph-rbd/kubevirt-ceph-rbd \
  --infra-storage-class-mapping=ocs-storagecluster-cephfs/kubevirt-ceph-cephfs \
  --release-image quay.io/openshift-release-dev/ocp-release:4.17.9-x86_64
```
