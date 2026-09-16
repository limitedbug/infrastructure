# infrastructure

GitOps para el clúster de Kubernetes del VPS. ArgoCD lee este repositorio y
reconcilia el clúster contra lo que hay aquí: si algo no está en este repo, no
debería estar en el clúster.

## Estructura

```
clusters/production/root.yaml        App raíz: sincroniza argocd/applications/
argocd/projects/production.yaml      AppProject: qué repos, namespaces y kinds se permiten
argocd/applications/                 Una Application de ArgoCD por app
infrastructure/ingress/base/         GatewayClass + Gateway compartidos (Envoy Gateway)
apps/<app>/base/                     Manifiestos de la app
apps/<app>/overlays/production/      Overlay: fija la imagen y las etiquetas de entorno
```

Es el patrón *app-of-apps*: `root.yaml` sincroniza `argocd/applications/`, y cada
Application de ahí sincroniza su propio overlay. Para añadir una app nueva basta
con crear `apps/<app>/` y su Application; ArgoCD la recoge sola.

## Apps

| App | Host | Namespace | Imagen |
|---|---|---|---|
| `portfolio` | `good-robert.com` | `portfolio` | `ghcr.io/limitedbug/portfolio` |
| `arcadia` | `arcadia.good-robert.com` | `arcadia` | `ghcr.io/limitedbug/classic-fun-together-web` |
| `simple-coffee` | `simple-coffee.com` | `simple-coffee` | (gestionada fuera de este repo) |

Los manifiestos de `portfolio` y `arcadia` viven **aquí**, no en los repos de las
apps. Esos repos construyen y publican la imagen; este repo decide qué se
ejecuta. Las copias que quedan en `limitedbug/portfolio` (`deploy/k8s/`) y en
`limitedbug/classic-fun-together-web` (`k8s/`) ya no son lo que lee el clúster
—ver "Pendientes" más abajo.

## Antes del primer despliegue

Hay tres cosas que este repo no puede hacer por ti.

### 1. El Secret de ARCADIA

La app no arranca sin `MONGODB_URI` y `NEXTAUTH_SECRET`. No están en Git a
propósito: el AppProject ni siquiera permite el kind `Secret`, así que ArgoCD no
puede crearlo ni sobrescribirlo. Créalo una vez contra el clúster:

```sh
kubectl create namespace arcadia --dry-run=client -o yaml | kubectl apply -f -
kubectl create secret generic arcadia-secrets \
  --namespace arcadia \
  --from-literal=MONGODB_URI='mongodb+srv://usuario:password@host/arcadia' \
  --from-literal=NEXTAUTH_SECRET="$(openssl rand -base64 32)"
```

Añade `--from-literal=GOOGLE_CLIENT_ID=...` y equivalentes solo si usas esos
proveedores OAuth. La plantilla completa está en
`apps/arcadia/base/secret.example.yaml`.

Mientras el Secret no exista, el pod se queda en `CreateContainerConfigError`.

### 2. Las imágenes tienen que existir en el registro

Los overlays fijan un tag concreto porque un tag flotante hace imposible saber
qué se está ejecutando y imposible hacer rollback. Ajusta el tag al que hayas
publicado de verdad:

```sh
# apps/<app>/overlays/production/kustomization.yaml -> images[].newTag
```

- `portfolio` publica su imagen sola: el workflow `release.yml` se dispara con
  un tag `v*` y sube `ghcr.io/limitedbug/portfolio`.
- `classic-fun-together-web` **no tiene workflow de release todavía**. Su CI
  hace typecheck, lint, test y build, pero no construye ni publica la imagen.
  Hasta que lo tenga, hay que subirla a mano:

  ```sh
  docker build -t ghcr.io/limitedbug/classic-fun-together-web:v0.1.0 .
  docker push ghcr.io/limitedbug/classic-fun-together-web:v0.1.0
  ```

Los paquetes de GHCR son **privados** por defecto. O bien los haces públicos
desde la página del paquete en GitHub, o le das credenciales al clúster:

```sh
kubectl create secret docker-registry ghcr \
  --namespace arcadia \
  --docker-server=ghcr.io \
  --docker-username=limitedbug \
  --docker-password='<token con read:packages>'
```

y añades `imagePullSecrets: [{name: ghcr}]` al pod spec (en cada namespace que
lo necesite). Si no, verás `ImagePullBackOff`.

### 3. DNS y TLS

TLS lo termina lo que tengas delante del clúster (Cloudflare o similar), así que
el Gateway sigue escuchando en HTTP:80 y no hay cert-manager de por medio.
Apunta `good-robert.com` y `arcadia.good-robert.com` a la IP del VPS y deja que
el proxy hable HTTP contra el Gateway.

`NEXTAUTH_URL` sí está puesto como `https://arcadia.good-robert.com`
(`apps/arcadia/base/configmap.yaml`): es la URL que ve el navegador y la que
NextAuth usa para construir los callbacks de OAuth. Ponerla en `http://` rompe
el round-trip del `state` de OAuth.

Si algún día terminas TLS en el clúster, hay que añadir un listener HTTPS al
Gateway en `infrastructure/ingress/base/gateway.yaml`, no tocar las apps.

## Verificar

```sh
# Renderiza lo que ArgoCD va a aplicar, sin aplicarlo
kustomize build apps/portfolio/overlays/production
kustomize build apps/arcadia/overlays/production

# Estado real
kubectl -n portfolio get pods,svc,httproute
kubectl -n arcadia   get pods,svc,httproute
kubectl -n arcadia   describe httproute arcadia   # "Accepted" y "ResolvedRefs"
```

Si una ruta devuelve 503 sin que ningún pod reciba la petición, sospecha de la
NetworkPolicy antes que del Gateway: ver la sección siguiente.

## Decisiones que conviene conocer

**Las NetworkPolicies asumen que los proxies de Envoy viven en
`envoy-gateway-system`.** Es el namespace de una instalación por defecto de
Envoy Gateway, pero si el tuyo es otro, el tráfico se corta y el síntoma es un
503 limpio. Compruébalo:

```sh
kubectl get pods -A -l app.kubernetes.io/name=envoy -o wide
```

y ajusta `kubernetes.io/metadata.name` en `apps/*/base/networkpolicy.yaml`. Cada
namespace tiene un `default-deny` de ingress y egress; todo lo que se permite
está escrito explícitamente.

**`arcadia` está fijado a 1 réplica y usa `strategy: Recreate`.** No es un
placeholder. El estado de las partidas vive en memoria de un único proceso Node
y `@socket.io/mongo-adapter` está instalado pero no conectado en el código, así
que dos réplicas se pisarían. Subirlo rompe partidas en curso. El razonamiento
largo está en `apps/arcadia/base/deployment.yaml`.

**La HTTPRoute de `arcadia` desactiva el timeout de request (`0s`).** El timeout
por defecto de Envoy son 15s, y el long-polling de Socket.IO mantiene una
petición abierta ~25s antes de actualizar a WebSocket. Con el valor por defecto
la conexión se cortaría cada pocos segundos.

**No hay HorizontalPodAutoscaler.** En un VPS de un solo nodo añadir pods no
añade capacidad, solo reparte la misma CPU, y obliga a mantener metrics-server.
`portfolio` corre con 2 réplicas fijas, que es lo mínimo para sobrevivir a un
drain y a un rolling update sin caída.

**Los overlays no usan `nameSuffix`.** Kustomize sabe reescribir el backend de
un Ingress, pero `HTTPRoute` es un CRD y no reescribe `backendRefs[].name`: un
sufijo renombraría el Service y dejaría la ruta apuntando a un backend que no
existe, sin error visible al renderizar.

**El AppProject es una lista blanca.** `argocd/projects/production.yaml` enumera
los kinds que ArgoCD puede aplicar. Si añades un kind nuevo (un `Ingress`, un
`CronJob`, un `StatefulSet`) la sync falla hasta que lo añadas ahí también. Es
deliberado: que aparezca un tipo de recurso nuevo en el clúster tiene que ser
una edición consciente de ese fichero.

## Pendientes

Cosas reales que quedan fuera de este repo:

- **`limitedbug/portfolio`**: el job `pin` de `release.yml` sigue reescribiendo
  `deploy/k8s/overlays/production/kustomization.yaml` dentro de ese repo, que ya
  no es lo que lee el clúster. O se le apunta a este repositorio, o se le quita
  el job y se fija el tag aquí a mano. Además, el placeholder de su
  `deployment.yaml` apunta a `ghcr.io/limitedbug/roberto-bueno-data-code`, que no
  es la imagen que su propio workflow publica (`ghcr.io/limitedbug/portfolio`).
- **`limitedbug/classic-fun-together-web`**: no publica imagen. Necesita un
  workflow de release equivalente al de `portfolio`.
- Los directorios `deploy/k8s/` y `k8s/` de los repos de las apps quedan como
  documentación duplicada. Conviene borrarlos o dejar un README que apunte aquí,
  para que nadie aplique por error una versión obsoleta.
- `simple-coffee` sigue siendo solo una HTTPRoute que apunta a un Service
  (`simple-coffee-edge`) que no está en este repo.
