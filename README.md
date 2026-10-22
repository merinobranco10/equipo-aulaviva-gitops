# AulaViva GitOps

Repositorio de despliegue GitOps del backend de **AulaViva**, una plataforma educativa que utiliza Kubernetes para gestionar sus despliegues en los entornos de desarrollo (`dev`) y producción (`prod`).

La configuración utiliza Argo CD para sincronizar los recursos de Kubernetes desde Git, Kustomize para organizar los manifiestos por entorno y `kind` para ejecutar un clúster Kubernetes local.

## 1. Objetivos

- Automatizar la entrega de aplicaciones mediante GitOps.
- Mantener separados los entornos de desarrollo y producción.
- Gestionar los manifiestos de Kubernetes mediante Kustomize.
- Permitir que Argo CD detecte y sincronice cambios del repositorio.
- Facilitar la reproducción del entorno local por parte de otros integrantes del equipo.

## 2. Arquitectura

El flujo de despliegue es el siguiente:

1. El código del backend se construye y se publica como imagen de contenedor en GitHub Container Registry (GHCR).
2. El repositorio GitOps mantiene los manifiestos y las configuraciones de cada entorno.
3. Argo CD observa el repositorio y sincroniza los recursos declarados.
4. Kubernetes ejecuta los pods del backend y administra sus réplicas.
5. Ingress NGINX permite acceder a las aplicaciones mediante los nombres de host configurados.

### Componentes principales

| Componente    | Función                                     |
| ------------- | ------------------------------------------- |
| Docker        | Construcción y ejecución de contenedores.   |
| kind          | Clúster Kubernetes local.                   |
| kubectl       | Administración del clúster.                 |
| Kustomize     | Personalización de manifiestos por entorno. |
| Argo CD       | Sincronización GitOps.                      |
| Ingress NGINX | Enrutamiento del tráfico HTTP/HTTPS.        |
| GHCR          | Almacenamiento de imágenes del backend.     |
| HPA           | Escalado horizontal de pods según métricas. |

## 3. Entornos

| Característica     | Desarrollo (`dev`)                                 | Producción (`prod`)                   |
| ------------------ | -------------------------------------------------- | ------------------------------------- |
| Aplicación Argo CD | `aulaviva-backend-dev`                             | `aulaviva-backend-prod`               |
| Namespace          | `equipo-aulaviva-dev`                              | `equipo-aulaviva-prod`                |
| Overlay Kustomize  | `apps/aulaviva-backend/overlays/dev`               | `apps/aulaviva-backend/overlays/prod` |
| Sincronización     | Automática, con poda y autocorrección configuradas | Manual, según la configuración actual |
| Host Ingress       | `dev.aulaviva.local`                               | `aulaviva.local`                      |

Los parámetros específicos, como el número de réplicas y los límites del HPA, se definen en los manifiestos de cada entorno.

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
git clone https://github.com/matiasdiaz434/equipo-aulaViva-gitops.git

cd equipo-aulaViva-gitops
```

## 5. Crear el clúster Kubernetes local

> Esta sección corresponde a una instalación desde cero. Si el clúster `aulaviva` ya existe y contiene recursos, no ejecutes nuevamente la creación ni elimines el clúster solo para seguir este README.

Se recomienda mantener en el repositorio el archivo `kind/cluster-config.yaml` con una configuración equivalente a la siguiente:

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

La etiqueta `ingress-ready=true` permite que el controlador Ingress utilizado en esta guía se programe en el nodo. Los mapeos de puertos permiten exponer el tráfico HTTP y HTTPS al equipo anfitrión.

Crear el clúster:

```powershell
kind create cluster --name aulaviva --config .\kind\cluster-config.yaml
```

Verificar el clúster:

```powershell
kubectl config use-context kind-aulaviva
kubectl get nodes
kubectl cluster-info
```

El nodo debe aparecer en estado `Ready`.

Si el clúster ya existe, basta con verificarlo:

```powershell
kind get clusters
kubectl get nodes
kubectl config current-context
```

## 6. Instalar Ingress NGINX

Ingress NGINX permite dirigir solicitudes HTTP/HTTPS hacia los servicios internos de Kubernetes según el host definido en cada recurso Ingress.

Instalar el controlador compatible con `kind`:

```powershell
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.12.1/deploy/static/provider/kind/deploy.yaml
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

Debe aparecer la clase `nginx` y el pod controlador debe alcanzar el estado `Running` y estar listo.

**Nota:** la instalación anterior crea los recursos del namespace `ingress-nginx`; no es necesario crearlo manualmente antes de aplicar ese manifiesto.

## 7. Instalar Argo CD

Crear el namespace e instalar Argo CD:

```powershell
kubectl create namespace argocd

kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

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

Para un procedimiento reproducible a largo plazo, se recomienda fijar una versión concreta de Argo CD en lugar de utilizar permanentemente el manifiesto `stable`.

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

Iniciar sesión con el usuario `admin` y la contraseña obtenida. Esta contraseña es inicial: se recomienda cambiarla después del primer acceso y no incluirla en archivos versionados ni capturas públicas.

## 8. Desplegar las aplicaciones mediante GitOps

Argo CD debe tener declaradas las aplicaciones `aulaviva-backend-dev` y `aulaviva-backend-prod`, con sus respectivos directorios de origen y namespaces de destino.

Antes de aplicarlas, revisar la estructura real del repositorio. Si los manifiestos de las aplicaciones están almacenados en `argocd/`, los comandos serán:

```powershell
kubectl apply -f .\argocd\aulaviva-backend-dev.yaml
kubectl apply -f .\argocd\aulaviva-backend-prod.yaml
```

Si los archivos están en otra ubicación, ajustar las rutas.

Verificar las aplicaciones:

```powershell
kubectl get applications -n argocd
```

Comprobar los recursos de desarrollo:

```powershell
kubectl get all -n equipo-aulaviva-dev
kubectl get ingress -n equipo-aulaviva-dev
kubectl get hpa -n equipo-aulaviva-dev
```

Comprobar los recursos de producción:

```powershell
kubectl get all -n equipo-aulaviva-prod
kubectl get ingress -n equipo-aulaviva-prod
kubectl get hpa -n equipo-aulaviva-prod
```

### 8.1 Namespaces

Las aplicaciones deben desplegar sus recursos en los namespaces correspondientes. Se debe escoger una única estrategia de gestión:

- Declarar los namespaces en los manifiestos del repositorio.
- Configurar `CreateNamespace=true` en las opciones de sincronización de cada aplicación Argo CD.
- Gestionarlos mediante Terraform.

No conviene mantener varias herramientas intentando administrar de forma independiente el mismo recurso sin una estrategia definida.

Se eligió usar temporalmente `CreateNamespace=true`.

### 8.2 Sincronización y autocorrección

En desarrollo, la aplicación está configurada para sincronizar automáticamente los cambios, con `prune` y `selfHeal` habilitados.

En producción, la sincronización es manual según la configuración actual. Antes de sincronizar cambios, revisar los manifiestos y validar que la imagen y la configuración sean las esperadas.

Para consultar el estado detallado:

```powershell
kubectl describe application aulaviva-backend-dev -n argocd
kubectl describe application aulaviva-backend-prod -n argocd
```

Si el comando `kubectl describe application` no reconoce el recurso, comprobar que el CRD de Argo CD esté instalado y que se esté utilizando el contexto correcto.

## 9. Verificar los despliegues

Revisar el estado general:

```powershell
kubectl get nodes
kubectl get pods -A
kubectl get applications -n argocd
```

Revisar los deployments:

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

Los pods deben estar `Running` y listos, los deployments deben tener sus réplicas disponibles y las aplicaciones de Argo CD deben mostrar el estado de salud esperado.

El HPA requiere métricas disponibles para calcular el uso de CPU y memoria. Si `kubectl top` falla, revisar la instalación de Metrics Server y los eventos del HPA.

## 10. Rollback mediante reversión de commit

El rollback permite recuperar una configuración anterior del sistema revirtiendo el commit que introdujo un cambio. En este proyecto, Git es la fuente de verdad y Argo CD sincroniza los recursos de Kubernetes con la configuración declarada en el repositorio.

### 10.1 Identificar el commit

Desde la raíz del repositorio GitOps, consultar el historial:

```bash
git log --oneline
```

Identificar el hash del commit que introdujo el cambio que se desea revertir.

### 10.2 Revertir el cambio

Ejecutar:

```bash
git revert <HASH_COMMIT>
```

Reemplazar `<HASH_COMMIT>` por el hash real del commit que se desea revertir. Este comando crea un nuevo commit que deshace los cambios seleccionados sin reescribir el historial.

### 10.3 Publicar la reversión

Subir el nuevo commit al repositorio remoto:

```bash
git push origin main
```

### 10.4 Verificar la recuperación

Consultar el estado de las aplicaciones Argo CD:

```bash
kubectl get applications -n argocd
```

Comprobar los recursos del entorno afectado:

```bash
kubectl get deployments -n equipo-aulaviva-dev
kubectl get deployments -n equipo-aulaviva-prod
```

Verificar también que la configuración del recurso afectado corresponda al estado anterior esperado.

### 10.5 Consideraciones por entorno

- **Desarrollo (`dev`):** Argo CD debe detectar la reversión y sincronizarla automáticamente, de acuerdo con la política configurada.
- **Producción (`prod`):** la reversión debe revisarse y sincronizarse manualmente, de acuerdo con la política actual de

## 11. Acceder a las aplicaciones mediante Ingress

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
```

Después, probar desde el navegador:

- `http://dev.aulaviva.local/`
- `http://aulaviva.local/`

Estas direcciones dependen de que el controlador Ingress esté listo, los puertos estén mapeados y los recursos Ingress apunten a los servicios correctos. Si no funcionan, comprobar primero el estado del controlador y la configuración de red del clúster.

## 12. Imagen de contenedor y GHCR

El backend se despliega utilizando una imagen publicada en GitHub Container Registry (GHCR).

Se recomienda utilizar una etiqueta de versión inmutable o, preferentemente, fijar el digest de la imagen en los manifiestos para que el despliegue sea reproducible.

Ejemplo de referencia:

```text
ghcr.io/merinobranco10/equipo-aulaviva-backend@sha256:<DIGEST>
```

Reemplazar `<DIGEST>` por el digest real de la imagen publicada. No se debe utilizar literalmente ese marcador.

Si el paquete GHCR es privado, Kubernetes necesita credenciales válidas mediante un `imagePullSecret` configurado en el namespace correspondiente. No almacenar tokens ni contraseñas en texto plano en el repositorio GitOps.

## 13. Solución de problemas

### Los pods no inician

```powershell
kubectl get pods -A
kubectl describe pod <NOMBRE_POD> -n <NAMESPACE>
kubectl logs <NOMBRE_POD> -n <NAMESPACE>
```

Revisar eventos, imagen configurada, permisos de descarga y configuración de la aplicación.

### Argo CD muestra `Progressing` o `Degraded`

```powershell
kubectl get applications -n argocd
kubectl describe application aulaviva-backend-dev -n argocd
kubectl describe application aulaviva-backend-prod -n argocd
```

Inspeccionar el recurso que no alcanza el estado saludable. Que una aplicación esté `Synced` no garantiza que todos sus recursos estén saludables.

### El controlador Ingress está en `Pending`

```powershell
kubectl get pods -n ingress-nginx
kubectl describe pod -n ingress-nginx <NOMBRE_POD>
kubectl get nodes --show-labels
```

Comprobar que el nodo de `kind` tenga la etiqueta `ingress-ready=true`, requerida por la configuración del controlador instalada en esta guía.

### No hay métricas para el HPA

```powershell
kubectl get apiservice v1beta1.metrics.k8s.io
kubectl top nodes
kubectl top pods -A
kubectl describe hpa -n equipo-aulaviva-dev
```

Si Metrics Server presenta errores TLS al consultar kubelet en el clúster local, investigar los certificados y la configuración de la instalación. No utilizar la desactivación de la verificación TLS como configuración de producción.

### El navegador no llega al backend

```powershell
kubectl get ingress -A
kubectl get svc -A
kubectl get pods -n equipo-aulaviva-dev
kubectl get pods -n equipo-aulaviva-prod
```

Comprobar el host, el puerto, el servicio de destino, los mapeos de puertos de `kind` y las entradas del archivo `hosts`.

## 14. Buenas prácticas de seguridad

- No versionar secretos, tokens ni contraseñas.
- Usar imágenes versionadas y fijadas por digest.
- Aplicar permisos mínimos a los componentes y usuarios.
- Mantener separados los namespaces de desarrollo y producción.
- Revisar los cambios antes de sincronizarlos en producción.
- No utilizar configuraciones TLS inseguras fuera del entorno local de pruebas.
- Mantener las versiones de las herramientas documentadas y actualizadas.
- Proteger el acceso a Argo CD y a los paquetes privados de GHCR.

## 15. Limitaciones del entorno local

El entorno `kind` sirve para desarrollar, probar y demostrar el flujo GitOps. No representa por sí solo una infraestructura de producción de alta disponibilidad.

En un entorno real deben evaluarse la disponibilidad del clúster, el almacenamiento persistente, la gestión de secretos, los certificados TLS, el monitoreo, las copias de seguridad, la seguridad de red y los recursos necesarios para soportar la carga.

## 16. Referencias oficiales

- Docker Desktop: [https://docs.docker.com/desktop/](https://docs.docker.com/desktop/)
- kind: [https://kind.sigs.k8s.io/](https://kind.sigs.k8s.io/)
- Kubernetes: [https://kubernetes.io/docs/](https://kubernetes.io/docs/)
- Kustomize: [https://kubectl.docs.kubernetes.io/references/kustomize/](https://kubectl.docs.kubernetes.io/references/kustomize/)
- Argo CD: [https://argo-cd.readthedocs.io/](https://argo-cd.readthedocs.io/)
- Ingress NGINX: [https://kubernetes.github.io/ingress-nginx/](https://kubernetes.github.io/ingress-nginx/)
- GitHub Container Registry: [https://docs.github.com/en/packages/working-with-a-github-packages-registry/working-with-the-container-registry](https://docs.github.com/en/packages/working-with-a-github-packages-registry/working-with-the-container-registry)
