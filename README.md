# Kubernetes Handbook

Kubernetes handbook, this repo utilizes Tencent Cloud K3S as the demo environment  

You should already have a K3S/miniKube/K8S env with kubectl and envsubst as precondition

## Install

### Set Kube Config

By default, `$HOME/.kube` might exist an `config` and you do not need config it

<details open>
<summary> [Optional] </summary>

#### 1. Edit `/etc/profile`, you may need `sudo` to add
```bash
export KUBECONFIG=/etc/rancher/k3s/k3s.yaml
```
For lighthouse user, you may need to add `KUBECONFIG` in `.bashrc`

> You can exit Vim by `:wq!`

> In Tencent Cloud, you might not be able to leave Insert Mode  
> Please press `Ctrl + Shift + F5`  
> Or use `Ctrl + C` to force Command Mode  
> Or maybe you are in Record Mode, press `Q` to finish  


#### 2. Apply

```
source /etc/profile
```
> You can check if it works by `echo $KUBECONFIG`
</details> 

#### You can set env `EMAIL` and `DOMAIN` for later use according to step 1:
```bash
export EMAIL=xxx.xxx@gmail.com
export DOMAIN=xxx.com
```

### Install Helm
```
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```

### Add Helm Repo
```
helm repo add jetstack https://charts.jetstack.io && helm repo update
```

## Cert

### Install Cert-Manager

#### 1. Create namespace
```
kubectl create namespace cert-manager
```

#### 2. Install cert-manager

Check installeable version
```
helm search repo cert-manager
```

```
helm install \
  cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --set installCRDs=true
```
>   `--set installCRDs=true` : For ClusterIssuer later  
>   `--version v1.xx.x` ：You can specify an old version

#### 3. [Optional] Install CRD

Skip this step ff you already set `installCRDs` in previous

<details>
<summary> Manual install CRD </summary>

Check installed version
```
helm list -n cert-manager
```
Apply the yaml, REMEMBER to replace the version number!!!
```
kubectl apply --validate=false -f https://github.com/jetstack/cert-manager/releases/download/v1.14.5/cert-manager.yaml
```
</details>  


> This step could be also used for installation without `helm`

#### 4. [Optional] Check Running Pods

```
kubectl get pods --namespace cert-manager
```

### Config ClusterIssuer

Refer to [K3S Rocks](https://k3s.rocks/https-cert-manager-letsencrypt/)

#### 1. Add ClusterIssuer

<details open>
<summary>letsencrypt-prod.yaml</summary>

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    email: $EMAIL    # Email Address
    server: https://acme-v02.api.letsencrypt.org/directory
    privateKeySecretRef:
      name: letsencrypt-prod
    solvers:
    - http01:
        ingress:
          class: traefik
---

```

</details>

#### 2. Add Middleware

<details open>
<summary>traefik-https-redirect-middleware.yaml</summary>

```yaml
apiVersion: traefik.containo.us/v1alpha1
kind: Middleware
metadata:
  name: redirect-https
spec:
  redirectScheme:
    scheme: https
    permanent: true
---


```

</details>

#### 3. Add Ingress

<details open>
<summary>whoami-ingress-tls.yaml</summary>

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: whoami-tls-ingress
  annotations:
    spec.ingressClassName: traefik
    cert-manager.io/cluster-issuer: letsencrypt-prod
    traefik.ingress.kubernetes.io/router.middlewares: default-redirect-https@kubernetescrd
    # default is Namespace, redirect-https is Middleware name
spec:
  rules:
    - host: whoami.$DOMAIN        # Domain name
      http:
        paths:
          - path: /
            pathType: Prefix      # Prefix | ImplementationSpecific
            backend:
              service:
                name: whoami-svc  # Service name
                port:
                  number: 8888    # Service port, not pod port
  tls:
    - secretName: whoami-tls      # Certificate name
      hosts:
        - whoami.$DOMAIN          # Domain name
---

```

</details>

#### 4. Add Service

<details open>
<summary>nginx-service.yaml</summary>

```yaml

apiVersion: v1
kind: Service
metadata:
  labels:
    app: nginx-app
  name: whoami-svc
  namespace: default
spec:
  ports:
    - port: 8888        # Service port
      protocol: TCP
      name: app-port
      targetPort: 80
  type: ClusterIP
  selector:
    app: nginx-app
---

```

</details>

#### 5. Add Deployment

<details open>
<summary>nginx-deployment.yaml</summary>

```yaml

apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  selector:
    matchLabels:
      app: nginx-app
  replicas: 2 # Tells deployment to run 2 pods matching the template
  template:
    metadata:
      labels:
        app: nginx-app
    spec:
      containers:
      - name: nginx-app
        image: nginx:1.26.0
        ports:
        - containerPort: 80
---

```

</details>


## Deployment

### Check Deployment

```
cat *.yaml | envsubst
```
> Check if `DOMAIN` and `EMAIL` is correctly filled

### Apply Deployment

```
cat *.yaml | envsubst | kubectl apply -f -
```
> Each yaml file should have a separator `---`  with line break in the end, otherwise it cannot be parsed

### Delete Deployment
```
kubectl delete all -l app=nginx-app
```

> `delete all` will not delete ingress,configmap,etc.

## Done

Now, you can check the welcome page by:  

https://whoami.$DOMAIN/

<EOF>

