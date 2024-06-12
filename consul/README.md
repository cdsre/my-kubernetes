# Consul

I have previously used consul for on prem and cloud workloads and have found it to be a very useful tool for service 
discovery and configuration management. This is a collection of notes and manifests to run consul in kubernetes.

It is mostly based on https://developer.hashicorp.com/consul/tutorials/get-started-kubernetes. All the commands are 
written from the perspective of running them from this consul directory.

## Install

### pre-requisites
We will use the helm chart to install consul. We will use the values-v2.yaml file to configure the installation. In order
for the helm chart to work we need to ensure there is a default storage class in the cluster. We can check this by running
the following command.

```bash
kubectl get storageclass
```

Since I have been doing this on a local k8 cluster on a VPS I needed configure a local storage class. I used the following
manifest to create a local storage class.

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: local-storage
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
provisioner: kubernetes.io/no-provisioner
volumeBindingMode: WaitForFirstConsumer
```
I am using a default storage class with no provisioner and the helm chart will create the necessary persistent
volume claim and expect the storage class provisioner to create the volume. Since my storage class doesnt have a
provisioner the volume will not be created and the pod will not start. I will need to create the volume manually.

```bash
mkdir -p /mnt/local-storage
chmod 777 /mnt/local-storage
```
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: local-pv
spec:
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce
  storageClassName: local-storage
  local:
    path: /mnt/local-storage
  nodeAffinity:
    required:
      nodeSelectorTerms:
        - matchExpressions:
            - key: kubernetes.io/hostname
              operator: In
              values:
                - vps-b21d2eed
```

### helm
Now that we have the storage setup we can run the helm chart to deploy the consul resources

```bash
helm repo add hashicorp https://helm.releases.hashicorp.com
helm install --values helm/values-v2.yaml consul hashicorp/consul --create-namespace --namespace consul --version "1.2.0"
```
## Usage
Now everything is installed we should be able to look up the nodePort that the consul UI is running on and using the 
nodes IP with the node port we should be able to access the consul UI.

```bash
kubectl get service -n consul consul-ui -o=jsonpath='{.spec.ports[?(@.name=="https")].nodePort}{"\n"}';
```