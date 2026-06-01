HCP notes
=========

This repo helps to setup HCP. Pre-reqs.

* OCP deployed
* Metal LB
* LVM Storage
* OCP Virt
* Pull secret

Deploying
------------

Deploy MCE operator and instance.

Get cluster pull secret.

```
oc get secret/pull-secret -n openshift-config --template='{{index .data ".dockerconfigjson" | base64decode}}' > /root/pull-secret
```

Make sure wildcard routes allowed:

```
oc patch ingresscontroller -n openshift-ingress-operator default --type=json -p '[{ "op": "add", "path": "/spec/routeAdmission", "value": {wildcardPolicy: "WildcardsAllowed"}}]'
```

Create cluster. Name here is pharriso.

```
hcp create cluster kubevirt --name pharriso --node-pool-replicas 2 --memory 8Gi --cores 2 --etcd-storage-class=lvms-vg1 --pull-secret /root/pull-secret  --release-image quay.io/openshift-release-dev/ocp-release:4.17.9-x86_64
```

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

Patch the Loadbalancer. Get the ports for http and https

```
oc --kubeconfig /tmp/pharriso-kubeconfig get services -n openshift-ingress router-nodeport-default -o jsonpath='{.spec.ports[?(@.name=="https")].nodePort}'
oc --kubeconfig /tmp/pharriso-kubeconfig get services -n openshift-ingress router-nodeport-default -o jsonpath='{.spec.ports[?(@.name=="http")].nodePort}'
```

Update the lb.yaml file and apply it.


Deploy HCP with multiple storageclass mappings:

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

Proxy considerations
------------

For a environment behind a proxy, you need to add the proxy details into the InfraEnv so that it can pull images as part of agent discovery. You can get the hosting cluster proxy settings as follows:

```
oc get proxy cluster -o yaml
```

In the spec for the InfraEnv add:


```
 spec:
    proxy:
      httpProxy: http://192.168.50.1:3128
      httpsProxy: http://192.168.50.1:3128
      noProxy: .cluster.local,.ocp4.pharriso.co.uk,.svc,10.128.0.0/14,127.0.0.1,172.30.0.0/16,192.168.50.0/24,api-int.ocp4.pharriso.co.uk,localhost
```

When deploying the hostedcluster, you need to add the proxy details for the nodepools to pick up proxy settings. HCP pods inherit cluster-wide proxy settings from the management cluster. In the hostedcluster object, add the following:


```
configuration:
      proxy:
        httpProxy: http://192.168.50.1:3128
        httpsProxy: http://192.168.50.1:3128
        noProxy: .cluster.local,.ocp4.pharriso.co.uk,.svc,10.128.0.0/14,127.0.0.1,172.30.0.0/16,192.168.50.0/24,api-int.ocp4.pharriso.co.uk,localhost
```

Proxy settings get set on the worker nodes in /etc/mco/proxy.env

This env file is then included in the systemd files for the relevant services. For example kubelet

/etc/systemd/system/kubelet.service.d contains the env file to include /etc/mco/proxy.env

Also see KCS = https://access.redhat.com/solutions/7090263
