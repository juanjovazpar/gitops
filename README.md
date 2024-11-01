# BAREMETAL K8S OAKDEW

## GitOps and ArgoCD

We will use ArgoCD to manage the deployment of the applications in the cluster with a GitOps approach.

The first step will be to install ArgoCD in the cluster with a Helm chart. An isolated namespace will be created for ArgoCD to avoid conflicts with other applications. This app will be in charge of deploying and synching the applications in the cluster later.

```
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update
helm install argocd argo/argo-cd --namespace argocd --create-namespace
```

## Persistent Storage

To create a persistent volume, we created a storageClass named `nfs-storage`. This StorageClass is used by all Persistent Volume Claims (PVCs) in the cluster, ensuring that any storage requests default to NFS storage.

All Persistent Volumes (PVs) are provisioned using the nfs-storage StorageClass. This allows for easy management of storage resources across the cluster.

#### NFS Node Configuration

The node designated for NFS storage is named `k8s-iris`. It serves as the centralized storage location for all data in the cluster. This setup provides a reliable and scalable storage solution, allowing multiple pods to access shared storage efficiently.

```
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: nfs-storage
  annotations:
    storageclass.kubernetes.io/is-default-class: 'true'
provisioner: cluster.local/nfs
reclaimPolicy: Retain
volumeBindingMode: Immediate
```

**A CEPH storage setup will be installed on the cluster, which will provide a shared storage pool for the pods instead of the NFS server.**

## DevOps Project

This project sets the main infrastructure for the development of the applications in the cluster. ArgoCD will be in charge to manage and to sync these applications into a isolated namespace `devops`.

### Porject Applications

#### 1. Cert-Manager

Cert-Manager is a Kubernetes add-on that automates the management and issuance of TLS certificates. It helps secure communication between services and ensures that all applications have valid certificates, thereby enhancing the overall security posture of our environment.

The main domain for the certificates will be `*.oakdew.biz`, which will be used to secure the devops applications in the cluster. A single certificate will be created for thr application in the project. The certificate will be issued by Let's Encrypt, which is a trusted CA that provides free SSL/TLS certificates and will be stored in a Kubernetes secret `oakdew-cert`.

#### 2. External-DNS

External-DNS automatically manages DNS records for Kubernetes services in external DNS providers. By integrating with DNS providers like `Cloudflare` (our case), it simplifies the management of DNS entries, allowing services to be easily accessible via user-friendly domain names.

External-DNS will be able to manage the DNS records in Clouflare by synchronizing the records with the Kubernetes ingress resources.

To accomplish this, External-DNS must have access to the CloudFlare API through a API token defined in the secret `cloudflare-api-key`:

```
apiVersion: v1
kind: Secret
metadata:
  name: cloudflare-api-key
  namespace: devops
type: Opaque
data:
  apiKey: <CLOUDFLARE_API_KEY>
```

Configure the API token:

| Cuenta | CloudFlare Tunnel | Leer   |
| ------ | ----------------- | ------ |
| Zonas  | DNS               | Editar |

#### 3. Cloudflared

Cloudflared is the official Cloudflare tunneling service that provides secure access to internal applications through Cloudflare. It enables the secure exposure of applications without opening additional ports, ensuring that they remain protected while being accessible from the internet.

A Cloudflare tunnel will be manually created with the following configurations, which will be used to expose the applications in the cluster.

Tunnel Name: **oakdewbiz-tunnel**

Public hostname:

| Subdomain | Domain     | Type  | URL                                                       |
| --------- | ---------- | ----- | --------------------------------------------------------- |
| \*        | oakdew.biz | HTTPS | **ingress-nginx-controller.devops.svc.cluster.local:443** |

Even though we do not define subdomain in this hostname, our tunnel will need DNS definitions for the subdomains. The tunnel will manage all of the traffic (`*`) while our instance of `external-DNS`will be in charge of defining DNS registries in CloudFlare dynamically. This public hostname must be set with `No TLS Verify: True` in the Cloudflare Tunnel settings when certificates in cluster are not used.

If you want to use the certificate in cluster, you must set `No TLS Verify: False` in the Cloudflare Tunnel settings and set the SSL/TLS
Mode to `Full (Sctrict)`

It is important to note that the tunnel will be redirected to the ingress-nginx-controller service in the devops namespace, which will be the entry point for the applications in the cluster.

When created, you must copy the tunnel ID and secret token, which will be used to configure the rest of the apps.

#### 4. NGINX Ingress Controller

Nginx is a high-performance web server that also acts as a reverse proxy and load balancer. In this project, it will be configured to serve as the entry point for our applications, handling HTTPS traffic and managing requests to various services within the cluster.

This ingress class will be configured as default in the cluster, which means that all services will be accessible through the ingress controller.

```
ingressClassResource: default: true
```

#### 5. Harbor `harbor.oakdew.biz`

Harbor is a cloud-native registry that stores, signs, and scans container images for vulnerabilities. It provides a secure way to manage Docker images, ensuring that only trusted images are deployed within our applications.

This registry will be used to store the container images for the applications in the cluster, images that will be consumed by ArgoCD to deploy the applications.

#### 6. Gitea `gitea.oakdew.biz`

Gitea is a lightweight self-hosted Git service that offers repository management, code review, and issue tracking. It provides a simplified interface for developers to collaborate on code while maintaining control over the source code management process.

This Git service will be used to store the source code for the applications in the cluster. This app will be also in charge of building the container images for the applications ans store them in the Harbor registry.

#### 7. Taiga `taiga.oakdew.biz`

Taiga is an agile project management tool that supports both Scrum and Kanban methodologies. It enables teams to manage their projects effectively, track progress, and collaborate efficiently, improving overall project delivery.
