# Baremetal HCP

## Deploy Central Infrastructure Management

This is essentially the assisted agent service. It requires some persistent storage for a database, coreos images and HCP manifest/kubeconfig/logs.

```
oc apply -f agent-service-config.yaml
```

## Get cluster pull secret.

```
oc get secret/pull-secret -n openshift-config --template='{{index .data ".dockerconfigjson" | base64decode}}' > /root/pull-secret.json
```

Base64 encode it:

```
cat /root/pull-secret.json | jq -c | base64 -w 0
```

## InfraEnv 

Create an InfraEnv to discover bare metal hosts. They should be grouped based on characteristics and also they are namespaced for allocation to HCP clusters.

Add the encoded pull secret and SSH key to the infraenv and apply:

```
oc apply -f infraenv.yaml
```

## Redfish and nmstate config

Create redfish secret and nmstate:

```
oc apply -f redfish-secret.yaml
oc apply -f nmstate-cluster2.yaml
```

## Create BMH

```
oc apply -f bmh-cluster2.yaml
```

Host should inspect and become available. For proxy examples, use a different interface e.g. podman0

## Deploy a cluster

```
oc apply -f deploy-hcp.yaml
```

## Get the HCP kubeconfig.

```
hcp create kubeconfig --name pharriso > /tmp/pharriso-kubeconfig
```

## Check the cluster state

```
oc --kubeconfig /tmp/pharriso-kubeconfig get co
oc get hostedclusters.hypershift.openshift.io  -n clusters
```

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

## Proxy considerations

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
