# HCP Kubevirt

This repo contains instructions for using kubevirt with HCP.

## Generate yaml

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


