# config-store · Helm chart
 
Chart de Helm para desplegar **config-store** en Kubernetes junto con su base de datos **PostgreSQL**.
 
config-store es un servicio HTTP (Node.js / Express) que guarda sus datos en PostgreSQL y expone un endpoint de salud en `/health`. El chart despliega la aplicación y, como dependencia, un PostgreSQL con almacenamiento persistente.
 
| | |
|---|---|
| Chart | `config-store` `0.2.0` |
| Versión de la app (`appVersion`) | `1.0.0` |
| Imagen | `docker.io/ecfcode/config-store:1.0.0` |
| Dependencia | `postgresql` `16.7.27` (Bitnami) |
 
## Arquitectura
 
```
 Cliente
    │
    ▼
 Service config-store (NodePort :80)
    │
    ▼
 Deployment config-store (1 pod, UID 1000)
    │   postgresql://app_user:***@postgresql-db:5432/db_app_store
    ▼
 Service postgresql-db (ClusterIP :5432)
    │
    ▼
 StatefulSet postgresql-db (1 pod)  ──►  PVC 8Gi
    ▲
    │ lee las contraseñas como ficheros
 Secret postgres-creds  (se crea a mano, no lo gestiona el chart)
```
 
## Contenido del repositorio
 
```
.
├── Chart.yaml              # metadatos del chart y dependencia de PostgreSQL
├── Chart.lock              # versión exacta de la dependencia
├── values.yaml             # valores por defecto (app + subchart postgresql)
├── templates/
│   ├── _helpers.tpl
│   ├── deployment.yaml     # la aplicación
│   ├── service.yaml
│   ├── serviceaccount.yaml
│   ├── ingress.yaml        # desactivado por defecto
│   ├── httproute.yaml      # desactivado por defecto (Gateway API)
│   ├── hpa.yaml            # desactivado por defecto
│   ├── NOTES.txt
│   └── tests/
│       └── test-connection.yaml
└── charts/                 # lo genera `helm dependency build`; no se versiona
```
 
## Requisitos
 
- Un clúster de Kubernetes con una `StorageClass` por defecto (para el PVC de PostgreSQL).
- `kubectl` y `helm`.
- Acceso a internet para descargar el chart de PostgreSQL y las imágenes.
## Instalación
 
**1. Crea el Secret con las credenciales.** El chart no lo crea y PostgreSQL no arranca sin él. Necesita las dos claves, `password` y `postgres-password`. Se usa hexadecimal para que la contraseña no contenga caracteres especiales, porque la app la inserta en una URL de conexión.
 
```bash
kubectl create secret generic postgres-creds \
  --from-literal=password="$(openssl rand -hex 24)" \
  --from-literal=postgres-password="$(openssl rand -hex 24)"
```
 
**2. Descarga la dependencia.** La carpeta `charts/` no se versiona, así que en un clon limpio hay que generarla:
 
```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
helm dependency build
```
 
**3. Valida e instala:**
 
```bash
helm lint .
helm install config-store . -f values.yaml
kubectl rollout status statefulset/postgresql-db
kubectl rollout status deploy/config-store
```
 
PostgreSQL tarda unos 30 segundos en estar listo. Si la app arranca antes que la base de datos, puede reiniciarse un par de veces hasta que conecta.
 
**4. Comprueba que funciona:**
 
```bash
kubectl get pods                          # config-store-… y postgresql-db-0 en 1/1 Running
kubectl logs deploy/config-store          # "Connected to PostgreSQL" y "Server running on port 80"
 
kubectl port-forward svc/config-store 8080:80
curl http://localhost:8080/health         # {"status":"ok"}
```
 
Opcionalmente, el chart incluye un pod de prueba: `helm test config-store`.
 
## Configuración
 
Los valores principales de `values.yaml`:
 
| Clave | Por defecto | Descripción |
|---|---|---|
| `replicaCount` | `1` | Réplicas de la aplicación |
| `image.repository` | `docker.io/ecfcode/config-store` | Imagen de la app |
| `image.tag` | `""` | Vacío = usa `appVersion` de `Chart.yaml` |
| `image.pullPolicy` | `IfNotPresent` | |
| `service.type` / `service.port` | `NodePort` / `80` | El puerto del Service es también el de la app (`PORT`) |
| `resources` | req `100m`/`128Mi`, lim `200m`/`256Mi` | Recursos de la app |
| `ingress.enabled` / `httpRoute.enabled` / `autoscaling.enabled` | `false` | |
| `postgresql.auth.database` | `db_app_store` | Base de datos que crea PostgreSQL |
| `postgresql.auth.username` | `app_user` | Usuario de la app |
| `postgresql.auth.existingSecret` | `postgres-creds` | Secret con las credenciales |
| `postgresql.auth.secretKeys.userPasswordKey` | `password` | Clave de la contraseña del usuario |
| `postgresql.auth.usePasswordFiles` | `true` | PostgreSQL lee las contraseñas como ficheros, no como variables de entorno |
| `postgresql.fullnameOverride` | `postgresql-db` | Nombre del Service de la BD. La app lo usa como host |
| `postgresql.image.*` | `docker.io/bitnamilegacy/postgresql:17.6.0-debian-12-r4` | Imagen de PostgreSQL |
| `global.security.allowInsecureImages` | `true` | Necesario para que el chart de Bitnami acepte imágenes que no son las suyas |
 
Variables de entorno que recibe la aplicación:
 
| Variable | Origen |
|---|---|
| `PORT` | `service.port` |
| `DB_USER`, `DB_NAME` | `postgresql.auth.username`, `postgresql.auth.database` |
| `DB_PASSWORD` | Secret `postgres-creds`, clave `password` |
| `DB_URL` | Se compone con las anteriores y el host `postgresql-db` |
 
## Actualizar
 
Pasa **siempre** `-f values.yaml`. Con `helm upgrade` sin `-f` ni `--set`, Helm reutiliza los valores guardados de la revisión anterior (comportamiento observado con Helm v4.3.1) y los cambios del fichero no se aplican, sin ningún error.
 
Revisa los cambios antes de aplicarlos:
 
```bash
helm get values config-store                                       # lo que hay desplegado
helm upgrade config-store . -f values.yaml --dry-run=server        # valida el render
helm diff upgrade config-store . -f values.yaml --dry-run=server   # qué cambiaría (plugin helm-diff)
helm upgrade config-store . -f values.yaml
```
 
- Usa `--dry-run=server`. Con `--dry-run=client` o `helm template --is-upgrade`, el subchart de PostgreSQL falla con `PASSWORDS ERROR`.
- Instalación del plugin `helm-diff` con Helm 4: `helm plugin install https://github.com/databus23/helm-diff --verify=false`. El plugin no publica firma, así que debes asumir esa confianza.
- Si la imagen de la app cambia, publica una versión nueva (`1.0.1`), sube `appVersion` en `Chart.yaml` y no reutilices tags ya publicados.
Las credenciales de PostgreSQL se aplican al inicializar el volumen, según el comportamiento de la imagen de Bitnami. Cambiar `postgres-creds` después no cambia la contraseña de la base de datos: hay que ejecutar `ALTER USER` dentro de PostgreSQL y actualizar el Secret.
 
## Seguridad
 
- **Aplicación:** corre sin root (`runAsUser: 1000`), con sistema de ficheros raíz de solo lectura y todas las capabilities eliminadas.
- **PostgreSQL:** usa el contexto de seguridad endurecido del chart de Bitnami (UID 1001, sin escalada de privilegios, `seccomp: RuntimeDefault`, raíz de solo lectura).
- **Credenciales:** no se guardan en el chart ni en los valores de Helm. Viven en el Secret `postgres-creds`, y PostgreSQL las lee desde ficheros montados.
## Empaquetar el chart
 
Para generar un paquete instalable (`.tgz`) con la dependencia incluida:
 
```bash
helm dependency build
helm package .                         # genera config-store-0.2.0.tgz
helm install config-store ./config-store-0.2.0.tgz -f values.yaml
```
 
## Desinstalar
 
```bash
helm uninstall config-store
kubectl delete pvc data-postgresql-db-0     # borra los datos de la base
kubectl delete secret postgres-creds
```
 
`helm uninstall` no elimina el PVC, así que los datos se conservan hasta que lo borras.
 
## Limitaciones conocidas
 
Este chart es una base funcional, no una plataforma lista para producción:
 
- Una sola réplica de la app y de PostgreSQL, sin alta disponibilidad ni copias de seguridad automáticas.
- La imagen de PostgreSQL viene de `bitnamilegacy`, un repositorio archivado que no recibe parches de seguridad. Para producción conviene migrar a la imagen oficial, a un operador o a una base gestionada.
- Las sondas de la app consultan `/`, no comprueban la conexión con la base de datos.
- La app necesita PostgreSQL disponible al arrancar. Si no lo está, se reinicia hasta que conecta.
- La contraseña llega a la app como variable de entorno (`DB_PASSWORD`). Solo PostgreSQL usa ficheros.
- La `NetworkPolicy` del subchart permite conexiones al puerto 5432 desde cualquier origen.
- El Secret `postgres-creds` se crea a mano antes de instalar.
- El chart de PostgreSQL se descarga del repositorio de Bitnami. Si la versión `16.7.27` desapareciera de él, habría que sustituir la dependencia.
## Probado con
 
- Helm v4.3.1
- minikube v1.39.0 (Kubernetes v1.37.0, containerd 2.3.4) sobre Fedora 44
- Plugin `helm-diff` 3.15.15
No se ha probado en otros clústeres ni versiones de Helm.
