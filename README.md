# AulaViva GitOps

Repositorio de despliegue GitOps del backend de **AulaViva**, una plataforma educativa que utiliza Kubernetes para gestionar sus despliegues en los entornos de desarrollo (`dev`) y producción (`prod`).

La configuración utiliza Argo CD para sincronizar los recursos de Kubernetes desde Git, Kustomize para organizar los manifiestos por entorno y `kind` para ejecutar un clúster Kubernetes local.

## 1. Objetivos

- Automatizar la entrega de aplicaciones mediante GitOps.
- Mantener separados los entornos de desarrollo y producción.
- Gestionar los manifiestos de Kubernetes mediante Kustomize.
- Permitir que Argo CD detecte y sincronice cambios del repositorio.
- Facilitar la reproducción del entorno local por parte de otros integrantes del equipo.
- Validar los manifiestos antes de utilizarlos en el clúster.
- Documentar el procedimiento de rollback mediante reversión de commits.

## 2. Arquitectura

El flujo de despliegue es el siguiente:

1. El código del backend se construye y se publica como imagen de contenedor en GitHub Container Registry (GHCR).
2. El repositorio GitOps mantiene los manifiestos y las configuraciones de cada entorno.
3. GitHub Actions valida los manifiestos Kustomize.
4. Argo CD observa el repositorio y compara el estado deseado con el estado del clúster.
5. Kubernetes ejecuta los pods del backend y administra sus réplicas.
6. Ingress NGINX permite acceder a las aplicaciones mediante los nombres de host configurados.

### Componentes principales

| Componente | Función |
|---|---|
| Docker | Construcción y ejecución de contenedores. |
| kind | Clúster Kubernetes local. |
| kubectl | Administración del clúster y validación de manifiestos. |
| Kustomize | Personalización de manifiestos por entorno. |
| GitHub Actions | Validación automatizada de los overlays. |
| Argo CD | Sincronización GitOps. |
| Ingress NGINX | Enrutamiento del tráfico HTTP/HTTPS. |
| GHCR | Almacenamiento de imágenes del backend. |
| HPA | Escalado horizontal de pods según métricas. |

### Estructura del repositorio

```text
equipo-aulaviva-gitops/
├── .github/
│   └── workflows/
│       └── kustomize-validate.yml
├── apps/
│   └── aulaviva-backend/
│       ├── base/
│       │   ├── configmap.yaml
│       │   ├── deployment.yaml
│       │   ├── hpa.yaml
│       │   ├── ingress.yaml
│       │   ├── kustomization.yaml
│       │   └── service.yaml
│       └── overlays/
│           ├── dev/
│           │   ├── deployment-patch.yaml
│           │   ├── hpa-patch.yaml
│           │   ├── ingress-patch.yaml
│           │   └── kustomization.yaml
│           └── prod/
│               ├── deployment-patch.yaml
│               ├── hpa-patch.yaml
│               ├── ingress-patch.yaml
│               └── kustomization.yaml
├── argocd/
│   ├── aulaviva-backend-dev.yaml
│   └── aulaviva-backend-prod.yaml
├── infra/
│   └── kind/
│       └── cluster-config.yaml
└── README.md
```

La estructura anterior representa los archivos esperados. Si se agrega o cambia un archivo, debe mantenerse la correspondencia con los recursos que referencian los manifiestos Kustomize.

## 3. Entornos

| Característica | Desarrollo (`dev`) | Producción (`prod`) |
|---|---|---|
| Aplicación Argo CD | `aulaviva-backend-dev` | `aulaviva-backend-prod` |
| Namespace | `equipo-aulaviva-dev` | `equipo-aulaviva-prod` |
| Overlay Kustomize | `apps/aulaviva-backend/overlays/dev` | `apps/aulaviva-backend/overlays/prod` |
| Sincronización | Automática | Manual |
| Host Ingress | `dev.aulaviva.local` | `aulaviva.local` |

Los parámetros específicos, como el número de réplicas y los límites del HPA, se definen en los manifiestos de cada entorno.

La aplicación `dev` utiliza sincronización automática, con `prune` y `selfHeal` habilitados. La aplicación `prod` utiliza sincronización manual para permitir la revisión de los cambios antes de aplicarlos.

Crear la aplicación `prod` en Argo CD no implica que sus recursos ya estén desplegados. Para crear esos recursos, primero se debe validar la configuración y luego ejecutar la sincronización manual.

## 4. Requisitos previos

Instalar las siguientes herramientas en el equipo:

- [Docker Desktop](https://www.docker.com/products/docker-desktop/)
- [kubectl](https://kubernetes.io/docs/tasks/tools/)
- [kind](https://kind.sigs.k8s.io/docs/user/quick-start/)
- [Git](https://git-scm.com/downloads)

En Windows, iniciar Docker Desktop y asegurarse de que el motor de contenedores Linux esté funcionando antes de continuar.

Verificar las instalaciones desde PowerShell:

```powershell
docker version
kubectl version --client
kind version
git --version
```

Clonar el repositorio:

```powershell
git clone https://github.com/merinobranco10/equipo-aulaviva-gitops.git
cd equipo-aulaviva-gitops
```

## 5. Crear el clúster Kubernetes local

> Esta sección corresponde a una instalación desde cero. Si el clúster `aulaviva` ya existe y contiene recursos, no vuelvas a crearlo ni elimines el clúster para seguir esta guía.

El archivo de configuración del clúster se encuentra en:

`infra/kind/cluster-config.yaml`

Una configuración compatible con el entorno local puede ser:

```yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
  - role: control-plane
    labels:
      ingress-ready: "true"
    extraPortMappings:
      - containerPort: 80
        hostPort: 80
        protocol: TCP
      - containerPort: 443
        hostPort: 443
        protocol: TCP
```

La etiqueta `ingress-ready=true` permite programar el controlador Ingress en el nodo correspondiente. Los mapeos de puertos permiten exponer el tráfico HTTP y HTTPS al equipo anfitrión.

Crear el clúster únicamente si todavía no existe:

```powershell
kind create cluster --name aulaviva --config .\infra\kind\cluster-config.yaml
```

Verificar el clúster:

```powershell
kubectl config use-context kind-aulaviva
kubectl get nodes
kubectl cluster-info
```

El nodo debe aparecer en estado `Ready`.

Si el clúster ya existe, basta con ejecutar:

```powershell
kind get clusters
kubectl get nodes
kubectl config current-context
```

**Nota:** si el clúster existente no tiene los mapeos de puertos necesarios, la configuración del archivo no los añade retroactivamente. Para corregirlos puede ser necesario recrear el clúster, lo cual requiere planificar previamente la pérdida de sus recursos locales.

## 6. Instalar Ingress NGINX

Ingress NGINX permite dirigir solicitudes HTTP/HTTPS hacia los servicios internos de Kubernetes según el host definido en cada recurso Ingress.

Instalar el controlador compatible con `kind`:

```powershell
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/kind/deploy.yaml
```

Esperar a que el controlador esté disponible:

```powershell
kubectl wait --namespace ingress-nginx `
  --for=condition=Ready pod `
  --selector=app.kubernetes.io/component=controller `
  --timeout=180s
```

Comprobar la instalación:

```powershell
kubectl get pods -n ingress-nginx
kubectl get ingressclass
```

Debe aparecer la clase `nginx` y el pod controlador debe estar listo.

La instalación anterior crea los recursos necesarios en el namespace `ingress-nginx`; no es necesario crearlo manualmente.

## 7. Instalar Argo CD

Crear el namespace e instalar Argo CD solo si todavía no está instalado:

```powershell
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

Si el namespace ya existe, omitir el primer comando. Si Argo CD ya está instalado, no repetir la instalación sin revisar el estado existente.

Esperar a que los deployments estén disponibles:

```powershell
kubectl wait --for=condition=Available `
  deployment --all `
  -n argocd `
  --timeout=300s
```

Verificar los componentes:

```powershell
kubectl get pods -n argocd
kubectl get svc -n argocd
```

Para una instalación reproducible a largo plazo, se recomienda fijar una versión concreta de Argo CD.

### 7.1 Acceder a la interfaz de Argo CD

Ejecutar el siguiente comando en una terminal y mantenerlo abierto:

```powershell
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

Abrir en el navegador:

`https://localhost:8080`

El certificado inicial puede generar una advertencia del navegador en el entorno local.

### 7.2 Obtener la contraseña inicial

En otra terminal de PowerShell:

```powershell
$secret = kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}"

[System.Text.Encoding]::UTF8.GetString(
  [System.Convert]::FromBase64String($secret)
)
```

Iniciar sesión con el usuario `admin` y la contraseña obtenida. Se recomienda cambiar la contraseña inicial y no incluirla en archivos versionados ni capturas públicas.

## 8. Validar los manifiestos Kustomize

El repositorio contiene una base común y dos overlays: `dev` y `prod`.

El workflow `.github/workflows/kustomize-validate.yml` valida estas tres rutas en GitHub Actions cuando se producen cambios que coinciden con sus filtros de ejecución.

Para validar los manifiestos localmente, ejecutar desde la raíz del repositorio:

```powershell
kubectl kustomize apps/aulaviva-backend/base
kubectl kustomize apps/aulaviva-backend/overlays/dev
kubectl kustomize apps/aulaviva-backend/overlays/prod
```

Los tres comandos deben finalizar sin errores.

Esta validación genera los manifiestos resultantes, pero no los aplica al clúster y no comprueba por sí sola que los pods puedan iniciar correctamente.

### 8.1 Validación automatizada con GitHub Actions

El archivo `.github/workflows/kustomize-validate.yml` debe contener:

```yaml
name: GitOps - Kustomize Validation

on:
  push:
    branches:
      - main
    paths:
      - "apps/**"
      - ".github/workflows/kustomize-validate.yml"

  pull_request:
    branches:
      - main
    paths:
      - "apps/**"
      - ".github/workflows/kustomize-validate.yml"

  workflow_dispatch:

permissions:
  contents: read

jobs:
  validate-kustomize:
    name: Validate base, dev and prod
    runs-on: ubuntu-latest

    steps:
      - name: Checkout GitOps repository
        uses: actions/checkout@v4

      - name: Set up kubectl
        uses: azure/setup-kubectl@v4
        with:
          version: v1.32.0

      - name: Validate base
        run: kubectl kustomize apps/aulaviva-backend/base

      - name: Validate dev overlay
        run: kubectl kustomize apps/aulaviva-backend/overlays/dev

      - name: Validate prod overlay
        run: kubectl kustomize apps/aulaviva-backend/overlays/prod
```

Para comprobar la validación:

1. Abrir el repositorio en GitHub.
2. Entrar en la pestaña **Actions**.
3. Seleccionar `GitOps - Kustomize Validation`.
4. Abrir la ejecución más reciente.
5. Comprobar que los pasos `Validate base`, `Validate dev overlay` y `Validate prod overlay` finalizan correctamente.

El workflow no necesita un token GitOps para validar los manifiestos. Tampoco crea aplicaciones de Argo CD ni despliega recursos.

## 9. Desplegar las aplicaciones mediante GitOps

Las aplicaciones Argo CD se definen en el directorio `argocd/`.

### 9.1 Aplicación de desarrollo

El archivo `argocd/aulaviva-backend-dev.yaml` debe contener:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: aulaviva-backend-dev
  namespace: argocd
spec:
  project: default

  source:
    repoURL: https://github.com/merinobranco10/equipo-aulaviva-gitops.git
    targetRevision: main
    path: apps/aulaviva-backend/overlays/dev

  destination:
    server: https://kubernetes.default.svc
    namespace: equipo-aulaviva-dev

  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

Esta aplicación debe conservar la sincronización automática que ya funciona en el entorno de desarrollo.

### 9.2 Aplicación de producción

El archivo `argocd/aulaviva-backend-prod.yaml` debe contener:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: aulaviva-backend-prod
  namespace: argocd
spec:
  project: default

  source:
    repoURL: https://github.com/merinobranco10/equipo-aulaviva-gitops.git
    targetRevision: main
    path: apps/aulaviva-backend/overlays/prod

  destination:
    server: https://kubernetes.default.svc
    namespace: equipo-aulaviva-prod

  syncPolicy:
    syncOptions:
      - CreateNamespace=true
```

La aplicación de producción mantiene la sincronización manual. No se debe agregar una política `automated` si se desea que los cambios de producción requieran una revisión explícita.

### 9.3 Crear las aplicaciones en Argo CD

Desde la raíz de una copia local del repositorio:

```powershell
kubectl apply -f .\argocd\aulaviva-backend-dev.yaml
kubectl apply -f .\argocd\aulaviva-backend-prod.yaml
```

Si `dev` ya existe y funciona, no es necesario volver a aplicarla. Para crear únicamente la aplicación de producción, ejecutar:

```powershell
kubectl apply -f https://raw.githubusercontent.com/merinobranco10/equipo-aulaviva-gitops/main/argocd/aulaviva-backend-prod.yaml
```

El comando anterior descarga el manifiesto desde GitHub y crea la aplicación en el clúster. Requiere que el manifiesto ya esté publicado en la rama `main`.

Comprobar las aplicaciones:

```powershell
kubectl get applications -n argocd
kubectl get application aulaviva-backend-dev -n argocd
kubectl get application aulaviva-backend-prod -n argocd
```

### 9.4 Sincronizar producción manualmente

Una vez creada la aplicación `aulaviva-backend-prod`:

1. Abrir la interfaz web de Argo CD.
2. Seleccionar `aulaviva-backend-prod`.
3. Revisar los recursos y el estado de sincronización.
4. Si el overlay es válido y la imagen es accesible, pulsar **Sync**.
5. Revisar los recursos que Argo CD propone crear o actualizar.
6. Confirmar la sincronización.
7. Esperar a que los recursos estén listos y verificar el estado de salud.

También puede consultarse el estado desde PowerShell:

```powershell
kubectl get application aulaviva-backend-prod -n argocd
kubectl describe application aulaviva-backend-prod -n argocd
```

La opción `CreateNamespace=true` permite a Argo CD crear el namespace de destino durante la sincronización.

**Importante:** registrar una aplicación no equivale a desplegarla. Si `equipo-aulaviva-prod` todavía no tiene recursos, comprobar primero que la sincronización haya comenzado y que Argo CD no presente errores de renderizado, imagen o configuración.

### 9.5 Namespaces

Las aplicaciones deben desplegar sus recursos en los namespaces correspondientes.

La estrategia elegida es utilizar `CreateNamespace=true` en las opciones de sincronización de Argo CD.

No conviene mantener varias herramientas intentando administrar de forma independiente el mismo recurso sin una estrategia definida.

### 9.6 Sincronización y autocorrección

En desarrollo, la aplicación sincroniza automáticamente los cambios, con `prune` y `selfHeal` habilitados.

En producción, la sincronización es manual. Antes de sincronizar cambios, revisar los manifiestos y validar que la imagen y la configuración sean las esperadas.

Para consultar el estado detallado:

```powershell
kubectl describe application aulaviva-backend-dev -n argocd
kubectl describe application aulaviva-backend-prod -n argocd
```

Si el recurso `Application` no se reconoce, comprobar que el CRD de Argo CD esté instalado y que se esté utilizando el contexto correcto.

## 10. Verificar los despliegues

Revisar el estado general del clúster:

```powershell
kubectl get nodes
kubectl get pods -A
kubectl get applications -n argocd
```

Consultar los deployments:

```powershell
kubectl get deployments -n equipo-aulaviva-dev
kubectl get deployments -n equipo-aulaviva-prod
```

Cuando existan los deployments, comprobar el rollout:

```powershell
kubectl rollout status deployment/aulaviva-backend -n equipo-aulaviva-dev
kubectl rollout status deployment/aulaviva-backend -n equipo-aulaviva-prod
```

Consultar las réplicas y métricas:

```powershell
kubectl get hpa -n equipo-aulaviva-dev
kubectl get hpa -n equipo-aulaviva-prod

kubectl top nodes
kubectl top pods -n equipo-aulaviva-dev
kubectl top pods -n equipo-aulaviva-prod
```

Si producción todavía no se ha sincronizado, es posible que el namespace o sus recursos no existan. En ese caso, comprobar primero el estado de la aplicación en Argo CD.

Los pods deben estar listos, los deployments deben tener sus réplicas disponibles y las aplicaciones de Argo CD deben alcanzar el estado de salud esperado.

El HPA requiere métricas disponibles para calcular el uso de CPU y memoria. Si `kubectl top` falla, revisar la instalación de Metrics Server y los eventos del HPA.

### 10.1 Criterios de salud

Una aplicación de Argo CD debe alcanzar `Healthy` cuando sus recursos han llegado a un estado de salud satisfactorio. El estado `Synced` indica que los recursos corresponden a la configuración deseada en Git.

Ambos estados son diferentes:

- `Synced`: el estado aplicado coincide con el estado deseado.
- `Healthy`: los recursos alcanzan el estado de salud esperado.
- `Progressing`: todavía hay recursos progresando hacia su estado esperado.
- `Degraded`: uno o más recursos presentan problemas de salud.

Para la validación final, comprobar `Healthy` y `Synced` en `dev`. En `prod`, verificar ambos estados después de la sincronización manual, siempre que la configuración y las dependencias permitan completar el despliegue.

## 11. Rollback mediante reversión de commit

El rollback permite recuperar una configuración anterior del sistema revirtiendo el commit que introdujo un cambio. En este proyecto, Git es la fuente de verdad y Argo CD sincroniza los recursos de Kubernetes con la configuración declarada en el repositorio.

### 11.1 Identificar el commit

Desde la raíz del repositorio GitOps, consultar el historial:

```bash
git log --oneline
```

Identificar el hash del commit que introdujo el cambio que se desea revertir.

### 11.2 Revertir el cambio

Ejecutar:

```bash
git revert <HASH_COMMIT>
```

Reemplazar `<HASH_COMMIT>` por el hash real del commit que se desea revertir. Este comando crea un nuevo commit que deshace los cambios seleccionados sin reescribir el historial.

### 11.3 Publicar la reversión

Subir el nuevo commit al repositorio remoto:

```bash
git push origin main
```

### 11.4 Verificar la recuperación

Consultar el estado de las aplicaciones Argo CD:

```powershell
kubectl get applications -n argocd
```

Comprobar los deployments del entorno afectado:

```powershell
kubectl get deployments -n equipo-aulaviva-dev
kubectl get deployments -n equipo-aulaviva-prod
```

Verificar que la configuración del recurso afectado corresponda al estado anterior esperado.

### 11.5 Consideraciones por entorno

- **Desarrollo (`dev`):** después de publicar la reversión en `main`, Argo CD debe detectar el cambio y sincronizarlo automáticamente, según la política configurada.
- **Producción (`prod`):** la reversión debe revisarse y sincronizarse manualmente.

### 11.6 Evidencia del rollback

Para documentar una prueba de rollback:

1. Registrar el SHA del commit que se desea revertir.
2. Ejecutar `git revert <SHA_DEL_COMMIT>` en una copia local del repositorio.
3. Revisar y publicar la reversión en `main`.
4. Comprobar que Argo CD detecta el nuevo commit.
5. Verificar que el recurso vuelve a la configuración anterior.
6. Registrar el estado de sincronización y salud de la aplicación.

La prueba solo se considera ejecutada cuando existe evidencia de la reversión y de la reconciliación correspondiente.

La reversión debe realizarse en Git, que constituye la fuente de verdad del despliegue. No se deben borrar recursos manualmente como sustituto del rollback GitOps.

## 12. Acceder a las aplicaciones mediante Ingress

Los hosts definidos para el proyecto son:

- Desarrollo: `dev.aulaviva.local`
- Producción: `aulaviva.local`

En Windows, si el clúster fue creado con los mapeos de puertos indicados anteriormente, agregar estas entradas al archivo `hosts`, ubicado en:

`C:\Windows\System32\drivers\etc\hosts`

```text
127.0.0.1 dev.aulaviva.local
127.0.0.1 aulaviva.local
```

Es posible que se requieran permisos de administrador para editar el archivo.

Verificar los recursos Ingress:

```powershell
kubectl get ingress -A
kubectl describe ingress aulaviva-ingress -n equipo-aulaviva-dev
kubectl describe ingress aulaviva-ingress -n equipo-aulaviva-prod
```

Después, probar desde el navegador:

- `http://dev.aulaviva.local/`
- `http://aulaviva.local/`

Estas direcciones dependen de que el controlador Ingress esté listo, los puertos estén mapeados y los recursos Ingress apunten a los servicios correctos.

Si no funcionan, comprobar el estado del controlador, la configuración de red del clúster y los endpoints del servicio de destino.

## 13. Imagen de contenedor y GHCR

El backend se despliega utilizando una imagen publicada en GitHub Container Registry (GHCR).

Se recomienda utilizar una etiqueta de versión inmutable o, preferentemente, fijar el digest de la imagen en los manifiestos para que el despliegue sea reproducible.

Ejemplo:

```text
ghcr.io/merinobranco10/equipo-aulaviva-backend@sha256:<DIGEST>
```

Reemplazar `<DIGEST>` por el digest real de la imagen publicada. No se debe utilizar literalmente ese marcador.

Los overlays deben hacer referencia a una imagen que exista y que sea compatible con la configuración del backend. Si se actualiza el digest, validar nuevamente ambos overlays y revisar el resultado en Argo CD.

Si el paquete GHCR es privado, Kubernetes necesita credenciales válidas mediante un `imagePullSecret` configurado en el namespace correspondiente. No almacenar tokens ni contraseñas en texto plano en el repositorio GitOps.

## 14. Solución de problemas

### 14.1 Los pods no inician

```powershell
kubectl get pods -A
kubectl describe pod <NOMBRE_POD> -n <NAMESPACE>
kubectl logs <NOMBRE_POD> -n <NAMESPACE>
```

Revisar eventos, imagen configurada, permisos de descarga y configuración de la aplicación.

### 14.2 Argo CD muestra `Progressing` o `Degraded`

```powershell
kubectl get applications -n argocd
kubectl describe application aulaviva-backend-dev -n argocd
kubectl describe application aulaviva-backend-prod -n argocd
```

Inspeccionar el recurso que no alcanza el estado saludable. Que una aplicación esté `Synced` no garantiza que todos sus recursos estén saludables.

### 14.3 Producción no tiene recursos

Si el comando:

```powershell
kubectl get all -n equipo-aulaviva-prod
```

devuelve `No resources found`, comprobar primero que la aplicación exista:

```powershell
kubectl get application aulaviva-backend-prod -n argocd
```

Revisar su estado y los mensajes de error:

```powershell
kubectl describe application aulaviva-backend-prod -n argocd
```

Si la aplicación todavía no existe, aplicar su manifiesto. Si existe pero no se ha sincronizado, revisar el overlay y ejecutar una sincronización manual desde Argo CD.

No crear los deployments manualmente como sustituto de la aplicación GitOps: los recursos deben generarse a partir de los manifiestos versionados.

### 14.4 El controlador Ingress no está listo

```powershell
kubectl get pods -n ingress-nginx
kubectl describe pod -n ingress-nginx <NOMBRE_POD>
kubectl get nodes --show-labels
kubectl get ingressclass
```

Comprobar que el nodo de `kind` tenga la etiqueta `ingress-ready=true` y que exista la clase `nginx`.

### 14.5 No hay métricas para el HPA

```powershell
kubectl get apiservice v1beta1.metrics.k8s.io
kubectl top nodes
kubectl top pods -A
kubectl describe hpa -n equipo-aulaviva-dev
```

Si Metrics Server presenta errores TLS al consultar kubelet en el clúster local, investigar los certificados y la configuración de la instalación. No utilizar la desactivación de la verificación TLS como configuración de producción.

### 14.6 El navegador no llega al backend

```powershell
kubectl get ingress -A
kubectl get svc -A
kubectl get endpoints -A
kubectl get pods -n equipo-aulaviva-dev
kubectl get pods -n equipo-aulaviva-prod
```

Comprobar el host, el puerto, el servicio de destino, los mapeos de puertos de `kind` y las entradas del archivo `hosts`.

## 15. Buenas prácticas de seguridad

- No versionar secretos, tokens ni contraseñas.
- Usar imágenes versionadas y fijadas por digest.
- Aplicar permisos mínimos a los componentes y usuarios.
- Mantener separados los namespaces de desarrollo y producción.
- Revisar los cambios antes de sincronizarlos en producción.
- No utilizar configuraciones TLS inseguras fuera del entorno local de pruebas.
- Mantener las versiones de las herramientas documentadas y actualizadas.
- Proteger el acceso a Argo CD y a los paquetes privados de GHCR.
- Validar los manifiestos antes de aplicar cambios al clúster.
- Utilizar Git como fuente de verdad para los cambios y las reversiones.

## 16. Limitaciones del entorno local

El entorno `kind` sirve para desarrollar, probar y demostrar el flujo GitOps. No representa por sí solo una infraestructura de producción de alta disponibilidad.

En un entorno real deben evaluarse la disponibilidad del clúster, el almacenamiento persistente, la gestión de secretos, los certificados TLS, el monitoreo, las copias de seguridad, la seguridad de red y los recursos necesarios para soportar la carga.

Los estados `Healthy` y `Synced` demuestran que los recursos se encuentran saludables según las comprobaciones de Argo CD y que coinciden con el estado declarado en Git. No sustituyen las pruebas funcionales, de seguridad o de rendimiento.

## 17. Evidencias para la entrega

Para demostrar el cumplimiento de los objetivos, guardar capturas de pantalla de los siguientes elementos:

1. Estructura del repositorio GitOps separado.
2. Workflow `GitOps - Kustomize Validation` con ejecución exitosa.
3. Aplicación `aulaviva-backend-dev` en estado `Healthy` y `Synced`.
4. Aplicación `aulaviva-backend-prod` registrada en Argo CD.
5. Política de sincronización manual de producción.
6. Estado saludable y sincronizado de producción después de su sincronización manual, si se ha completado el despliegue.
7. Recursos Kubernetes y pods listos de los entornos que se hayan desplegado.
8. README con instrucciones de validación y rollback.
9. Evidencia de rollback si la pauta exige ejecutar una prueba real.

No afirmar que producción está desplegada o saludable hasta que se haya comprobado en el clúster.

## 18. Referencias oficiales

- Docker Desktop: https://docs.docker.com/desktop/
- kind: https://kind.sigs.k8s.io/
- Kubernetes: https://kubernetes.io/docs/
- Kustomize: https://kubectl.docs.kubernetes.io/references/kustomize/
- Argo CD: https://argo-cd.readthedocs.io/
- Ingress NGINX: https://kubernetes.github.io/ingress-nginx/
- GitHub Container Registry: https://docs.github.com/en/packages/working-with-a-github-packages-registry/working-with-the-container-registry
- GitHub Actions: https://docs.github.com/en/actions
