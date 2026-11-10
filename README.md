# github-actions

Repositorio de **composite actions** reutilizables (`action.yaml`) para los workflows de GitHub de Billin. Cada carpeta del raíz es una action distinta; en los demás repos se referencian así:

```yaml
- uses: BillinTeam/github-actions/<nombre-de-la-carpeta>@v2.24
  with:
    # inputs de esa action
```

## Versionado

Todas las actions comparten **el mismo número de versión**. No hay tags por carpeta: un tag `v2.N` apunta a un commit de `main` y fija la snapshot de **todas** las actions de ese momento.

| Concepto | Detalle |
|----------|---------|
| Tag en remoto | `v2.23` (último publicado en GitHub al documentar esto) |
| Próximo tag previsto | `v2.24` — Helm 4: `--verify=false` en instalación de plugins (`delivery-k8s`, `helm-push`) |
| Referencia en YAML | Siempre el prefijo `v` + semver menor de equipo: `@v2.24`, no `@2.24` ni `@main` en producción |

Al subir versión en un repo consumidor (p. ej. tortilla), conviene **alinear** todas las referencias a `BillinTeam/github-actions/*` del mismo tag salvo que haya una razón documentada para quedarse en uno antiguo.

## Acciones disponibles

| Carpeta | Para qué sirve | Inputs / outputs destacados |
|---------|----------------|----------------------------|
| [`setup-node`](setup-node/action.yaml) | Checkout, Node (20.x por defecto) y caché de `node_modules` vía `package-lock.json`. | Input: `node-version`. Output: `dependencies-cache-hit`. |
| [`pr-title-validator`](pr-title-validator/action.yaml) | Comprueba que el título del PR sigue [Conventional Commits](https://www.conventionalcommits.org/); comenta en el PR y falla el job si no cumple. | Input: `repo-token`. |
| [`gcp-auth`](gcp-auth/action.yaml) | Auth en GCP (`google-github-actions/auth` + `setup-gcloud`); opcionalmente credenciales del cluster GKE. | Inputs: `gcpProjectId`, `gcpServiceAccountKey`, `getGkeCredentials`, `gcpZone`. |
| [`feat-branch-name`](feat-branch-name/action.yaml) | Deriva namespace y nombre corto de rama feat/hotfix para deploys efímeros (regex de `REGEX_*` en el workflow). | Input: `appPrefix`, `deployFromPR`. Outputs: `namespace`, `branch_short`. |
| [`docker-push`](docker-push/action.yaml) | Build y push de imagen Docker a Artifact Registry (`europe-west1-docker.pkg.dev`); expone digest SHA256. | Inputs: `dockerfilePath`, `dockerImageUrl`, `dockerImageTag`, GCP creds. Output: `docker_image_sha`. |
| [`helm-push`](helm-push/action.yaml) | Empaqueta chart Helm, sustituye tag/digest en `values.yaml`, sube a Chart Museum (`helm cm-push`). Descifra repos con sops. | Inputs: rutas de chart, versión, Chart Museum, tag/digest de imagen. |
| [`delivery-k8s`](delivery-k8s/action.yaml) | Deploy en GKE con **helmfile**: sops + repos Helm, plugin helm-secrets, `helmfile sync` (feat/dev). Puede inyectar tag/digest y vars extra en values. | Inputs: `baseDeployDir`, `environment`, `featNamespace`, `featBranchShort`, cluster/zone, `extraHelmEnvVars`, etc. |
| [`destroy-environment`](destroy-environment/action.yaml) | Borra el namespace de Kubernetes de una rama feat/hotfix al eliminar la rama (destroy workflow). | Inputs: `namespacePrefix`, `gkeClusterName`, `gcpClusterZone`. Output: `branch_short`. |

### Ejemplos de uso (tortilla, referencia)

| Action | Versión usada en tortilla (ejemplo) | Dónde |
|--------|-------------------------------------|-------|
| `setup-node` | `@v2.14` | install-dependencies, lint, test, build-app, helm-push workflow |
| `gcp-auth` | `@v2.22` | helm-deploy, e2e, destroy-feature |
| `feat-branch-name` | `@v2.23` | helm-deploy |
| `delivery-k8s` | `@v2.23` | helm-deploy |
| `destroy-environment` | `@v2.23` | destroy-feature |
| `helm-push` | `@v2.18` | composite helm-push (tortilla fija Helm 3.10 antes de llamarla) |
| `pr-title-validator` | `@v2.16` | workflow pr-title |

Tras publicar `v2.24`, actualizar al menos `delivery-k8s` (y, si se quiere consistencia, el resto de `@v2.23`) en los repos afectados.

## Publicar una nueva versión (tag usable en otros repos)

1. **Cambios en `main`**  
   - Modificar las actions necesarias.  
   - Añadir al inicio de este README una sección `## TAG 2.N` con una línea que resuma el cambio (como las versiones anteriores).

2. **Merge**  
   - El tag debe crearse sobre el commit que ya está en `origin/main` (normalmente el merge del PR).

3. **Crear y subir el tag** (desde clone actualizado):

   ```bash
   cd /path/to/github-actions
   git fetch origin
   git checkout main
   git pull origin main

   # Revisar que el commit es el correcto
   git log -1 --oneline

   # Tag ligero (mismo estilo que v2.23 y anteriores)
   git tag v2.24

   # Opcional: tag anotado (mensaje visible con git show)
   # git tag -a v2.24 -m "Helm 4: --verify=false on helm plugin install"

   git push origin main
   git push origin v2.24
   ```

4. **Comprobar en GitHub**  
   - En **Releases / Tags** debe aparecer `v2.24`.  
   - Probar en un workflow: `uses: BillinTeam/github-actions/delivery-k8s@v2.24`.

5. **Repos consumidores**  
   - PR que sustituya `@v2.23` (u otra) por `@v2.24` donde corresponda.  
   - No hace falta publicar cada carpeta por separado: un solo tag actualiza todas las actions.

### Notas

- **No reescribir tags** ya usados en producción (`git tag -f` / force-push de tags): los workflows cachean por referencia; un tag movido rompe reproducibilidad.
- Si solo cambia una action pero el equipo versiona el monorepo junto, **igualmente** se incrementa `v2.N` global.
- Para probar un cambio **antes** del tag, los repos pueden apuntar temporalmente a un SHA (`@<commit-full-sha>`), solo en ramas de prueba; en `main` de producto usar siempre tag `v2.N`.

## Historial de versiones

Cambios anotados en este README. Tags `v2.15`–`v2.17` existen en el remoto sin entrada aquí (ver `git log` entre tags si hace falta detalle).

### TAG 2.24

Helm 4: añadir `--verify=false` al instalar plugins desde URLs git (`helm-secrets` en `delivery-k8s`, `helm-push` en chartmuseum).

### TAG 2.23

Variables extra de entorno para Helm en `delivery-k8s` y refactor de actions.

### TAG 2.22

Corrección de formato YAML en `gcp-auth`.

### TAG 2.21

`gcpZone` deja de ser obligatorio en `gcp-auth`.

### TAG 2.20

Nueva action `gcp-auth`.

### TAG 2.19

Login a Google Artifact Registry en `docker-push`.

### TAG 2.18

`helm-push`: uso de `helm cm-push` (Chart Museum).

### TAG 2.14

`setup-node`: Node 20.x.

### TAG 2.8

`setup-node`: Node 16.x.

### TAG 2.7

`setup-node`: Node 14.x.
