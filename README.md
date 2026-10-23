# AulaViva GitOps

Repositorio GitOps de despliegue del backend de **AulaViva**, una plataforma educativa que utiliza Kubernetes para gestionar los despliegues en los entornos de desarrollo (`dev`) y producción (`prod`).

La configuración utiliza Argo CD para sincronizar los recursos de Kubernetes desde Git, Kustomize para organizar los manifiestos por entorno, GitHub Actions para validar la configuración y `kind` para ejecutar un clúster Kubernetes local.

## 1. Objetivos

- Automatizar la entrega de aplicaciones mediante GitOps.
- Mantener separados los entornos de desarrollo y producción.
- Gestionar los manifiestos de Kubernetes mediante Kustomize.
- Permitir que Argo CD detecte y sincronice cambios del repositorio.
- Validar automáticamente la estructura Kustomize mediante GitHub Actions.
- Documentar un procedimiento de rollback mediante reversión de commit.
- Facilitar la reproducción del entorno local por parte de otros integrantes del equipo.

## 2. Arquitectura

El flujo de despliegue es el siguiente:

1. El código del backend se construye y se publica como imagen de contenedor en GitHub Container Registry (GHCR).
2. El repositorio GitOps mantiene los manifiestos de Kubernetes y las configuraciones de cada entorno.
3. GitHub Actions valida la construcción de Kustomize para la base y ambos overlays.
4. Argo CD observa el repositorio GitOps y compara el estado deseado con los recursos del clúster.
5. Kubernetes ejecuta los pods del backend y administra sus réplicas.
6. Ingress NGINX dirige las solicitudes HTTP según el host configurado.
7. El Horizontal Pod Autoscaler (HPA) ajusta la cantidad de réplicas según las métricas disponibles y los límites definidos.

### Componentes principales

| Componente | Función |
|---|---|
| Docker | Construcción y empaquetado de contenedores. |
| kind | Clúster Kubernetes local. |
| kubectl | Administración del clúster. |
| Kustomize | Personalización de manifiestos por entorno. |
| GitHub Actions | Validación automática de los manifiestos. |
| Argo CD | Sincronización GitOps. |
| Ingress NGINX | Enrutamiento del tráfico HTTP/HTTPS. |
| GHCR | Almacenamiento de imágenes del backend. |
| HPA | Escalado horizontal de pods según métricas. |

## 3. Estructura del repositorio

La estructura principal del repositorio es:

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

La estructura puede ampliarse a medida que se incorporen nuevos servicios o entornos.

- `base/`: contiene los recursos compartidos de Kubernetes.
- `overlays/dev/`: personaliza la configuración de desarrollo.
- `overlays/prod/`: personaliza la configuración de producción.
- `argocd/`: contiene las declaraciones de las aplicaciones Argo CD.
- `.github/workflows/`: contiene la validación automática de Kustomize.
- `infra/kind/`: contiene la configuración del clúster local.

## 4. Entornos

| Característica | Desarrollo (`dev`) | Producción (`prod`) |
|---|---|---|
| Aplicación Argo CD | `aulaviva-backend-dev` | `aulaviva-backend-prod` |
| Namespace | `equipo-aulaviva-dev` | `equipo-aulaviva-prod` |
| Overlay Kustomize | `apps/aulaviva-backend/overlays/dev` | `apps/aulaviva-backend/overlays/prod` |
| Sincronización | Automática | Manual |
| `prune` y `selfHeal` | Habilitados | No habilitados en la configuración actual |
| Host Ingress | `dev.aulaviva.local` | `prod.aulaviva.local` |

Existe además un Ingress previo en el namespace `equipo-aulaviva` con el host `aulaviva.local`. Se conserva separado de las aplicaciones GitOps de desarrollo y producción.

Los parámetros específicos, como el número de réplicas y los límites del HPA, se definen en los manifiestos de cada entorno.

## 5. Requisitos previos

Instalar las siguientes herramientas:

- [Docker Desktop](https://www.docker.com/products/docker-desktop/)
- [kubectl](https://kubernetes.io/docs/tasks/tools/)
- [kind](https://kind.sigs.k8s.io/docs/user/quick-start/)
- [Git](https://git-scm.com/downloads)

En Windows, iniciar Docker Desktop y asegurarse de que el motor de contenedores Linux esté funcionando.

Verificar las instalaciones desde PowerShell:

```powershell
docker version
kubectl version --client
kind version
git --version
```

### 5.1 Clonar el repositorio

```powershell
git clone https://github.com/merinobranco10/equipo-aulaviva-gitops.git
cd equipo-aulaviva-gitops
```

Comprobar la rama actual y el estado del repositorio:

```powershell
git branch --show-current
git status
```

Los comandos siguientes deben ejecutarse desde la raíz del repositorio, salvo que se indique lo contrario.

## 6. Crear el clúster Kubernetes local

Esta sección corresponde a una instalación desde cero. Si el clúster `aulaviva` ya existe y contiene recursos, no lo elimines ni lo recrees solo para seguir esta guía.

El archivo de configuración del clúster se encuentra en:

`infra/kind/cluster-config.yaml`

Una configuración compatible con el entorno local puede utilizar el siguiente contenido:

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

La etiqueta `ingress-ready=true` permite programar el controlador Ingress en el nodo. Los mapeos de puertos permiten exponer el tráfico HTTP y HTTPS al equipo anfitrión.

Para crear el clúster desde cero:

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

Si el clúster ya existe, basta con comprobarlo:

```powershell
kind get clusters
kubectl get nodes
kubectl config current-context
```

## 7. Instalar Ingress NGINX

Ingress NGINX permite dirigir solicitudes HTTP/HTTPS hacia los servicios de Kubernetes según el host definido en cada recurso Ingress.

Para una instalación nueva compatible con `kind`, se puede aplicar el manifiesto oficial:

```powershell
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/kind/deploy.yaml
```

Esperar a que el controlador esté disponible:

```powershell
kubectl wait --namespace ingress-nginx --for=condition=Ready pod --selector=app.kubernetes.io/component=controller --timeout=180s
```

Comprobar la instalación:

```powershell
kubectl get pods -n ingress-nginx
kubectl get ingressclass
```

Debe aparecer la clase `nginx` y el pod controlador debe estar listo.

La instalación anterior crea los recursos del namespace `ingress-nginx`; no es necesario crearlo manualmente antes de aplicar el manifiesto.

Si el controlador ya está instalado y funcionando, no es necesario repetir la instalación.

## 8. Instalar Argo CD

Esta sección se ejecuta solo cuando Argo CD no está instalado.

Crear el namespace:

```powershell
kubectl create namespace argocd
```

Instalar Argo CD:

```powershell
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

Si el namespace ya existe, no es necesario crearlo otra vez.

Esperar a que los deployments estén disponibles:

```powershell
kubectl wait --for=condition=Available deployment --all -n argocd --timeout=300s
```

Verificar los componentes:

```powershell
kubectl get pods -n argocd
kubectl get svc -n argocd
```

Para una instalación reproducible a largo plazo, se recomienda fijar una versión concreta de Argo CD en lugar de depender permanentemente del manifiesto `stable`.

### 8.1 Acceder a la interfaz web

Ejecutar el siguiente comando y mantener la terminal abierta:

```powershell
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

Abrir en el navegador:

`https://localhost:8080`

En el entorno local puede aparecer una advertencia de certificado.

### 8.2 Obtener la contraseña inicial

En otra terminal de PowerShell:

```powershell
$secret = kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}"

[System.Text.Encoding]::UTF8.GetString(
  [System.Convert]::FromBase64String($secret)
)
```

Iniciar sesión con el usuario `admin` y la contraseña obtenida.

La contraseña es inicial. Se recomienda cambiarla después del primer acceso y no incluirla en archivos versionados ni capturas públicas.

## 9. Validación automática de Kustomize

El repositorio dispone del workflow:

`.github/workflows/kustomize-validate.yml`

Este workflow se ejecuta cuando se modifican los manifiestos bajo `apps/` o el propio archivo del workflow en la rama `main`, tanto en `push` como en `pull_request`. También permite ejecución manual mediante `workflow_dispatch`.

El workflow ejecuta los siguientes pasos:

1. Descarga el repositorio.
2. Configura `kubectl`.
3. Construye la base mediante Kustomize.
4. Construye el overlay de desarrollo.
5. Construye el overlay de producción.

Los comandos de validación son:

```bash
kubectl kustomize apps/aulaviva-backend/base
kubectl kustomize apps/aulaviva-backend/overlays/dev
kubectl kustomize apps/aulaviva-backend/overlays/prod
```

Para validar localmente, ejecutar los comandos desde la raíz del repositorio clonado.

El workflow comprueba que Kustomize pueda construir los manifiestos. No despliega recursos en el clúster ni reemplaza las verificaciones de salud de Argo CD.

### 9.1 Consultar las ejecuciones

Abrir la pestaña Actions del repositorio:

[GitHub Actions — AulaViva GitOps](https://github.com/merinobranco10/equipo-aulaviva-gitops/actions)

La ejecución correspondiente debe terminar correctamente para confirmar que las tres rutas pueden construirse sin errores.

## 10. Desplegar las aplicaciones mediante GitOps

Las aplicaciones declaradas son:

- `aulaviva-backend-dev`
- `aulaviva-backend-prod`

Ambas apuntan al repositorio GitOps, rama `main`, pero utilizan overlays y namespaces diferentes.

### 10.1 Aplicar las declaraciones de Argo CD

Desde la raíz del repositorio:

```powershell
kubectl apply -f .\argocd\aulaviva-backend-dev.yaml
kubectl apply -f .\argocd\aulaviva-backend-prod.yaml
```

Estos comandos crean o actualizan las declaraciones `Application` en Argo CD. Los recursos del backend son gestionados posteriormente por Argo CD según la configuración de cada aplicación.

### 10.2 Verificar las aplicaciones

```powershell
kubectl get applications -n argocd
```

El estado esperado una vez finalizada la sincronización es:

```text
NAME                   SYNC STATUS   HEALTH STATUS
aulaviva-backend-dev   Synced        Healthy
aulaviva-backend-prod  Synced        Healthy
```

Los estados pueden variar durante una operación de sincronización o si algún recurso presenta problemas.

### 10.3 Verificar desarrollo

```powershell
kubectl get all -n equipo-aulaviva-dev
kubectl get ingress -n equipo-aulaviva-dev
kubectl get hpa -n equipo-aulaviva-dev
```

### 10.4 Verificar producción

```powershell
kubectl get all -n equipo-aulaviva-prod
kubectl get ingress -n equipo-aulaviva-prod
kubectl get hpa -n equipo-aulaviva-prod
```

### 10.5 Gestión de namespaces

Las aplicaciones utilizan la opción `CreateNamespace=true` en su configuración de sincronización, de modo que Argo CD puede crear los namespaces de destino cuando sea necesario.

Los namespaces de los entornos son:

- `equipo-aulaviva-dev`
- `equipo-aulaviva-prod`

Se recomienda mantener una estrategia definida para la gestión de namespaces y evitar que distintas herramientas administren independientemente los mismos recursos.

### 10.6 Política de sincronización

En desarrollo, la aplicación está configurada para sincronizar automáticamente los cambios, con `prune` y `selfHeal` habilitados.

- `prune`: permite eliminar recursos gestionados por Argo CD que hayan sido retirados del estado deseado en Git.
- `selfHeal`: permite corregir cambios manuales en el clúster para restablecer el estado declarado en Git.

En producción, la sincronización es manual según la configuración actual. Los cambios deben revisarse antes de sincronizarlos.

Para consultar el estado detallado:

```powershell
kubectl describe application aulaviva-backend-dev -n argocd
kubectl describe application aulaviva-backend-prod -n argocd
```

## 11. Verificar los despliegues

Revisar el estado general:

```powershell
kubectl get nodes
kubectl get pods -A
kubectl get applications -n argocd
```

Comprobar los deployments:

```powershell
kubectl rollout status deployment/aulaviva-backend -n equipo-aulaviva-dev
kubectl rollout status deployment/aulaviva-backend -n equipo-aulaviva-prod
```

Consultar las réplicas y métricas:

```powershell
kubectl get deployments -n equipo-aulaviva-dev
kubectl get deployments -n equipo-aulaviva-prod

kubectl get hpa -n equipo-aulaviva-dev
kubectl get hpa -n equipo-aulaviva-prod

kubectl top nodes
kubectl top pods -n equipo-aulaviva-dev
kubectl top pods -n equipo-aulaviva-prod
```

Los pods deben estar listos, los deployments deben tener sus réplicas disponibles y las aplicaciones de Argo CD deben presentar el estado de salud esperado.

El HPA requiere métricas disponibles para calcular el uso de CPU y memoria. Si `kubectl top` falla, se debe revisar la instalación de Metrics Server y los eventos del HPA.

Que una aplicación esté `Synced` no garantiza por sí solo que todas las peticiones HTTP al backend funcionen correctamente.

## 12. Acceder a las aplicaciones mediante Ingress

Los hosts utilizados por las aplicaciones GitOps son:

- Desarrollo: `dev.aulaviva.local`
- Producción: `prod.aulaviva.local`

Existe además un Ingress previo en el namespace `equipo-aulaviva` que utiliza `aulaviva.local`. Se conserva separado para evitar conflictos de host y ruta con los Ingress de desarrollo y producción.

En Windows, editar como administrador el archivo:

`C:\Windows\System32\drivers\etc\hosts`

Agregar las entradas:

```text
127.0.0.1 dev.aulaviva.local
127.0.0.1 prod.aulaviva.local
```

Si también se necesita acceder al Ingress anterior, se puede conservar la entrada:

```text
127.0.0.1 aulaviva.local
```

Verificar los recursos:

```powershell
kubectl get ingress -A
kubectl describe ingress aulaviva-ingress -n equipo-aulaviva-dev
kubectl describe ingress aulaviva-ingress -n equipo-aulaviva-prod
```

Probar desde el navegador:

- `http://dev.aulaviva.local/`
- `http://prod.aulaviva.local/`

Estas direcciones dependen de que el controlador Ingress esté listo, los puertos estén mapeados y los recursos Ingress apunten a los servicios correctos.

Si no funcionan, comprobar el estado del controlador, los endpoints del servicio, las entradas del archivo `hosts` y la configuración de red del clúster.

## 13. Imagen de contenedor y GHCR

El backend se despliega utilizando una imagen publicada en GitHub Container Registry (GHCR).

La referencia configurada actualmente en los manifiestos utiliza el digest de la imagen, por ejemplo:

```text
ghcr.io/merinobranco10/equipo-aulaviva-backend@sha256:c914d8112cac2cafed01b61682459d870bea657aecf2bc531b357f36e29bb356
```

El digest identifica una versión concreta de la imagen y permite que el despliegue sea reproducible.

Cuando se publique una nueva imagen, debe actualizarse la referencia correspondiente en el repositorio GitOps mediante el procedimiento de promoción definido por el equipo.

Si el paquete GHCR es privado, Kubernetes necesita credenciales válidas mediante un `imagePullSecret` configurado en el namespace correspondiente. No almacenar tokens ni contraseñas en texto plano en el repositorio GitOps.

## 14. Rollback mediante reversión de commit

El rollback permite recuperar una configuración anterior revirtiendo el commit que introdujo un cambio.

En este proyecto, Git es la fuente de verdad. Argo CD compara el estado deseado del repositorio con los recursos de Kubernetes y aplica los cambios de acuerdo con la política de sincronización de cada aplicación.

### 14.1 Identificar el commit

Desde la raíz del repositorio GitOps, consultar el historial:

```bash
git log --oneline
```

Identificar el hash del commit que introdujo el cambio que se desea revertir.

Antes de continuar, comprobar que la rama de trabajo sea `main` y que no existan cambios locales sin guardar:

```bash
git branch --show-current
git status
```

### 14.2 Revertir el cambio

Ejecutar:

```bash
git revert <HASH_COMMIT>
```

Reemplazar `<HASH_COMMIT>` por el hash real del commit que se desea revertir.

Este comando crea un nuevo commit que deshace los cambios seleccionados sin reescribir el historial. Si aparecen conflictos, deben resolverse antes de completar la reversión.

### 14.3 Publicar la reversión

Después de verificar el resultado local, publicar el nuevo commit:

```bash
git push origin main
```

La reversión debe revisarse de acuerdo con las prácticas de colaboración del equipo y las protecciones configuradas en GitHub.

### 14.4 Verificar la recuperación

Consultar el estado de las aplicaciones:

```powershell
kubectl get applications -n argocd
```

En desarrollo, Argo CD debe detectar la nueva revisión y sincronizarla automáticamente, de acuerdo con su política configurada.

En producción, revisar el cambio y ejecutar una sincronización manual desde la interfaz de Argo CD si la aplicación permanece `OutOfSync`.

Comprobar los recursos de los entornos:

```powershell
kubectl get deployments -n equipo-aulaviva-dev
kubectl get deployments -n equipo-aulaviva-prod

kubectl get ingress -n equipo-aulaviva-dev
kubectl get ingress -n equipo-aulaviva-prod
```

También se debe verificar que la configuración del recurso modificado corresponda al estado anterior esperado.

El rollback se considera completado cuando:

1. El commit de reversión está publicado en GitHub.
2. Argo CD ha detectado la revisión.
3. La aplicación afectada ha aplicado el estado deseado correspondiente.
4. Los recursos afectados presentan el estado esperado.

### 14.5 Consideraciones por entorno

- **Desarrollo (`dev`):** la reversión debe sincronizarse automáticamente según la política configurada.
- **Producción (`prod`):** la reversión debe revisarse y sincronizarse manualmente según la política actual.

No se recomienda utilizar cambios manuales en Kubernetes como mecanismo principal de rollback, ya que Git debe seguir siendo la fuente de verdad.

## 15. Solución de problemas

### 15.1 Los comandos Kustomize indican que la ruta no existe

Comprobar que la terminal esté en la raíz del repositorio:

```powershell
Get-Location
Get-ChildItem
```

Después ejecutar:

```powershell
kubectl kustomize apps/aulaviva-backend/base
kubectl kustomize apps/aulaviva-backend/overlays/dev
kubectl kustomize apps/aulaviva-backend/overlays/prod
```

Si se ejecutan desde otro directorio, las rutas relativas pueden fallar.

### 15.2 Los pods no inician

```powershell
kubectl get pods -A
kubectl describe pod <NOMBRE_POD> -n <NAMESPACE>
kubectl logs <NOMBRE_POD> -n <NAMESPACE>
```

Revisar eventos, imagen configurada, permisos de descarga y configuración de la aplicación.

### 15.3 Argo CD muestra `Progressing`, `Degraded` u `OutOfSync`

```powershell
kubectl get applications -n argocd
kubectl describe application aulaviva-backend-dev -n argocd
kubectl describe application aulaviva-backend-prod -n argocd
```

Inspeccionar el recurso que no alcanza el estado saludable. Si la operación de sincronización falla, revisar el mensaje de error completo.

### 15.4 Conflicto de host en Ingress

NGINX puede rechazar un Ingress si otro recurso utiliza el mismo host y la misma ruta.

Consultar los recursos:

```powershell
kubectl get ingress -A
```

Los hosts de desarrollo y producción deben mantenerse separados:

- `dev.aulaviva.local`
- `prod.aulaviva.local`

No crear manualmente recursos fuera de GitOps como solución permanente. Corregir el manifiesto del repositorio y permitir que Argo CD aplique el estado deseado.

### 15.5 El controlador Ingress no está disponible

```powershell
kubectl get pods -n ingress-nginx
kubectl describe pod -n ingress-nginx <NOMBRE_POD>
kubectl get nodes --show-labels
kubectl get ingressclass
```

Comprobar que el controlador esté listo, que la clase `nginx` exista y que el clúster tenga la configuración de red necesaria.

### 15.6 No hay métricas para el HPA

```powershell
kubectl get apiservice v1beta1.metrics.k8s.io
kubectl top nodes
kubectl top pods -A
kubectl describe hpa aulaviva-backend-hpa -n equipo-aulaviva-dev
kubectl describe hpa aulaviva-backend-hpa -n equipo-aulaviva-prod
```

Si Metrics Server presenta errores TLS al consultar kubelet en el clúster local, revisar los certificados y la configuración de la instalación. No desactivar la verificación TLS como configuración de producción.

### 15.7 El navegador no llega al backend

```powershell
kubectl get ingress -A
kubectl get svc -A
kubectl get endpoints -A
kubectl get pods -n equipo-aulaviva-dev
kubectl get pods -n equipo-aulaviva-prod
```

Comprobar el host, el puerto, el servicio de destino, los endpoints, los mapeos de puertos de `kind` y las entradas del archivo `hosts`.

## 16. Buenas prácticas de seguridad

- No versionar secretos, tokens ni contraseñas.
- Utilizar imágenes versionadas y fijadas por digest.
- Aplicar permisos mínimos a los componentes y usuarios.
- Mantener separados los namespaces de desarrollo y producción.
- Revisar los cambios antes de sincronizarlos en producción.
- No utilizar configuraciones TLS inseguras fuera del entorno local de pruebas.
- Mantener las versiones de las herramientas documentadas y actualizadas.
- Proteger el acceso a Argo CD y a los paquetes privados de GHCR.
- Revisar cuidadosamente las operaciones que eliminan recursos mediante `prune`.

## 17. Limitaciones del entorno local

El entorno `kind` sirve para desarrollar, probar y demostrar el flujo GitOps. No representa por sí solo una infraestructura de producción de alta disponibilidad.

En un entorno real deben evaluarse la disponibilidad del clúster, el almacenamiento persistente, la gestión de secretos, los certificados TLS, el monitoreo, las copias de seguridad, la seguridad de red y los recursos necesarios para soportar la carga.

La validación de Kustomize confirma que los manifiestos se pueden construir, pero no garantiza por sí sola la disponibilidad funcional del backend.

## 18. Evidencias para la entrega

Para demostrar el cumplimiento de la actividad, se recomienda conservar las siguientes evidencias:

1. Repositorio GitOps separado del repositorio del backend.
2. Estructura de carpetas `base`, `overlays/dev` y `overlays/prod`.
3. Ejecución exitosa del workflow `GitOps - Kustomize Validation`.
4. Aplicaciones `aulaviva-backend-dev` y `aulaviva-backend-prod` en Argo CD con estado `Synced` y salud `Healthy`.
5. Salida de `kubectl get ingress -A` con los hosts separados.
6. Commit que introdujo un cambio en Git y evidencia de que el cambio se reflejó en Kubernetes.
7. Procedimiento de rollback documentado en esta sección.

Si la evaluación exige demostrar un rollback real, realizar una prueba controlada sobre un cambio no crítico y documentar el commit de reversión y la sincronización posterior.

## 19. Referencias oficiales

- Docker Desktop: https://docs.docker.com/desktop/
- kind: https://kind.sigs.k8s.io/
- Kubernetes: https://kubernetes.io/docs/
- Kustomize: https://kubectl.docs.kubernetes.io/references/kustomize/
- Argo CD: https://argo-cd.readthedocs.io/
- Ingress NGINX: https://kubernetes.github.io/ingress-nginx/
- GitHub Container Registry: https://docs.github.com/en/packages/working-with-a-github-packages-registry/working-with-the-container-registry
