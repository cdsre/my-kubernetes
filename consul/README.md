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
nodes IP with the node port we should be able to access the consul UI. In my case I have setup nginx as a reverse proxy
to forward the traffic to the consul UI. I have also setup a DNS record to point to the nginx server so I can access the
consul UI using a domain name.

```bash
kubectl get service -n consul consul-ui -o=jsonpath='{.spec.ports[?(@.name=="https")].nodePort}{"\n"}';
```

We can also get the root token to access the consul UI. This will be stored in a k8 secret. We can also use this token
to access the consul CLI

```bash
kubectl get secret -n consul consul-bootstrap-acl-token -o jsonpath="{.data.token}" | base64 --decode
```

## Deploy sample app

The demo repo includes a sample app called hashicups. In V1 we deploy it and use nginx to act as the gateway to the front 
end. If we apply this and expose the nginx service via a proxy

```bash
kubectl apply -f hashicups/v1/
kubectl port-forward svc/nginx --namespace default 8080:80
```

when we try to connect we will get an error `RBAC: access denied`. This is because we have not setup the intentions yet 
in consul that state which services whithin the mesh are allowed to communicate with each other. We can do this by running
the following command

```bash
kubectl apply -f hashicups/intentions/allow.yaml
```

Now we should be able to access the hashicups app via the nginx proxy. This is ok but nginx its self is running as a 
service within the mesh, and we had to expose it via a port-forward to get access to it. In V2 we will change this so that
we now use the consul api-gateway to provide external access the hashicups app via nginx.

The API gateway provides a means to allow ingress to the services within the mesh. We will first deploy the gateway and 
then setup the routes and intentions to allow access to the hashicups app.

```bash
kubectl apply -f api-gw/consul-api-gateway.yaml --namespace consul && \
kubectl wait --for=condition=accepted gateway/api-gateway --namespace consul --timeout=90s && \
kubectl apply -f api-gw/routes.yaml --namespace consul && \
kubectl apply -f api-gw/intentions.yaml --namespace consul
```

In order for us to be able to route from the gateway to the services we need to create a reference grant that allows the
gateway to access the services in the default namespace for the purposes of routing

```bash
kubectl apply -f hashicups/v2/
```

we can then find its node port and access the hashicups app via the gateway

```bash
kubectl get service -n consul api-gateway -o=jsonpath='{.spec.ports[?(@.name=="https")].nodePort}{"\n"}'
```

## Observability
We can also setup observability for the consul cluster. We can use the prometheus and grafana helm charts to setup and
then based on the helm values in our helm v2 file we can configure the consul cluster to send metrics to the prometheus
endpoint and enrich the services view in consul.