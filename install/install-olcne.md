# Installation on Oracle Linux Cloud Native Environment

This document describes installing Verrazzano in an [Oracle Linux Cloud Native Environment (OCLNE)](https://docs.oracle.com/en/operating-systems/olcne/) cluster.

Verrazzano requires Oracle Linux Cloud Native Environment version 1.1 or later.
> **NOTE**: You should only install this alpha release of Verrazzano in a cluster that can be safely deleted when your evaluation is complete.

## Requirements

### Deployment system requirements

The following software must be installed on your deployment system:
* curl
* helm
* jq
* kubectl
* openssl

If you are using Oracle Linux 7, then you can install these requirements with:
```
    sudo yum install -y oracle-olcne-release-el7
    sudo yum-config-manager --enable ol7_olcne11 ol7_addons ol7_latest
    sudo yum install -y kubectl helm jq openssl curl
```

### Cluster requirements
Deploy Oracle Linux Cloud Native Environment with the Kubernetes module following instructions from the [Getting Started](https://docs.oracle.com/en/operating-systems/olcne/start/deploy-kube.html) guide.

####  Prerequisites summary
A Verrazzano Oracle Linux Cloud Native Environment deployment requires:
* A default storage provider that supports "Multiple Read/Write" mounts. For example, an NFS service like:
    * Oracle Cloud Infrastructure File Storage Service.
    * A hardware-based storage system that provides NFS capabilities.
* Load balancers in front of the worker nodes in the cluster.
* DNS records that reference the load balancers.

Examples for meeting these requirements follow.

#### Prerequisites details

##### Storage
A default storage class is necessary. When using preallocated PersistentVolumes, for example Gluster/NFS,
they should be declared with a `storageClassName` as shown:
* Create a default `StorageClass`
  ```yaml
  cat << EOF | kubectl apply -f -
    apiVersion: storage.k8s.io/v1
    kind: StorageClass
    metadata:
      name: datacentre5-nfs
      annotations:
        storageclass.kubernetes.io/is-default-class: "true"
    provisioner: kubernetes.io/no-provisioner
    volumeBindingMode: WaitForFirstConsumer
  EOF
  ```
* Create a `PersistentVolume`
  ```yaml
  cat << EOF | kubectl apply -f -
    apiVersion: v1
    kind: PersistentVolume
    metadata:
      name: pv0001
    spec:
      storageClassName: datacentre5-nfs
      accessModes:
        - ReadWriteOnce
        - ReadWriteMany
      capacity:
        storage: 50Gi
      nfs:
        path: /datapool/k8s/pv0001
        server: 10.10.10.10
      volumeMode: Filesystem
  EOF
  ```

#### Networking

Verrazzano on Oracle Linux Cloud Native Environment will use your own external load balancer services, not those dynamically provided by Kubernetes.
Prior to installation two load balancers should be deployed outside of Kubernetes, one for management traffic and one
for general traffic.

Specific steps will differ for each load balancer provider, but a generic configuration and an example follow.

##### Configuration
* Target Host: Hostnames of Kubernetes worker nodes
* Target Port: See table
* Distribution: Round Robin
* Health Check: TCP

Traffic Type | Service Name | Target Port | Type | Suggested External Port
 ---|---|---|---|---
General | `istio-ingressgateway` | 31380 | TCP | 80
General | `istio-ingressgateway` | 31390 | TCP | 443
Management | `ingress-controller-nginx-ingress-controller` | 30080 | TCP | 80
Management | `ingress-controller-nginx-ingress-controller` | 30443 | TCP | 443

##### Configuration example
An NGINX example of the general traffic load balancer:
```
load_module /usr/lib64/nginx/modules/ngx_stream_module.so;
events {
  worker_connections 2048;
}
stream {
  upstream backend_http {
    server worker-0.example.com:31380;
    server worker-1.example.com:31380;
    server worker-2.example.com:31380;
  }
  upstream backend_https {
    server worker-0.example.com:31390;
    server worker-1.example.com:31390;
    server worker-2.example.com:31390;
  }
  server {
    listen 80;
    proxy_pass backend_http;
  }
  server {
    listen 443;
    proxy_pass backend_https;
  }
}
```
An NGINX example of the management load balancer:
```
load_module /usr/lib64/nginx/modules/ngx_stream_module.so;
events {
  worker_connections 2048;
}
stream {
  upstream backend_http {
    server worker-0.example.com:30080;
    server worker-1.example.com:30080;
    server worker-2.example.com:30080;
  }
  upstream backend_https {
    server worker-0.example.com:30443;
    server worker-1.example.com:30443;
    server worker-2.example.com:30443;
  }
  server {
    listen 80;
    proxy_pass backend_http;
  }
  server {
    listen 443;
    proxy_pass backend_https;
  }
}
```

##### Manual DNS type
When using the `manual` DNS type, the installer searches the DNS zone you provide for two specific A records.
These are used to configure the cluster and should refer to external addresses of the load balancers in the previous step.

Record | Use
---|---
ingress-mgmt | Set as the `.spec.externalIPs` value of the `ingress-controller-nginx-ingress-controller` service
ingress-verrazzano | Set as the `.spec.externalIPs` value of the `istio-ingressgateway` service

For example:
```
     198.51.100.10                                   A       ingress-mgmt.myenv.mydomain.com.
     203.0.113.10                                    A       ingress-verrazzano.myenv.mydomain.com.
```

Installation will result in a number of management services that need to point to the ingress-mgmt address.
```
    keycloak.myenv.mydomain.com                      CNAME   ingress-mgmt.myenv.mydomain.com.
    rancher.myenv.mydomain.com                       CNAME   ingress-mgmt.myenv.mydomain.com.

    grafana.vmi.system.myenv.mydomain.com            CNAME   ingress-mgmt.myenv.mydomain.com.
    prometheus.vmi.system.myenv.mydomain.com         CNAME   ingress-mgmt.myenv.mydomain.com.
    kibana.vmi.system.myenv.mydomain.com             CNAME   ingress-mgmt.myenv.mydomain.com.
    elasticsearch.vmi.system.myenv.mydomain.com      CNAME   ingress-mgmt.myenv.mydomain.com.
```

Deployment of applications as a VerrazzanoBinding will create four more services in the form:
* grafana.vmi.**binding-name**.myenv.mydomain.com
* prometheus.vmi.**binding-name**.myenv.mydomain.com
* kibana.vmi.**binding-name**.myenv.mydomain.com
* elasticsearch.vmi.**binding-name**.myenv.mydomain.com

For simplicity, an administrator may want to create [wildcard DNS records](https://tools.ietf.org/html/rfc1034#section-4.3.3) for the management addresses:
```
    *.system.myenv.mydomain.com                      CNAME   ingress-mgmt.myenv.mydomain.com.
    *.mybinding.myenv.mydomain.com                   CNAME   ingress-mgmt.myenv.mydomain.com.
```
or
```
    *.myenv.mydomain.com                             CNAME   ingress-mgmt.myenv.mydomain.com.
```


**NOTE:** At this time, the only supported deployment for OLCNE is the manual DNS type.

## Installation

### 1. Configure the environment

Set the following `ENV` vars:
  ```
    export CLUSTER_TYPE=OLCNE
    export VERRAZZANO_KUBECONFIG=<path to valid kubernetes config>
    export KUBECONFIG=$VERRAZZANO_KUBECONFIG
  ```

### 2. Create the Oracle Container Registry secret
You need to create an "ocr" secret for pulling images from the container-registry.oracle.com repository.
```
   kubectl create secret docker-registry ocr \
       --docker-username=<username> \
       --docker-password=<password> \
       --docker-server=container-registry.oracle.com
```

### 3. Install Verrazzano

You will install Verrazzano using the `manual` DNS type.

Run the following scripts in order:
```
   ./1-install-istio.sh                       -d manual -n <env-name> -s <dns-suffix>
   ./2a-install-system-components-magicdns.sh -d manual -n <env-name> -s <dns-suffix>
   ./3-install-verrazzano.sh                  -d manual -n <env-name> -s <dns-suffix>
   ./4-install-keycloak.sh                    -d manual -n <env-name> -s <dns-suffix>
```

### 4. Verify installation
Verrazzano installs multiple objects in multiple namespaces.  All the pods in the `verrazzano-system` namespaces in the `Running` status does not guarantee but likely indicates that Verrazzano is up and running.
```
kubectl get pods -n verrazzano-system
verrazzano-admission-controller-84d6bc647c-7b8tl   1/1     Running   0          5m13s
verrazzano-cluster-operator-57fb95fc99-kqjll       1/1     Running   0          5m13s
verrazzano-monitoring-operator-7cb5947f4c-x9kfc    1/1     Running   0          5m13s
verrazzano-operator-b6d95b4c4-sxprv                1/1     Running   0          5m13s
vmi-system-api-7c8654dc76-2bdll                    1/1     Running   0          4m44s
vmi-system-es-data-0-6679cf99f4-9p25f              2/2     Running   0          4m44s
vmi-system-es-data-1-8588867569-zlwwx              2/2     Running   0          4m44s
vmi-system-es-ingest-78f6dfddfc-2v5nc              1/1     Running   0          4m44s
vmi-system-es-master-0                             1/1     Running   0          4m44s
vmi-system-es-master-1                             1/1     Running   0          4m44s
vmi-system-es-master-2                             1/1     Running   0          4m44s
vmi-system-grafana-5f7bc8b676-xx49f                1/1     Running   0          4m44s
vmi-system-kibana-649466fcf8-4n8ct                 1/1     Running   0          4m44s
vmi-system-prometheus-0-7f97ff97dc-gfclv           3/3     Running   0          4m44s
vmi-system-prometheus-gw-7cb9df774-48g4b           1/1     Running   0          4m44s
```

### 5. Get the console URLs
Verrazzano installs several consoles.  You can get the ingress for the consoles with the following command:  

`kubectl get ingress -A`

Simply prefix `https://` to the host name to get the URL.  For example `https://rancher.myenv.mydomain.com`

Following is an example of the ingresses:
```
   NAMESPACE           NAME                               HOSTS                                          ADDRESS          PORTS     AGE
   cattle-system       rancher                            rancher.myenv.mydomain.com                     128.234.33.198   80, 443   93m
   keycloak            keycloak                           keycloak.myenv.mydomain.com                    128.234.33.198   80, 443   69m
   verrazzano-system   verrazzano-operator-ingress        api.myenv.mydomain.com                         128.234.33.198   80, 443   81m
   verrazzano-system   vmi-system-api                     api.vmi.system.myenv.mydomain.com              128.234.33.198   80, 443   80m
   verrazzano-system   vmi-system-es-ingest               elasticsearch.vmi.system.myenv.mydomain.com    128.234.33.198   80, 443   80m
   verrazzano-system   vmi-system-grafana                 grafana.vmi.system.myenv.mydomain.com          128.234.33.198   80, 443   80m
   verrazzano-system   vmi-system-kibana                  kibana.vmi.system.myenv.mydomain.com           128.234.33.198   80, 443   80m
   verrazzano-system   vmi-system-prometheus              prometheus.vmi.system.myenv.mydomain.com       128.234.33.198   80, 443   80m
   verrazzano-system   vmi-system-prometheus-gw           prometheus-gw.vmi.system.myenv.mydomain.com    128.234.33.198   80, 443   80m
```

### 6. Get console credentials
You will need the credentials to access the various consoles installed by Verrazzano.

#### Consoles accessed by the same user name/password
- Grafana
- Prometheus
- Kibana
- Elasticsearch

**User:**  `verrazzano`

Run the following command to get the password:

`kubectl get secret --namespace verrazzano-system verrazzano -o jsonpath={.data.password} | base64 --decode; echo`

#### The Keycloak admin console
**User:** `keycloakadmin`

Run the following command to get the password:

`kubectl get secret --namespace keycloak keycloak-http -o jsonpath={.data.password} | base64 --decode; echo`

#### The Rancher console
**User:** `admin`

Run the following command to get the password:

`kubectl get secret --namespace cattle-system rancher-admin-secret -o jsonpath={.data.password} | base64 --decode; echo`


### 7. (Optional) Install the example applications
Example applications are located in the `examples` directory.
