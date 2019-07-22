This repo is a test
# Eirini Release

This is a `helm` release for Project [Eirini](https://code.cloudfoundry.org/eirini).

**NOTE**: This is an **_experimental_** release and is still considered _work in progress_.

## Prerequisites

* Make sure your Kubernetes cluster meets all [SCF related Kubernetes Requirements](https://github.com/SUSE/scf/wiki/How-to-Install-SCF#requirements-for-kubernetes).
* Install [Metrics server](https://github.com/kubernetes-incubator/metrics-server) in the system namespace
* Install [helm](https://helm.sh/)
* To be able to use the [bits service](https://github.com/cloudfoundry-incubator/bits-service) private registry in your Kubernetes cluster,
you need to have a signed TLS certificate, with a CA that the docker or containerd daemon on the nodes trust, and a CN that is pointing to the bits service.

**Note**: Eirini is currently being tested with HELM > 2.14.1, Kubernetes 1.13, and containerd as the container runtime.

### Minimum cluster requirements

We have validated that the deployment can start with a single-node 4 core 16GB cluster, although the initial startup will be very slow with this setup.
We recommend at least 8 cores and 16GB of RAM for the SCF control plane and at least two additional nodes that you can scale relative to the average consumption
of resource of the applications that you will be deploying. To make staging of applications faster with Diego, operators should scale the Diego cells to the number of additional nodes.

## Installation

1. Choose a [non NFS based `StorageClass`](https://github.com/SUSE/scf/wiki/How-to-Install-SCF#choosing-a-storage-class) because MySQL does not work well with it. For additional information you can take a look at [Storage Class](#storage-class)
1. Create a `values.yaml` based on [this](https://github.com/cloudfoundry-incubator/eirini-release/blob/master/values.yaml) template.
1. Make the Eirini helm repository available to helm:

    ```bash
    helm repo add eirini https://cloudfoundry-incubator.github.io/eirini-release
    ```

1. Make the Bits helm repository available to helm:

     ```bash
    helm repo add bits https://cloudfoundry-incubator.github.io/bits-service-release/helm
    ```
1. Install UAA:

    ```bash
    helm install eirini/uaa --namespace uaa --name uaa --values <your-values.yaml>
    ```

1. Export the UAA ca certificate using the following commands:

    ```bash
    SECRET=$(kubectl get pods --namespace uaa -o jsonpath='{.items[?(.metadata.name=="uaa-0")].spec.containers[?(.name=="uaa")].env[?(.name=="INTERNAL_CA_CERT")].valueFrom.secretKeyRef.name}')
    CA_CERT="$(kubectl get secret $SECRET --namespace uaa -o jsonpath="{.data['internal-ca-cert']}" | base64 --decode -)"
    ```

1. Export the Registry certificate in the `BITS_TLS_KEY` and `BITS_TLS_CRT` environment variables. (see [Certificates](#Certificates))

1. Install CF:

    ```bash
    helm install eirini/cf --namespace scf --name scf --values <your-values.yaml> --set "secrets.UAA_CA_CERT=${CA_CERT}" --set "eirini.secrets.BITS_TLS_KEY=${BITS_TLS_KEY}" --set "eirini.secrets.BITS_TLS_CRT=${BITS_TLS_CRT}" 
    ```

    **NOTICE**

    The above command will take the default value for `rootfs_version`. In case you want to specify a rootfs_version at deploy time use

    ```bash
    --set "global.rootfs_version=vx.x.x"
    ```

This will download the mentioned version of `eirinifs.tar`. (see [eirinifs releases](https://github.com/cloudfoundry-incubator/eirinifs/releases))
1. Use the following command to verify that every CF control plane pod is `running` and `ready`:

    ```bash
    watch kubectl get pods -n scf
    ```

## Notes

### Storage Class

It is highly recommended to use fast storage class for the blobstore. MySQL does
not work with NFS-based storage.

#### Using the HostPath Provisioner

As storage class, you can deploy a `hostpath` provisioner to your cluster. You can for example follow the documentation in this [repository](https://github.com/MaZderMind/hostpath-provisioner#dynamic-provisioning-of-kubernetes-hostpath-volumes). `hostpath` is not recommended for production use.

You can execute the following commands to have the `hostpath` provisioner installed in your Kubernetes cluster:

```bash
kubectl create -f https://raw.githubusercontent.com/MaZderMind/hostpath-provisioner/master/manifests/rbac.yaml
kubectl create -f https://raw.githubusercontent.com/MaZderMind/hostpath-provisioner/master/manifests/deployment.yaml
kubectl create -f https://raw.githubusercontent.com/MaZderMind/hostpath-provisioner/master/manifests/storageclass.yaml
```

#### Production Deployment

In a production settings ideally there should be existing storage classes that work with the deployment. In that case, you can either remove the `storage_class` properties from your `scf-config-values.yaml` file to use the default storage class, or alternatively set the properties to the storage class needed.

#### IBMCloud Kubernetes Service (IKS)

In IBM Kubernetes Service, it is recommended to use storage block storage class. See more how to enable it in [IBM Cloud documentation](https://console.bluemix.net/docs/containers/cs_storage_block.html#block_storage)

Additional details about deploying Eirini can be found in the `contrib` folder.

### Certificates

Please provide a serving certificate for bits service trusted by containerd/dockerd. In addition to usual globally trusted certificates, dockerd also supports self signed certificates. To know more about them please refer to [docker documentation](https://docs.docker.com/engine/security/certificates/).

However, containerd requires the signing authority for the registry certificate to be trusted OS wide. You could do this by getting a [Let's encrypt certificate](https://letsencrypt.org) or in IBMCloud Kubernetes Service, you could follow these instructions:

IKS provides ingress with a globally trusted certificate. The certificate is stored in a secret in the `default` namespace and has the same name as your cluster. You can use the following commands to export the certificates in the required environment variables:

```bash
BITS_TLS_CRT="$(kubectl get secret "$(kubectl config current-context)" --namespace default -o jsonpath="{.data['tls\.crt']}" | base64 --decode -)"
BITS_TLS_KEY="$(kubectl get secret "$(kubectl config current-context)" --namespace default -o jsonpath="{.data['tls\.key']}" | base64 --decode -)"
```

It is recommended to deploy Eirini with ingress and use that certificate in IKS.

### Service Account

When an app is pushed with Eirini, the pods are assigned the default Service Account in `opi.namespace`. By default, when the cluster is deployed with `RBAC` authentication method, that Service Account should not have any read/write permissions to the Kubernetes API. Since `RBAC` is preffered to `ABAC`, we recommend using the former.

### Network policies

Apps pushed by Eirini currently cannot be accessed directly from another app container. This is accomplished by creating a [NetworkPolicy](https://kubernetes.io/docs/concepts/services-networking/network-policies/) resource in the namespace in which Eirini deploys apps.

In order to use network policies in your cluster, you must use a compatible container network plug-in, otherwise creating a `NetworkPolicy` resource will have no effect.

Both [IKS](https://cloud.ibm.com/docs/containers?topic=containers-network_policies) (is automatically setup) and [GKE](https://cloud.google.com/kubernetes-engine/docs/how-to/network-policy#enabling_network_policy_enforcement) (has to be enabled) support a network plug-in called [Calico](https://www.projectcalico.org/), which supports defining network policies.

For other implementations of the Kubernetes networking model, take a look [here](https://kubernetes.io/docs/concepts/cluster-administration/networking/#how-to-implement-the-kubernetes-networking-model). Keep in mind that not all implementations support defining network polcies (e.g. Flannel). For a more detailed comparison between different plugins, take a look [here](https://docs.google.com/spreadsheets/d/1qCOlor16Wp5mHd6MQxB5gUEQILnijyDLIExEpqmee2k/edit#gid=0) (not maintained by us).

#### Securing SCF endpoints

It is not possible to do it with native Kubernetes network policies. In order to achieve this, the CNI plugin can be used directly. If you're using [Calico](https://www.projectcalico.org/) on IBMCloud, you can run the following command:

```bash
calicoctl apply --config $CALICOCNF -f - <<EOF
apiVersion: projectcalico.org/v3
kind: NetworkPolicy
metadata:
  name: deny-scf-access
  namespace: eirini
spec:
  types:
  - Egress
  egress:
  - action: Deny
    source:
      selector: source_type == 'APP'
    destination:
      namespaceSelector: name == 'scf'
  - action: Allow
EOF
```

You can use [this](https://www.ibm.com/cloud/blog/configure-calicoctl-for-ibm-cloud-kubernetes-service) guide to export `$CALICOCNF` on IBM Cloud. 

Note that GKE does not currently support creating custom Calico network policies.


## Resources

* [SCF documentation](https://github.com/SUSE/scf/wiki/How-to-Install-SCF#deploy-using-helm)
* [Eirini Continuous Integration Pipeline](https://ci.flintstone.cf.cloud.ibm.com/teams/eirini/pipelines/ci)
