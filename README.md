# civo-workshop

## Bootstrapping the gitops automation

Create a Github Personal Access Token over https://github.com/settings/tokens with repo read/write permission.

```
export GITHUB_USER=<your-username>
```

```
flux bootstrap github \
  --owner=$GITHUB_USER \
  --repository=gitops-fleet \
  --branch=main \
  --path=./clusters/civo-workshop \
  --personal
```

```
git clone https://$GITHUB_USER@github.com/$GITHUB_USER/gitops-fleet
cd gitops-fleet
```

### Verifying Flux

```
flux get sources git
flux get kustomizations
```

```
NAME            REVISION        SUSPENDED       READY   MESSAGE
flux-system     main/b4ed92d    False           True    stored artifact for revision 'main/b4ed92d8ecf897ae32675813d564ce48bd39c62c'

NAME            REVISION        SUSPENDED       READY   MESSAGE
flux-system     main/b4ed92d    False           True    Applied revision: main/b4ed92d
```

```
$ kubectl get crd | grep flux

alerts.notification.toolkit.fluxcd.io        2023-02-06T17:45:41Z
buckets.source.toolkit.fluxcd.io             2023-02-06T17:45:41Z
gitrepositories.source.toolkit.fluxcd.io     2023-02-06T17:45:41Z
helmcharts.source.toolkit.fluxcd.io          2023-02-06T17:45:41Z
helmreleases.helm.toolkit.fluxcd.io          2023-02-06T17:45:41Z
helmrepositories.source.toolkit.fluxcd.io    2023-02-06T17:45:42Z
kustomizations.kustomize.toolkit.fluxcd.io   2023-02-06T17:45:42Z
ocirepositories.source.toolkit.fluxcd.io     2023-02-06T17:45:42Z
providers.notification.toolkit.fluxcd.io     2023-02-06T17:45:42Z
receivers.notification.toolkit.fluxcd.io     2023-02-06T17:45:42Z
```

```
kubectl get gitrepositories.source.toolkit.fluxcd.io -A
kubectl get kustomizations.kustomize.toolkit.fluxcd.io -A
```

```
kubectl get ns
kubectl get pods -n flux-system
```

```
kubectl logs -n flux-system deploy/source-controller
kubectl logs -n flux-system deploy/kustomize-controller
```

## Flux sync config

```
cat ~/gitops-fleet/clusters/civo-workshop/flux-system/gotk-sync.yaml
```

```
---
apiVersion: source.toolkit.fluxcd.io/v1beta2
kind: GitRepository
metadata:
  name: flux-system
  namespace: flux-system
spec:
  interval: 1m0s
  ref:
    branch: main
  secretRef:
    name: flux-system
  url: ssh://git@github.com/laszlocph/gitops-fleet
---
apiVersion: kustomize.toolkit.fluxcd.io/v1beta2
kind: Kustomization
metadata:
  name: flux-system
  namespace: flux-system
spec:
  interval: 10m0s
  path: ./clusters/civo-workshop
  prune: true
  sourceRef:
    kind: GitRepository
    name: flux-system
```

Check https://github.com/<your-user>/gitops-fleet/settings/keys
Flux bootstrap added deploy keys that are only good for this one repo.

## Deploy your first app

```
flux create source git podinfo \
  --url=https://github.com/stefanprodan/podinfo \
  --branch=master \
  --interval=30s \
  --export > ~/gitops-fleet/clusters/civo-workshop/podinfo-source.yaml
```

```
flux create kustomization podinfo \
  --target-namespace=default \
  --source=podinfo \
  --path="./kustomize" \
  --prune=true \
  --interval=5m \
  --export > ~/gitops-fleet/clusters/civo-workshop/podinfo-kustomization.yaml
```

```
tree
cat ~/gitops-fleet/clusters/civo-workshop/podinfo-source.yaml
cat ~/gitops-fleet/clusters/civo-workshop/podinfo-kustomization.yaml

git status
git add .
git commit -m "First application"
git push origin main

flux get kustomizations --watch
flux get kustomizations

kubectl get pods -A
```

```
 user2-c9985df7f-fsr8x  ~ / gitops-fleet  my-vcluster/default  flux get kustomizations --watch
NAME            REVISION                                        SUSPENDED       READY   MESSAGE
flux-system     main/b4ed92d8ecf897ae32675813d564ce48bd39c62c   False           Unknown Reconciliation in progress
podinfo         False   False   waiting to be reconciled
podinfo         False   False   waiting to be reconciled
flux-system     main/b4ed92d    False   True    Applied revision: main/0890d4b
flux-system     main/0890d4b    False   True    Applied revision: main/0890d4b
podinfo         False   False   Source is not ready, artifact not found
podinfo         False   Unknown Reconciliation in progress
podinfo         False   Unknown Reconciliation in progress
podinfo         False   Unknown Reconciliation in progress
podinfo         False   Unknown Reconciliation in progress
podinfo         False   True    Applied revision: master/eac008b
podinfo master/eac008b  False   True    Applied revision: master/eac008b
^[[A^C
 user2-c9985df7f-fsr8x  ~ / gitops-fleet  my-vcluster/default  flux get kustomizations
NAME            REVISION        SUSPENDED       READY   MESSAGE
flux-system     main/0890d4b    False           True    Applied revision: main/0890d4b
podinfo         master/eac008b  False           True    Applied revision: master/eac008b
 user2-c9985df7f-fsr8x  ~ / gitops-fleet  my-vcluster/default  k get pods -A
NAMESPACE     NAME                                       READY   STATUS    RESTARTS   AGE
kube-system   coredns-664d7f9fd9-skndj                   1/1     Running   0          56m
flux-system   kustomize-controller-69fc9bdc9-hgwpf       1/1     Running   0          32m
flux-system   notification-controller-5976649595-c9kbk   1/1     Running   0          32m
flux-system   helm-controller-57779c6448-948cg           1/1     Running   0          32m
flux-system   source-controller-77655b868b-spsk6         1/1     Running   0          32m
default       podinfo-5c8b979458-vz6hl                   1/1     Running   0          33s
default       podinfo-5c8b979458-sk62m                   1/1     Running   0          18s
```

## Cluster components

### Ingress controller

```
cat > ~/gitops-fleet/clusters/civo-workshop/ingress-nginx.yaml << EOF
---
apiVersion: source.toolkit.fluxcd.io/v1beta1
kind: HelmRepository
metadata:
  name: ingress-nginx
  namespace: default
spec:
  interval: 60m
  url: https://kubernetes.github.io/ingress-nginx
---
apiVersion: helm.toolkit.fluxcd.io/v2beta1
kind: HelmRelease
metadata:
  name: ingress-nginx
  namespace: default
spec:
  interval: 60m
  releaseName: ingress-nginx
  chart:
    spec:
      chart: ingress-nginx
      version: 4.2.3
      sourceRef:
        kind: HelmRepository
        name: ingress-nginx
      interval: 10m
  values:
    controller:
      metrics:
        enabled: true
        service:
          annotations:
            prometheus.io/port: "10254"
            prometheus.io/scrape: "true"
      service:
        annotations:
          external-dns.alpha.kubernetes.io/hostname: "*.$NS.workshop.gimlet.io"
    defaultBackend:
      enabled: true
EOF
```

```
tree
cat ~/gitops-fleet/clusters/civo-workshop/ingress-nginx.yaml

git status
git add .
git commit -m "Ingress controller"
git push origin main

flux get kustomizations

kubectl get pods -A
kubectl get svc -A
```

```
ingress-nginx-controller             LoadBalancer   10.43.91.152    74.220.28.247   80:32354/TCP,443:30870/TCP   5m46s
```

```
dig xx.$NS.workshop.gimlet.io
```

Visit http://xx.$NS.workshop.gimlet.io

#### Ingress for podinfo

```
cat > ~/gitops-fleet/clusters/civo-workshop/podinfo-ingress.yaml << EOF
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: podinfo
  namespace: default
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - "podinfo.$NS.workshop.gimlet.io"
      secretName: tls-podinfo
  rules:
    - host: "podinfo.$NS.workshop.gimlet.io"
      http:
        paths:
          - path: "/"
            pathType: "Prefix"
            backend:
              service:
                name: podinfo
                port:
                  number: 9898
EOF
```

```
tree
cat ~/gitops-fleet/clusters/civo-workshop/podinfo-ingress.yaml

git status
git add .
git commit -m "Ingress test"
git push origin main

flux get kustomizations

kubectl get ing
```

Visit http://podinfo.$NS.workshop.gimlet.io/

### SSL

```
flux create source git civo-workshop \
  --url=https://github.com/gimlet-io/civo-workshop \
  --branch=main \
  --interval=30s \
  --export > ~/gitops-fleet/clusters/civo-workshop/civo-workshop-source.yaml

flux create kustomization cert-manager \
  --target-namespace=default \
  --source=civo-workshop \
  --path="./cert-manager" \
  --prune=true \
  --interval=5m \
  --export > ~/gitops-fleet/clusters/civo-workshop/cert-manager.yaml

flux create kustomization cert-manager-issuer \
  --target-namespace=default \
  --source=civo-workshop \
  --path="./cert-manager-issuer" \
  --prune=true \
  --interval=5m \
  --export > ~/gitops-fleet/clusters/civo-workshop/cert-manager-issuer.yaml \
  --depends-on cert-manager
```

```
tree
cat ~/gitops-fleet/clusters/civo-workshop/cert-manager.yaml
cat ~/gitops-fleet/clusters/civo-workshop/cert-manager-issuer.yaml
```

### Prometheus

```
cat > ~/gitops-fleet/clusters/civo-workshop/prometheus.yaml << EOF
---
apiVersion: source.toolkit.fluxcd.io/v1beta1
kind: HelmRepository
metadata:
  name: prometheus
  namespace: default
spec:
  interval: 60m
  url: https://prometheus-community.github.io/helm-charts
---
apiVersion: helm.toolkit.fluxcd.io/v2beta1
kind: HelmRelease
metadata:
  name: prometheus
  namespace: default
spec:
  interval: 60m
  releaseName: prometheus
  chart:
    spec:
      chart: prometheus
      version: 15.10.1
      sourceRef:
        kind: HelmRepository
        name: prometheus
      interval: 10m
  values:
    pushgateway:
      enabled: false
    alertmanager:
      enabled: false
    server: 
      retention: "14d"
      global:
        scrape_interval: 15s
EOF
```

### Grafana

```
cat > ~/gitops-fleet/clusters/civo-workshop/grafana.yaml << EOF
---
apiVersion: source.toolkit.fluxcd.io/v1beta1
kind: HelmRepository
metadata:
  name: grafana
  namespace: default
spec:
  interval: 60m
  url: https://grafana.github.io/helm-charts
---
apiVersion: helm.toolkit.fluxcd.io/v2beta1
kind: HelmRelease
metadata:
  name: grafana
  namespace: default
spec:
  interval: 60m
  releaseName: grafana
  chart:
    spec:
      chart: grafana
      version: 6.32.2
      sourceRef:
        kind: HelmRepository
        name: grafana
      interval: 10m
  values:
    sidecar:
      datasources:
        enabled: true
      dashboards:
        enabled: true
    ingress:
      enabled: true
      ingressClassName: nginx
      annotations:
        cert-manager.io/cluster-issuer: letsencrypt
      tls:
        - secretName: tls-grafana
          hosts:
            - grafana.$NS.workshop.gimlet.io
      hosts:
        - grafana.$NS.workshop.gimlet.io
      path: /
EOF
```

```
flux create kustomization grafana-config \
  --target-namespace=default \
  --source=civo-workshop \
  --path="./grafana-config" \
  --prune=true \
  --interval=5m \
  --export > ~/gitops-fleet/clusters/civo-workshop/grafana-config.yaml
```

Deploy, then visit: https://grafana.$NS.workshop.gimlet.io

Grafana `admin` user password:
```
kubectl get secret grafana --template='{{ index .data "admin-password"}}' | base64 -d
```

## Deploy to gitops with Github Actions

- fork & clone https://github.com/gimlet-io/civo-workshop
- adjust https://github.com/gimlet-io/civo-workshop/blob/main/.github/workflows/gitops-deploy.yaml
- an example run: https://github.com/gimlet-io/civo-workshop/actions/runs/4109111636
