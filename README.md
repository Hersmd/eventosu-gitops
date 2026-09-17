# eventosu-gitops

Repositorio de manifiestos declarativos de EventosU — **única fuente de
verdad** del estado del clúster. Nada se aplica al clúster fuera de lo
que ArgoCD sincroniza desde aquí.

Repo de código (Dockerfiles, charts de Helm, pipeline de CI/CD):
`https://github.com/<owner>/<repo-de-codigo>` — ver `/P8` ahí.

## Estructura

```
argocd/
  project.yaml          AppProject — limita ArgoCD a namespaces sa-p5/sa-p5-dev
  applicationset.yaml   Genera 1 Application por servicio × ambiente
apps/
  <servicio>/
    image.yaml           ← el pipeline de CI/CD SOLO toca este archivo, vía PR
    sealed-secrets/
      sealed-secret-dev.yaml    (no versionado hasta que lo generes con kubeseal)
      sealed-secret-prod.yaml
```

Los charts de Helm en sí (plantillas, `values.yaml`/`values-dev.yaml`/
`values-prod.yaml`) viven en el repo de código, en `P8/charts/`. Cada
`Application` que genera el `ApplicationSet` combina 3 fuentes:
1. el chart, desde el repo de código,
2. los valores por ambiente (`values-{dev,prod}.yaml`, del propio
   chart) + `image.yaml` (de este repo, vía referencia `$values`),
3. el `SealedSecret` de ese ambiente, como manifiesto crudo.

## Bootstrap (una sola vez, manual — es la única excepción a "todo pasa
por ArgoCD", porque es literalmente instalar ArgoCD)

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-rollouts/stable/manifests/install.yaml

# reemplaza hersmd/Practicas-SA-B-201704312 en project.yaml y
# applicationset.yaml con tus valores reales antes de aplicar
kubectl apply -f argocd/project.yaml
kubectl apply -f argocd/applicationset.yaml
```

A partir de aquí, cualquier cambio a este repo (o a `P8/charts/` en el
repo de código) se sincroniza solo.

## Cómo llega un cambio hasta aquí

1. Alguien mergea a `main` en el repo de código.
2. Se crea un tag `vX.Y.Z` → dispara el pipeline (build, Trivy, SBOM,
   firma con Cosign).
3. El pipeline abre un Pull Request **en este repo**, actualizando
   `apps/*/image.yaml` con el nuevo tag.
4. Alguien revisa y mergea ese PR (revisión humana — es la puerta de
   calidad antes de que ArgoCD vea el cambio).
5. ArgoCD detecta el diff y sincroniza — Argo Rollouts hace la
   promoción canary paso a paso, con el `AnalysisTemplate` de cada
   servicio como gate.
