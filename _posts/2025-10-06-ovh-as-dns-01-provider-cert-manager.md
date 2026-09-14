---
layout: post
title:  "Use OVH as a DNS-01 provider for cert-manager"
date:   2025-10-06 18:25:00 +0100
categories: Kubernetes Networking
tags: linux dns ovh
---

If you host your domains on OVH and you’re using **cert-manager** in Kubernetes (or any cluster) to automate Let’s Encrypt certificates, you can hook into OVH’s DNS API so that cert-manager can solve the DNS-01 challenge automatically.

## What you’ll end up with

You’ll have a **ClusterIssuer** (or Issuer) in cert-manager that uses a DNS-01 challenge via a **webhook** to OVH’s DNS API. Whenever cert-manager needs to prove domain ownership, it will create the necessary TXT records in your OVH DNS zone automatically.

## Prerequisites & assumptions

- You already have a Kubernetes cluster up and running.  
- You have admin rights (or enough rights) to install Helm charts and CRDs.  
- Your domain DNS is managed in OVH and you can generate API credentials (key/secret/consumer) in OVH.  
- Basic familiarity with cert-manager, Kubernetes Secrets, and RBAC.  

---

## Step 1: Install cert-manager

First things first: install cert-manager using Helm (or however you prefer). For example:

```bash
kubectl create namespace cert-manager

helm repo add jetstack https://charts.jetstack.io  
helm repo update  

helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --version v1.9.1 \
  --set installCRDs=true
```

## Step 2: Install the OVH webhook for cert-manager

The webhook is what translates cert-manager’s DNS-01 requests into OVH API calls to manage your DNS zones.

```bash
git clone https://github.com/baarde/cert-manager-webhook-ovh.git
cd cert-manager-webhook-ovh
helm install cert-manager-webhook-ovh ./deploy/cert-manager-webhook-ovh \
  --set groupName='acme.sernafa.com'
```

* The groupName is an identifier you’ll choose.
* You’re deploying in the same cluster where cert-manager lives.
* This webhook will act on behalf of cert-manager, creating and removing DNS TXT records in your OVH zone.

## Step 3: Create OVH API credentials & store them in Kubernetes

You need credentials so the webhook can authenticate with OVH’s API. Go to the OVH token creation page and create an API token with appropriate rights:

* GET /domain/zone/*
* PUT /domain/zone/*
* POST /domain/zone/*
* DELETE /domain/zone/*

Once you have:
* applicationKey
* applicationSecret
* consumerKey

Create a Kubernetes secret in the *cert-manager namespace*:

```bash
kubectl create secret generic ovh-credentials \
  --namespace cert-manager \
  --from-literal=applicationSecret='<OVHSECRET>'
```

We only store the secret here; the other values (applicationKey, consumerKey) will be referenced in the issuer manifest. 

You must also allow the webhook’s controller to read this secret. Create RBAC binding:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: cert-manager-webhook-ovh:secret-reader
rules:
  - apiGroups: [""]
    resources: ["secrets"]
    resourceNames: ["ovh-credentials"]
    verbs: ["get", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: cert-manager-webhook-ovh:secret-reader
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cert-manager-webhook-ovh:secret-reader
subjects:
  - apiGroup: ""
    kind: ServiceAccount
    name: cert-manager-webhook-ovh
    namespace: default
```

##Step 4: Create a ClusterIssuer (or Issuer)

Here’s a sample **ClusterIssuer** definition that uses the OVH DNS webhook for DNS-01:

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt
  namespace: cert-manager
spec:
  acme:
    # Use Let’s Encrypt production or staging endpoint
    server: https://acme-v02.api.letsencrypt.org/directory
    email: '<YOUR_EMAIL@example.com>'
    privateKeySecretRef:
      name: letsencrypt-account-key
    solvers:
      - dns01:
          webhook:
            groupName: 'acme.sernafa.com'
            solverName: ovh
            config:
              endpoint: ovh-eu
              applicationKey: '<APP_KEY>'
              applicationSecretRef:
                name: ovh-credentials
                key: applicationSecret
              consumerKey: '<CONSUMER_KEY>'
```

## Step 5: Request a certificate

Now that you have a working ClusterIssuer, create a Certificate resource referencing it. For example:

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: example-certificate
spec:
  dnsNames:
    - test.mydomain.com
  issuerRef:
    name: letsencrypt
    kind: ClusterIssuer
  secretName: test-mydomain-com-tls
```

## Troubleshooting & tips

* Use **kubectl describe certificate <name>** and **kubectl describe challenge <...>** to get clues if things fail.

