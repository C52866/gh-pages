---
layout: "product"
title: "Local Installtion,,Pre-requisites,Installation,,Pre-requisites,Install,,Pre-requisites,Install"
tags:
- "Argo Workflows"
- " Local Installation"
header:
  title: null
  desc: null
sections: []
product: "argo_workflows"
permalink: "/infrastructure/argo_workflows/argo_installation_local"
toc: []

---
## Introduction

This repository contains code for Argo Workflows, Dex OpenID Connect Server, Postgres DB & GitHub SSO setup in Local K8S.

## Note
All tools installations (Argo + Dex + Postgres) & running Workflows uses same "argo" namespace, recommended to use different namespaces for tools installations & workflows runs separately.


## Pre-Installation

1. There is already a documentation available for Argo Workflows setup in local environment and this repository covers remaining setup process for DEX, GitHub SSO and Postgres DB.
   Follow the steps from [Base Installation](https://developer.acs.org/bitbucket/projects/DSGOPS/repos/argo-workflows-demo/browse) to setup local Argo + MinIO.

2. If GitHub SSO preferred to use, it is required to setup an Organization (Eg: argo-access-org/ACSPubsEDSG) & Teams (Eg: argo-readonly,argo-admin) in GitHub. Verify all references in configuration files provided in this repository & update them with created Organization & Team Names.

3. Create GitHub OAuth application in same GitHub Organization and update ClientId & ClientSecrets in all references in configuration files.

4. Before run Argo, verify Postgres, DEX services are accessible from your local machine. Please follow the steps from "Installation" section.

## Installation

### Postgres DB Setup
By default, Argo user kubernetes internal storage (EtcD) to store each workflow run status and logs will be maitained in Artifact repository.
Argo supports either Postgres/MySQL as external DB and it is also optional step. If you not recommended to use postgres DB skip this step and remove "persistence" configuration from workflow-controller-configmap.yaml file.

#### Installation
~~~bash
kubectl apply -f postgres-setup.yaml -n argo
~~~

### Dex Server Setup
SSO integration requires DEX server setup. Follow the below steps to configure and skip this installation if SSO not required.

Replace <github_oauth_client_id>,<github_oauth_client_secret> with actual GitHub OAuth Application secrets. Please follow the steps from link - [GitHub OAuth](https://docs.github.com/en/apps/oauth-apps/building-oauth-apps/creating-an-oauth-app) to create OAuth application in GitHub.

If Organization name in GitHub given other than 'argo-access-org', then replace it with new name from *.yaml files.

~~~bash
kubectl apply -f dex-setup.yaml
~~~

Run this command for DEX accessible to Argo Server otherwise Argo fails to start.
~~~bash
kubectl -n argo port-forward svc/dex 5556:5556
~~~

### SSO Integration
Argo supports multiple auth modes - client, server, sso. If you prefer to use SSO (GitHub), follow the instructions below else, skip this installation and remove "sso" configuration from workflow-controller-configmap.yaml file.

Replace <github_oauth_client_id>,<github_oauth_client_secret> with actual GitHub OAuth Application secrets. Please follow the steps from link - [GitHub OAuth](https://docs.github.com/en/apps/oauth-apps/building-oauth-apps/creating-an-oauth-app) to create OAuth application in GitHub.

If Organization name in GitHub given other than 'argo-access-org', then replace it with new name from *.yaml files.

In GitHub OAuth application, use - http://localhost:5556 as Home Page URL and http://localhost:5556/dex/callback as Call Back URL. Here, we are using DEX as middleware to connect with GitHub OIDC provider for Argo SSO integration.

#### Create SSO RBAC rules for Argo Workflows User Access:
~~~bash
kubectl apply -f sso-rbac-config.yaml
~~~

### Workflow Controller Configuration
Apply configurations to use SSO, DEX and Postgres, update existing ConfigMap.
~~~bash
kubectl apply -f workflow-controller-configmap.yaml -n argo
~~~

### Start Argo Server

#### Check Points
1. Verify DEX is running & accessible if SSO used in setup.
2. Verify Postgres is running & accessible if "persistence" enabled. 

To enable SSO, we must pass '--auth-mode sso' for argo server command or else remove and start Argo Server.
~~~bash
argo server --auth-mode sso --auth-mode client
~~~

If no authentication is required for Argo to use, run below command
~~~bash
argo server --auth-mode server
~~~

Access Argo from your browser - https://localhost:2746/ 

### References
1. [GitHub OAuth Application](https://docs.github.com/en/apps/oauth-apps/building-oauth-apps/creating-an-oauth-app)
2. [DEX Connectors](https://dexidp.io/docs/connectors/oidc/)
3. [DEX GitHub](https://dexidp.io/docs/connectors/github/)
4. [Argo DB](https://argoproj.github.io/argo-workflows/workflow-archive/)
5. [Argo SSO](https://argoproj.github.io/argo-workflows/argo-server-sso-argocd/)
6. [Argo ConfigMap](https://argoproj.github.io/argo-workflows/workflow-controller-configmap.yaml)
7. [Argo Artifacts](https://argoproj.github.io/argo-workflows/configure-artifact-repository/)

