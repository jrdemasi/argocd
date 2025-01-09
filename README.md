# argocd
This repo aims to capture the way I run my personal kubernetes cluster.  I use managed kubernetes from Vultr, as at the time of deployment Linode and DigitalOcean both lacked dual stack support.  That being said, most of my infrastructure IS on Linode, which is why you'll see my external-dns looking there to create records.  Additionally, as neither Linode nor Vultr offers secrets management, I'm relying on AWS Secrets Manager to plumb external-secrets.  In the future, I'd consider moving to Bitwarden's secrets manager if it has better support - today, there isn't an equivalent of a `ClusterSecretStore` which would make a personal setup, where security is important but perhaps there is some liberty, painful.  

If not obvious, everything in this repo is installed using helm and extended using templates.  While I am actually partial to Kustomize, my day job uses Helm and it's a skill I wanted to learn.  The app-of-apps pattern I follow mirrors the argocd example repo exactly - thare are other ways to do app-of-apps, and there are also applicationsets, which may be a better fit for your use case.  For a personal cluster, I felt like this was a reasonable path forward.  

The final note here - running a kubernetes cluster isn't for the faint of heart.  I think it's paramount that you have networking, Linux, containerization, and some security fundamentals before you embark on ANY kubernetes journey, but especially managed kubernetes that is inherently on the Internet.  My plan is to only run services I'd be exposing anyway even if they were running on a VPS, and ultimately I will move towards locking access down more using a VPN or some kind of mesh.  Use anything and everything in this repo entirely at your own risk, and know what you are getting yourself into.  <3

## Bootstrapping a Cluster
Spin up a cluster (I'll assume in Vultr with the Vultr Kubernetes Engine) however you want.  This can be done manually, using terraform, or using IaC of your choice.  The initial cluster needs to have exactly two things prior to adding the app-of-apps from this repository:
1. The cluster needs to have argocd helm installed from this repo itself.
2. The cluster needs initial secrets seeded for external-secrest to communicate with AWS.

The way that I'm bootstrapping a cluster is detailed below, and assumes you've already downloaded and referenced the VKE config file from Vultr.  I'm also assuming you've got helm installed locally :-) 

```
git clone git@github.com:jrdemasi/argocd # this repo
cd argocd/argocd # the argocd app within this repo
helm dependency build
helm install argocd . --namespace=argocd --create-namespace # this installs argo the exact same way that it will ultimately manage itself, so nothing should happen once you add the app-of-apps!
kubectl create secret generic awssm-secret --from-file=./access-key --from-file=./secret-access-key # these files have to exist with aws access key and secret access key
```

At this point, you've got a cluster with argocd already installed and you've seeded the initial external-secrets AWS creds.  The next step is to add the argocd repository (the one you're looking at right now, maybe) to your argocd instance.  You can do this using the argocd UI, or using kube manifests.  If you want to use the ArgoCD UI, you can get the initial admin password and port-forward the application and access it at https://localhost:8080 - you will have to accept a self-signed certificate that argocd ships with:

```
kubectl -n argocd get secrets argocd-initial-admin-secret \
    -o jsonpath='{.data.password}' | base64 -d
kubectl -n argocd port-forward service/argocd-server 8080:80
```

Personally, this is the only way I access the UI.  Once my cluster is setup, I don't use it a whole lot.  Obviously you can throw a load balancer or ingress controller, some certs, and DNS in front of the argocd service once your cluster is ready to go if you want.  For adding the repo and apps initially, I use kube manifests.  These could honestly be added as templates in the argocd helm chart in the repo, which is a future feature enhancement I just hadn't thought of initially.  No secrets are necessary since the repo is public: 

```
apiVersion: v1
kind: Secret
metadata:
  name: this-repo
  namespace: argocd
  labels:
    argocd.argoproj.io/secret-type: repository
stringData:
  type: git
  url: https://github.com/jrdemasi/argocd
---
project: default
source:
  repoURL: https://github.com/jrdemasi/argocd
  path: apps
  targetRevision: HEAD
destination:
  server: https://kubernetes.default.svc
  namespace: argocd
syncPolicy:
  automated:
    prune: true
    selfHeal: true
  syncOptions:
    - CreateNamespace=true
  retry:
    limit: 2
    backoff:
      duration: 5s
      factor: 2
      maxDuration: 3m0s
```

Once these are added, sit back and wait for everything to sync the first time.  Usually ingress-nginx takes the longest since it has to wait for a load balancer to spin up.