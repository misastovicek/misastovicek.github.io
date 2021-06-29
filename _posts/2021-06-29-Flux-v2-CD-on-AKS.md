---
layout: posts
title: Flux v2 CD on Azure Kubernetes Service
author: Michal Stovicek
---

This time, I'd like to introduce you to Flux v2 on Azure Kubernetes Service (AKS). Flux v1 has been around for some time and it is pretty neat tool. With version two (currently beta), Flux is actually set of Kubernetes operators refered to as a *GitOps toolkit* and simple command line utility - *flux*. What I found awesome about Flux v2 is the fact that you can bootstrap it's configuration in a GitOps way. And what does that mean? Well, you get the flux binary and run `flux bootstrap github` with couple of parameters. The `bootstrap` command supports not only `github`, but also `gitlab` or generic `git`. The command will push all the necesary configurations to your git repository, deploys the Kubernetes operators and creates CRDs used to configure those operators. It means that once the command is finished, you can control your operators straight from Github/Gitlab or whatever git system you are using. Just a single command!

Anyway, the installation is only one part of it. Flux can also watch your container registry (ACR on Azure) and update your apps whenever a new image is pushed to the registry. It can also utilize *Mozilla SOPS* so that you can securely store your secrets in git and SOPS can leverage Azure KeyVault Keys for data encryption/decryption. SOPS is a nice and easy way of handling secrets in YAML files and is also supported by many other tools such as [Terragrunt](https://terragrunt.gruntwork.io/). I'll show this in action leveraging [Azure AD Pod Identity](https://azure.github.io/aad-pod-identity/docs/) later in this post.

Flux v2 supports Helm and Kustomize and so if you are using plain kubernetes manifests, you will have to use Kustomize with them. This is pretty easy to use though. You can just run `kustomize create --autodetect` and you are good to go as it will create a `kustomization.yaml` with all the Kubernetes manifests included for you. Honestly, I found Kustomize to be quite useful. You can easily have a base manifests and configure apps based on environment/location/etc. using Kustomize. With Flux, I even use Kustomize with Helm releases. Some of your values may be common for all the environments, but some may be specific. In such case Kustomize shines.

>**Note:** You can use a [Kustomize installation script](https://kubectl.docs.kubernetes.io/installation/kustomize/binaries/) or go to [Github release page](https://github.com/kubernetes-sigs/kustomize/releases).

Alright, enough talking, let's see this in action. I am going to deploy some Helm Charts and also plain Kubernetes manifests. Flux will observer my Azure Container Registry for new container image and will push changes to my Github repo (into the main branch for simplicity). I am going to use AAD Pod Identity and Azure Key Vault as well so that SOPS can be used to decrypt the secrets.

First of all let's create couple of Azure resources. I need a VNet with a subnet for my AKS cluster, the AKS cluster itself, Azure Container Registry and also an Azure KeyVault. All of it can be deployed with [Azure Bicep template stored in my github repo](https://github.com/misastovicek/flux-v2-example/tree/main/bicep) which also contains example files from this post. I am going to use that tempate and will provide the `aks.parameters.json` file to fill some of the parameters (others have default values):

```powershell
PS> Get-Content -Path .\bicep\aks.parameters.json
{
    "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentParameters.json#",
    "contentVersion": "1.0.0.0",
    "parameters": {
        "name": {
            "value": "mstov"
        },
        "publicSSHKey": {
            "value": "<PUT YOUR PUBLIC SSH KEY HERE>"
        }
    }
}

PS> New-AzDeployment `
    -TemplateFile .\bicep\main.bicep `
    -TemplateParameterFile .\bicep\aks.parameters.json `
    -Location northeurope -Name ExampleAks

DeploymentName          : ExampleAks
ResourceGroupName       : rg-mstov-aks
ProvisioningState       : Succeeded
Timestamp               : 6/18/2021 12:51:07 PM
Mode                    : Incremental
TemplateLink            :
Parameters              :
                          Name                Type                       Value
                          ==================  =========================  ==========
                          location            String                     northeurope
                          name                String                     mstov
                          cidrBlock           String                     172.16.0.0/23
                          dockerBridgeCidr    String                     172.20.0.1/16
                          dockerDnsIp         String                     172.21.0.10
                          serviceCidr         String                     172.21.0.0/16
                          aksNodecount        Int                        1
                          aksNodeSize         String                     Standard_B2ms
                          publicSSHKey        String                     <REDACTED>

Outputs                 :
                          Name                  Type                       Value
                          ====================  =========================  ==========
                          keyVaultSopsKeyUri    String                     https://kv-mstov-test.vault.azure.net/keys/sops/5de841e201d14a62a04ffbb4d0d03fad

DeploymentDebugLogLevel :

```

The output above shows successful deployment and as you can see, there is a URI to the Azure KeyVault secret which will be used later with SOPS.

Now when the infrastructure is ready, let's install *Flux v2*. First of all I need to obtain the new AKS cluster credentials, flux CLI app and then I can bootstrap the flux CD with it's Github integration (I am going to use Ubuntu in WSL v2 for the rest of this post):

```bash
az aks get-credentials --admin -n aks-mstov -g rg-mstov-aks

curl -LO https://github.com/fluxcd/flux2/releases/download/v0.15.1/flux_0.15.1_linux_amd64.tar.gz
tar xzf flux_0.15.1_linux_amd64.tar.gz
sudo mv ./flux /usr/bin/

export GITHUB_TOKEN=<REDACTED>
flux bootstrap github \
  --owner=misastovicek \
  --repository=flux-v2-example \
  --path=clusters/ \
  --personal \
  --read-write-key \
  --components-extra image-reflector-controller,image-automation-controller \
  --network-policy=false
```

>**Note:** I have set `--network-policy` to `false` because there is an issue specificaly related to Azure Network Policies. It works with Calico though. Also note that I am provisioning *ReadWrite Github Deployment Key* as I want Flux to be able to push changes.

The commands above will result in deployment of the Flux v2 CRD components to the cluster and Flux YAML manifests and kustomization file being committed to the Github repo - path `clusters/flux-system/`.

```bash
$ kubectl get pod,crd -n flux-system

NAME                                              READY   STATUS    RESTARTS   AGE
pod/helm-controller-69d8f57b97-82l62              1/1     Running   2          4d3h
pod/image-automation-controller-5b58fc848-8qxxf   1/1     Running   0          6m39s
pod/image-reflector-controller-7c54cf9b99-jjsnb   1/1     Running   0          6m39s
pod/kustomize-controller-84b7b4fd4b-45t45         1/1     Running   1          4d3h
pod/notification-controller-757876869-4rqt7       1/1     Running   1          4d3h
pod/source-controller-78c456cbc7-2b4qs            1/1     Running   0          4m30s

NAME                                                                                             CREATED AT
customresourcedefinition.apiextensions.k8s.io/alerts.notification.toolkit.fluxcd.io              2021-06-21T08:13:02Z
customresourcedefinition.apiextensions.k8s.io/azureassignedidentities.aadpodidentity.k8s.io      2021-06-21T08:41:14Z
customresourcedefinition.apiextensions.k8s.io/azureidentities.aadpodidentity.k8s.io              2021-06-21T08:41:14Z
customresourcedefinition.apiextensions.k8s.io/azureidentitybindings.aadpodidentity.k8s.io        2021-06-21T08:41:14Z
customresourcedefinition.apiextensions.k8s.io/azurepodidentityexceptions.aadpodidentity.k8s.io   2021-06-21T08:41:14Z
customresourcedefinition.apiextensions.k8s.io/buckets.source.toolkit.fluxcd.io                   2021-06-21T08:13:02Z
customresourcedefinition.apiextensions.k8s.io/gitrepositories.source.toolkit.fluxcd.io           2021-06-21T08:13:03Z
customresourcedefinition.apiextensions.k8s.io/helmcharts.source.toolkit.fluxcd.io                2021-06-21T08:13:03Z
customresourcedefinition.apiextensions.k8s.io/helmreleases.helm.toolkit.fluxcd.io                2021-06-21T08:13:03Z
customresourcedefinition.apiextensions.k8s.io/helmrepositories.source.toolkit.fluxcd.io          2021-06-21T08:13:03Z
customresourcedefinition.apiextensions.k8s.io/imagepolicies.image.toolkit.fluxcd.io              2021-06-25T12:50:32Z
customresourcedefinition.apiextensions.k8s.io/imagerepositories.image.toolkit.fluxcd.io          2021-06-25T12:50:32Z
customresourcedefinition.apiextensions.k8s.io/imageupdateautomations.image.toolkit.fluxcd.io     2021-06-25T12:50:32Z
customresourcedefinition.apiextensions.k8s.io/kustomizations.kustomize.toolkit.fluxcd.io         2021-06-21T08:13:03Z
customresourcedefinition.apiextensions.k8s.io/providers.notification.toolkit.fluxcd.io           2021-06-21T08:13:03Z
customresourcedefinition.apiextensions.k8s.io/receivers.notification.toolkit.fluxcd.io           2021-06-21T08:13:03Z
```

As you can see, there are some controllers running and also some CRDs. Now the flux will reconcile itself with any change made to it via git. You will see that later once the Azure Pod Identity Binding is deployed as I want Flux to use an Azure MSI to access the Azure Key Vault and be able to decrypt the SOPS encrypted files. So let's prepare the AAD Pod Identity Binding - Helm Chart deployment - using Flux:

```bash
git clone https://github.com/misastovicek/flux-v2-example.git
cd flux-v2-example
mkdir -p apps/test/aad-pod-identity
mkdir -p clusters/test
mkdir sources
cat >sources/aad-pod-identity-helm.yaml<<-EOF
---
apiVersion: source.toolkit.fluxcd.io/v1beta1
kind: HelmRepository
metadata:
  name: aad-pod-identity
  namespace: flux-system
spec:
  interval: 30m
  url: https://raw.githubusercontent.com/Azure/aad-pod-identity/master/charts
EOF

cat >sources/kustomization.yaml<<-EOF
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - aad-pod-identity-helm.yaml
EOF

cat >clusters/test/apps.yaml<<-EOF
apiVersion: kustomize.toolkit.fluxcd.io/v1beta1
kind: Kustomization
metadata:
  name: apps
  namespace: flux-system
spec:
  interval: 10m0s
  sourceRef:
    kind: GitRepository
    name: flux-system
  path: ./apps/test/
  prune: true
  validation: client
  decryption:
    provider: sops
EOF

cat >apps/test/kustomization.yaml<<-EOF
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - aad-pod-identity/
  - ../../sources/
EOF

cat >apps/test/aad-pod-identity/aad-pod-identity.yaml<<-EOF
---
apiVersion: v1
kind: Namespace
metadata:
  name: aad-pod-identity
---
apiVersion: helm.toolkit.fluxcd.io/v2beta1
kind: HelmRelease
metadata:
  name: aad-pod-identity
  namespace: aad-pod-identity
spec:
  releaseName: aadpodidentity
  targetNamespace: aad-pod-identity
  interval: 10m
  chart:
    spec:
      chart: aad-pod-identity
      sourceRef:
        kind: HelmRepository
        name: aad-pod-identity
        namespace: flux-system
  values:
    azureIdentities:
      kv-access:
        namespace: flux-system
        type: 0
        resourceID: "/subscriptions/xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx/resourceGroups/rg-mstov-aks/providers/Microsoft.ManagedIdentity/userAssignedIdentities/flux-cd"
        clientID: "xxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
        binding:
          name: kv-access
          selector: kv-access
EOF

cat >apps/test/aad-pod-identity/kustomization.yaml<<-EOF
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - aad-pod-identity.yaml
EOF

kubectl apply -f clusters/test/apps.yaml
```

Let's check the Flux HelmReleases CRD:

```bash
$ flux get helmreleases --all-namespaces

NAMESPACE               NAME                    READY   MESSAGE                                 REVISION        SUSPENDED
aad-pod-identity        aad-pod-identity        True    Release reconciliation succeeded        4.1.1           False
```

And here we go! The AAD Pod Identity is deployed and reconciled and so I can go ahead and update `kustomization.yaml` file for the *Flux CD* components to allow flux to utilize the AAD Pod Identity Binding. The Kustomization will just add a label to all the Kubernetes *deployments* in the *flux-system* namespace.

```bash
cat >>clusters/flux-system/kustomization.yaml<<-EOF
patches:
- path: ./patch-aad-pod-binding-label.yaml
  target:
    kind: Deployment
EOF

cat >clusters/flux-system/patch-aad-pod-binding-label.yaml<<-EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  # The name is not important since patch in kustomization.yaml is applied to all Deployments
  name: unknown
spec:
  template:
    metadata:
      labels:
        aadpodidbinding: kv-access
    spec:
      containers:
      - name: manager
        env:
        - name: AZURE_AUTH_METHOD
          value: msi
EOF

git add -A && git commit -m "Flux AAD Pod Identity label added" && git push
```

You should now see the pods labeled as `aadpodidbinding=kv-access`. Here is a sample:

```bash
kubectl get pod -n flux-system --show-labels
NAME                                      READY   STATUS    RESTARTS   AGE   LABELS
helm-controller-69d8f57b97-82l62          1/1     Running   0          23m   aadpodidbinding=kv-access,app=helm-controller,pod-template-hash=69d8f57b97
kustomize-controller-84b7b4fd4b-45t45     1/1     Running   0          23m   aadpodidbinding=kv-access,app=kustomize-controller,pod-template-hash=84b7b4fd4b
notification-controller-757876869-4rqt7   1/1     Running   0          23m   aadpodidbinding=kv-access,app=notification-controller,pod-template-hash=757876869
source-controller-78c456cbc7-lvnlj        1/1     Running   0          23m   aadpodidbinding=kv-access,app=source-controller,pod-template-hash=78c456cbc7
```

So far I have the Flux CD and Azure AD Pod Identity Binding ready. Let's put another app into the game - *NGINX ingress controller* available at [Artifact Hub](https://artifacthub.io/packages/helm/bitnami/nginx-ingress-controller). I am going to prepare the Helm release for the NGINX Ingress Controller. You may see that I also need to update the `kustomization.yaml` file in `apps/test/` and also create a new `kustomization.yaml` file in the `apps/test/ingress-nginx` directory so that Kustomize knows what to deploy.

```bash
# Create source for the Nginx Ingress controller helm chart (I use Bitnami's repo)
cat >sources/bitnami.yaml<<-EOF
---
apiVersion: source.toolkit.fluxcd.io/v1beta1
kind: HelmRepository
metadata:
  name: bitnami
  namespace: flux-system
spec:
  interval: 30m
  url: https://charts.bitnami.com/bitnami
EOF

# Add records to the kustomization yamls
echo "  - bitnami.yaml" >> sources/kustomization.yaml
echo "  - ingress-nginx/" >> apps/test/kustomization.yaml
cat >apps/test/ingress-nginx/kustomization.yaml<<-EOF
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - ingress-nginx.yaml
EOF

# Add Helm release for the Nginx Ingress Controller
cat >apps/test/ingress-nginx/ingress-nginx.yaml<<-EOF
---
apiVersion: v1
kind: Namespace
metadata:
  name: ingress-nginx
---
apiVersion: helm.toolkit.fluxcd.io/v2beta1
kind: HelmRelease
metadata:
  name: ingress-nginx
  namespace: ingress-nginx
spec:
  releaseName: ingress
  targetNamespace: ingress-nginx
  interval: 10m
  chart:
    spec:
      chart: ingress-nginx
      sourceRef:
        kind: HelmRepository
        name: ingress-nginx
        namespace: flux-system
  values:
    replicaCount: 2
    service:
      externalTrafficPolicy: Local
      loadBalancerIP: ""
    ingressClass: nginx-external
    publishService:
      enabled: true
EOF

git add -A && git commit -m "Nginx ingress controller added" && git push
```

After couple of seconds the NGINX Ingress Controller is running in the cluster and has an external IP address:

```bash
$ kubectl get svc -n ingress-nginx
NAME                                               TYPE           CLUSTER-IP       EXTERNAL-IP   PORT(S)                      AGE
ingress-nginx-ingress-controller                   LoadBalancer   172.21.213.148   x.x.x.x       80:30503/TCP,443:31755/TCP   52s
ingress-nginx-ingress-controller-default-backend   ClusterIP      172.21.40.9      <none>        80/TCP                       52s
```

So what next? The cluster is ready, ingress controller works, Flux can decrypt files encrypted by SOPS using Azure Key Vault's Key and now I am going to build an NGINX container with a custom HTML and push it to the Azure Container Registry created by Bicep earlier and use Flux to deploy it. I am also going to set Flux to watch for the new versions of the image and push the changes directly to the *main* branch of the repo, so that the image will be updated automatically once there is a new version (using SemVer) in the container registry.

So let's prepare the NGINX image first:

```bash
mkdir images
cat >images/index.html<<-EOF
<!doctype html>
<html lang="en">
<head>
  <meta charset="utf-8">
  <title>Nginx - Test 1</title>
</head>
<body>
  <h2>Hello from Nginx container - Version 1</h2>
</body>
</html>
EOF
cat >images/Dockerfile<<-EOF
FROM nginx:latest
COPY ./index.html /usr/share/nginx/html/index.html
EOF

cd images
docker build -t mstov.azurecr.io/mynginx:v1.0.0 .
az acr login -n mstov
docker push mstov.azurecr.io/mynginx:v1.0.0
cd -
```

The container is ready, sitting in my ACR, but now I need to create a new source for the Flux and I need to provide pull secrets for it as **Flux v2 can't utilize MSIs to access ACR** just now (I have to create a Service Principal for that). I am also going to create Flux `ImageRepository`, `imagePolicy` and `imageUpdateAutomation`. Let's use command line to create these resources this time:

```bash
ACR_REGISTRY_ID=$(az acr show --name mstov.azurecr.io --query id --output tsv)
SP_PASSWD=$(az ad sp create-for-rbac --name http://flux-cd-example --scopes $ACR_REGISTRY_ID --role acrpull --query password --output tsv)
SP_APP_ID=$(az ad sp show --id http://flux-cd-example --query appId --output tsv)

echo "Service principal ID: $SP_APP_ID"
echo "Service principal password: $SP_PASSWD"

kubectl create secret docker-registry flux-acr\
    -n flux-system \
    --docker-server=mstov.azurecr.io \
    --docker-username=$SP_APP_ID \
    --docker-password=$SP_PASSWD

flux create image repository mynginx \
    --interval 1m \
    --image mstov.azurecr.io/mynginx \
    --secret-ref flux-acr

flux create image policy mynginx \
  --select-semver vx.x.x \
  --image-ref mynginx

flux create image update mynginx \
  --author-email fluxcd-noreply@example.com \
  --author-name flux-cd \
  --checkout-branch main \
  --commit-template '{{range .Updated.Images}}{{println .}}{{end}}' \
  --git-repo-ref flux-system \
  --git-repo-path "apps/test/mynginx/"
```

>**Note:** You may wan to run the previous `flux` commands with `--export` option to get the YAML output and put that to the Git repo. You can also use `flux export <parameters>` to get the YAMLs of the resources already deployed to the Kubernetes cluster.

The commands above will create an *Azure AD Service Principal*, assign a role to it, create *image pull secrets* and create Flux related resources which will check ACR for the image versions and when an updated image version is pushed to the registry, Flux will update files with the proper tag. Flux knows what to change based on a comment on the specified line in the Kubernetes YAML files (see the `deployment` resource bellow).

I want the NGINX Ingress controller to terminate TLS and so here comes *SOPS* into play as I am going to store the certificates in the Git.
First, let's use openssl to generate a key and a certificate and create a secret object from them:

```bash
openssl req -x509 -newkey rsa:4096 -keyout key.pem -out cert.pem -days 365 -nodes
mkdir -p apps/test/mynginx
kubectl create secret tls example-certs --key=key.pem --cert=cert.pem --dry-run=client --output=yaml > apps/test/mynginx/certificates-secret.yaml
```

Let's create a `.sops.yaml` file and encrypt the values in the secret:

```bash
cat >apps/test/mynginx/.sops.yaml<<-EOF
creation_rules:
  - path_regex: .*.yaml
    encrypted_regex: ^(tls.key|tls.crt)$
    azure_keyvault: https://kv-mstov-test.vault.azure.net/keys/sops/5de841e201d14a62a04ffbb4d0d03fad
EOF
sops -e -i apps/test/mynginx/certificates-secret.yaml
```

>**Note:** You can obtain SOPS from it's [Github releases](https://github.com/mozilla/sops/releases)

Encryption is complete and now I am going to create the Kubernetes resources for my NGINX container. This includes `deployment`, `service` and `ingress`.

```bash
mkdir apps/test/mynginx
echo "  - mynginx/" >> apps/test/kustomization.yaml
cat >apps/test/mynginx/kustomization.yaml<<-EOF
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - mynginx.yaml
  - certificates-secret.yaml
EOF

cat >apps/test/mynginx/mynginx.yaml<<-EOF
---
apiVersion: v1
kind: Service
metadata:
  name: mynginx
spec:
  selector:
    app: mynginx
  ports:
  - port: 80
    targetPort: 80
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: mynginx
  labels:
    name: mynginx
spec:
  tls:
    - hosts:
      - "mynginx.example.com"
      secretName: example-certs
  rules:
  - host: mynginx.example.com
    http:
      paths:
      - pathType: Prefix
        path: "/"
        backend:
          service:
            name: mynginx
            port: 
              number: 80

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mynginx
spec:
  selector:
    matchLabels:
      app: mynginx
  template:
    metadata:
      labels:
        app: mynginx
    spec:
      containers:
      - name: mynginx
        image: mstov.azurecr.io/mynginx:v1.0.0 # {"$imagepolicy": "flux-system:mynginx"}
        resources:
          limits:
            memory: "128Mi"
            cpu: "500m"
        ports:
        - containerPort: 80
EOF

git add -A && git commit -m "MyNginx added" && git push
```

Wait a few seconds and the Nginx is ready. Let's see what is the version of the image here and check if the secret with the certificates is present:

```bash
$ kubectl get deploy -n default mynginx --output=jsonpath='{.spec.template.spec.containers[*].image}'
mstov.azurecr.io/mynginx:v1.0.0

$ kubectl get secret -n default example-certs
NAME                              TYPE                                  DATA   AGE
example-certs                     kubernetes.io/tls                     2      2m23s
```

Perfect. So let's try to build the image again, this time version 2:

```bash
cd images
sed -i 's/1/2/' index.html
docker build -t mstov.azurecr.io/mynginx:v2.0.0 .
docker push mstov.azurecr.io/mynginx:v2.0.0
```

After a while, we should see that the deployment image has been updated:

```bash
$ kubectl get deploy -n default mynginx --output=jsonpath='{.spec.template.spec.containers[*].image}'
mstov.azurecr.io/mynginx:v2.0.0
```

And there we go! The image has been updated automatically and the version has been changed in the git repository. All of that without touching anything! Anyway, I just want you to know that the examples in this blog post are far from perfect. This just highlights how Flux v2 works and what are it's capabilities. For more "production ready" usage, you may want to utilize Kustomize more (for example split the common parts of the YAML files to a "global" directory and update just values which are different accross the environments), you also may want to put the infrastructure components like AAD Pod Identity and Ingress controller to a different directory than your applications, etc., etc. BUT, you should have pretty good idea of how it all works and how it can save a lot of time with deployments and also prevent manual changes to your clusters as any manual change is reconciled back to the state described in the git repository. Not mentioning the fact that you don't need to store credentials in your CI/CD system as Flux is based on the Pull model instead of traditional Push from a CI/CD worker node.

Just try it and see if it works for you! It definitely works for me! (-:
