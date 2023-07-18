# Using the Networking Calico extension with Gardener as end-user

The [`core.gardener.cloud/v1beta1.Shoot` resource](https://github.com/gardener/gardener/blob/master/example/90-shoot.yaml) declares a `networking` field that is meant to contain network-specific configuration.

In this document we are describing how this configuration looks like for Calico and provide an example `Shoot` manifest with minimal configuration that you can use to create a cluster.

## Calico Typha

Calico Typha is an optional component of Project Calico designed to offload the Kubernetes API server. The Typha daemon sits between the datastore (such as the Kubernetes API server which is the one used by Gardener managed Kubernetes) and many instances of Felix. Typha’s main purpose is to increase scale by reducing each node’s impact on the datastore. You can opt-out Typha via `.spec.networking.providerConfig.typha.enabled=false` of your Shoot manifest. By default the Typha is enabled.

## EBPF Dataplane

Calico can be run in ebpf dataplane mode. This has several benefits, calico scales to higher troughput, uses less cpu per GBit and has native support for kubernetes services (without needing kube-proxy).
To switch to a pure ebpf dataplane it is recommended to run without an overlay network. The following configuration can be used to run without an overlay and without kube-proxy.

An example ebpf dataplane `NetworkingConfig` manifest:

```yaml
apiVersion: calico.networking.extensions.gardener.cloud/v1alpha1
kind: NetworkConfig
ebpfDataplane:
  enabled: true
overlay:
  enabled: false
```

To disable kube-proxy set the enabled field to false in the shoot manifest.

```yaml
apiVersion: core.gardener.cloud/v1beta1
kind: Shoot
metadata:
  name: ebpf-shoot
  namespace: garden-dev
spec:
  kubernetes:
    kubeProxy:
      enabled: false
```

### Know limitations of the EBPF Dataplane

Please note that the default settings for calico's ebpf dataplane may interfere with
[accelerated networking in azure](https://learn.microsoft.com/en-us/azure/virtual-network/accelerated-networking-overview)
rendering nodes with accelerated networking unusable in the network. The reason for this is that calico does not ignore
the accelerated networking interface `enP...` as it should, but applies its ebpf programs to it. A simple mitigation for
this is to adapt the `FelixConfiguration` `default` and ensure that the `bpfDataIfacePattern` does not include `enP...`.
Per default `bpfDataIfacePattern` is not set. The default value for this option can be found
[here](https://github.com/projectcalico/calico/blob/3f7fe4d290541bbdd73c97bdc89a29a29855a48a/felix/config/config_params.go#L180).
For example, you could apply the following change:

```
$ kubectl edit felixconfiguration default
...
apiVersion: crd.projectcalico.org/v1
kind: FelixConfiguration
metadata:
  ...
  name: default
  ...
spec:
  bpfDataIfacePattern: ^((en|wl|ww|sl|ib)[opsx].*|(eth|wlan|wwan).*|tunl0$|vxlan.calico$|wireguard.cali$|wg-v6.cali$)
  ...
```

## Example `NetworkingConfig` manifest

An example `NetworkingConfig` for the Calico extension looks as follows:

```yaml
apiVersion: calico.networking.extensions.gardener.cloud/v1alpha1
kind: NetworkConfig
ipam:
  type: host-local
  cidr: usePodCIDR
vethMTU: 1440
typha:
  enabled: true
overlay:
  enabled: true
```

## Example `Shoot` manifest

Please find below an example `Shoot` manifest with calico networking configratations:

```yaml
apiVersion: core.gardener.cloud/v1beta1
kind: Shoot
metadata:
  name: johndoe-azure
  namespace: garden-dev
spec:
  cloudProfileName: azure
  region: westeurope
  secretBindingName: core-azure
  provider:
    type: azure
    infrastructureConfig:
      apiVersion: azure.provider.extensions.gardener.cloud/v1alpha1
      kind: InfrastructureConfig
      networks:
        vnet:
          cidr: 10.250.0.0/16
        workers: 10.250.0.0/19
      zoned: true
    controlPlaneConfig:
      apiVersion: azure.provider.extensions.gardener.cloud/v1alpha1
      kind: ControlPlaneConfig
    workers:
    - name: worker-xoluy
      machine:
        type: Standard_D4_v3
      minimum: 2
      maximum: 2
      volume:
        size: 50Gi
        type: Standard_LRS
      zones:
      - "1"
      - "2"
  networking:
    type: calico
    nodes: 10.250.0.0/16
    providerConfig:
      apiVersion: calico.networking.extensions.gardener.cloud/v1alpha1
      kind: NetworkConfig
      ipam:
        type: host-local
      vethMTU: 1440
      overlay:
        enabled: true
      typha:
        enabled: false
  kubernetes:
    version: 1.24.3
  maintenance:
    autoUpdate:
      kubernetesVersion: true
      machineImageVersion: true
  addons:
    kubernetesDashboard:
      enabled: true
    nginxIngress:
      enabled: true
```