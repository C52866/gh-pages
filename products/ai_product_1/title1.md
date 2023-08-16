---
layout: "product"
title: "Title1"
tags:
- "AI Tag"
header:
  title: null
  desc: null
sections: []
product: "ai_product_1"
permalink: "/ai/ai_product_1/page_1/"
toc: []

---
## Introduction
This repository contains Argo Workflows & Argo Events infrastructure setup in GKE and all possible customization parameters are exposed from `values.yaml` file and make sure all of them properly configured before starting installation.

## Note
1. Existing Ingress: Create DNS records for both DEX & Argo domains using in `values.yaml` to point to Ingress Object before installation 
2. New Ingress Installation: Create global IP & Create DNS records for both DEX & Argo domains to point to this IP
3. It installs all Argo resources in "default" namespace only and make sure running Helm installation from "default". It doesn't support other namespaces.

In both the cases, Ingress configuration will be done at last step by the Helm (Out of All resources used in this installation) and both DEX & Argo servers fails to start.
We should wait for sometime to work properly or else, delete DEX, Argo pods manually in case both servers are not accessible.

## Pre-Installation
Follow the below steps before Argo Workflows Installation in GKE. 

1. Create a static IP address (Name: argo-workflows-ip) either internal or external – This IP address used by the Ingress installation if Ingress creation is enabled. 

2. Create a GCP bucket for argo artificats and update `values.yaml` file to point to new bucket name.

3. Create a Cloud Armor Policy (Name: argo-workflows-policy) - To allow/deny IP addresses to access Argo Dashboard. It is required when you enable GKE Ingress installation.

4. Create GKE Cluster – Depends on your requirement create a GKE cluster.

## Installation
Installation part is separated into two step process.
1. Argo Resources Setup using Helm
2. Additional Resource Creation - Workflows to run in separate namespace instead of using Argo namespace.

### K8S Credentials Setup in Cloud Shell
Open Cloud Shell and run below command to setup k8s credentials. Update command to use right values of cluster & zone details.

gcloud container clusters get-credentials <cluster name> --zone <zone>

~~~bash
gcloud container clusters get-credentials argo-workflows-dev --zone us-central1-c
~~~

### Checkout Git Repository

~~~bash
git clone https://github.com/ACSPubsEDSG/at-argo-workflows.git
~~~

### Installation
Update values.yaml file with SSO, Persistence, Artifacts & Ingress configurations before start installation.

#### Updating Helm Dependencies
~~~bash
cd at-argo-workflows
cd gke
cd chart
helm dep update
~~~

#### Dry Run allows you to simulate the installation of a chart without actually creating any resources on the cluster.
~~~bash
helm install argo . --dry-run --debug
~~~

#### Installation
~~~bash
helm install argo . --debug #argo is helm release name
~~~

~~~text
NAME: argo
LAST DEPLOYED: Fri Jul  7 04:36:56 2023
NAMESPACE: default
STATUS: deployed
REVISION: 1
TEST SUITE: None
~~~

Verify argo resources using kubectl
~~~bash
kubectl get pods
~~~

~~~text
NAME                                                      READY   STATUS    RESTARTS   AGE
argo-argo-events-controller-manager-668445c79b-fh2jq      1/1     Running   0          45s
argo-argo-workflows-server-b7fbb557d-hnxfh                1/1     Running   0          45s
argo-argo-workflows-workflow-controller-57fbb5587-77cx6   1/1     Running   0          44s
~~~

Verify Ingress status. It will take some time to finish ingress setup.
~~~bash
kubectl describe ingress argo-workflows
~~~

~~~text
Name:             argo-workflows
Labels:           app.kubernetes.io/managed-by=Helm
Namespace:        default
Address:          34.36.74.16
Ingress Class:    <none>
Default backend:  argo-argo-workflows-server:2746 (10.60.2.4:2746)
Rules:
  Host        Path  Backends
  ----        ----  --------
  *           *     argo-argo-workflows-server:2746 (10.60.2.4:2746)
Annotations:  ingress.kubernetes.io/backends: {"k8s1-5c546a67-default-argo-argo-workflows-server-2746-917f40f9":"Unknown"}
              ingress.kubernetes.io/forwarding-rule: k8s2-fr-wqrsveq7-default-argo-workflows-cwzm50mh
              ingress.kubernetes.io/target-proxy: k8s2-tp-wqrsveq7-default-argo-workflows-cwzm50mh
              ingress.kubernetes.io/url-map: k8s2-um-wqrsveq7-default-argo-workflows-cwzm50mh
              kubernetes.io/ingress.class: gce
              kubernetes.io/ingress.global-static-ip-name: argo-workflows-dev
              meta.helm.sh/release-name: argo
              meta.helm.sh/release-namespace: default
Events:
  Type     Reason     Age                  From                     Message
  ----     ------     ----                 ----                     -------
  Warning  Translate  108s (x6 over 108s)  loadbalancer-controller  Translation failed: invalid ingress spec: error getting BackendConfig for port "&ServiceBackendPort{Name:,Number:2746,}" on service "default/argo-argo-workflows-server", err: no BackendConfig for service port exists.
  Normal   Sync       57s                  loadbalancer-controller  UrlMap "k8s2-um-wqrsveq7-default-argo-workflows-cwzm50mh" created
  Normal   Sync       54s                  loadbalancer-controller  TargetProxy "k8s2-tp-wqrsveq7-default-argo-workflows-cwzm50mh" created
  Normal   Sync       41s (x5 over 108s)   loadbalancer-controller  Scheduled for sync
  Normal   Sync       41s                  loadbalancer-controller  ForwardingRule "k8s2-fr-wqrsveq7-default-argo-workflows-cwzm50mh" created
  Normal   IPChanged  41s                  loadbalancer-controller  IP is now 34.36.74.16
~~~

#### Creating Additional Resources for Workflows
Run below command to create new namespace - "workflows", roles, rolebindings to maintain a separate namespace for Workflows to run.

~~~bash
kubectl apply -f argo-workflows-resources.yaml
~~~

### Create GitHub docker registry secret
It is optional step if you are not using any private docker registry for your images used in workflows.

~~~bash
kubectl create secret docker-registry docker-registry-github-com --docker-server=ghcr.io --docker-username=C52866 --docker-password=<GitHub PAT>  --docker-email=R_kumar@acs.org -n workflows
~~~
~~~text
Secret Name: docker-registry-github-com
Namespace: workflows
docker-password: <Use GitHub PAT/docker password>
docker-email: User your email
docker-username: Use docker username
~~~

### References
1. [Helm](https://helm.sh/)
2. [Argo Helm](https://github.com/argoproj/argo-helm/tree/main/charts/argo-workflows)
3. [GitHub OAuth Application](https://docs.github.com/en/apps/oauth-apps/building-oauth-apps/creating-an-oauth-app)
4. [DEX Connectors](https://dexidp.io/docs/connectors/oidc/)
5. [DEX GitHub](https://dexidp.io/docs/connectors/github/)
6. [Argo DB](https://argoproj.github.io/argo-workflows/workflow-archive/)
7. [Argo SSO](https://argoproj.github.io/argo-workflows/argo-server-sso-argocd/)
8. [Argo ConfigMap](https://argoproj.github.io/argo-workflows/workflow-controller-configmap.yaml)
9. [Argo Artifacts](https://argoproj.github.io/argo-workflows/configure-artifact-repository/)





