# Laboratorio Kubernetes: Kind + MetalLB + Envoy Gateway

## 1. Objetivo

Implementar un entorno Kubernetes local sobre WSL2/Ubuntu usando Kind,
con acceso desde Windows y varias aplicaciones independientes publicadas
mediante un único Envoy Gateway.

El laboratorio contiene dos aplicaciones:

-   **Pagos**, en el namespace `pagos`.
-   **Compras**, en el namespace `compras`.

Cada aplicación tiene su propio Deployment, Service tipo `ClusterIP` y
`HTTPRoute`. Envoy Gateway centraliza la entrada y enruta las
solicitudes según la ruta URL.

## 2. Arquitectura implementada

``` text
Chrome en Windows
http://localhost:8080
        |
        | puerto 8080 del host
        v
Kind: extraPortMappings
hostPort 8080 -> containerPort 30080
        |
        v
NodePort 30080
        |
        v
Envoy Gateway (namespace envoy-gateway-system)
        |
        v
Gateway: gateway-infra (namespace default)
        |
        +-----------------------------+
        |                             |
        v                             v
HTTPRoute: /pagos              HTTPRoute: /compras
namespace pagos                namespace compras
        |                             |
        v                             v
pagos-service                  compras-service
ClusterIP                      ClusterIP
10.96.126.100:80               10.96.84.235:80
        |                             |
        v                             v
2 Pods NGINX                    2 Pods NGINX
```

**MetalLB** se instala en el clúster y asigna una IP externa a los
Services tipo `LoadBalancer`. En este laboratorio, el Service proxy
generado por Envoy recibió la IP `172.18.255.200`. El acceso desde
Windows se realiza por el mapeo de Kind y NodePort `30080`, publicado
como `localhost:8080`.

## 3. Archivos del proyecto

  -----------------------------------------------------------------------
  Archivo                             Función
  ----------------------------------- -----------------------------------
  `kind-config.yaml`                  Define el clúster Kind y el mapeo
                                      de puertos entre Windows y el nodo.

  `metallb-config.yaml`               Define el pool de direcciones IP y
                                      el anuncio L2 de MetalLB.

  `gateway-infra.yaml`                Crea el `GatewayClass` y el Gateway
                                      compartido `gateway-infra`.

  `pagos-app.yaml`                    Despliega Pagos: Namespace,
                                      ConfigMap, Deployment, Service y
                                      HTTPRoute.

  `compras-app.yaml`                  Despliega Compras: Namespace,
                                      ConfigMap, Deployment, Service y
                                      HTTPRoute.

  `README.md`                         Documentación del laboratorio.
  -----------------------------------------------------------------------

> Convención: `gateway-infra.yaml` se reserva para la infraestructura
> compartida de Envoy Gateway. Los manifiestos de aplicaciones usan
> nombres propios, por ejemplo `pagos-app.yaml` y `compras-app.yaml`.

## 4. Requisitos

-   Windows con WSL2 y Ubuntu.
-   Docker disponible desde Ubuntu/WSL2.
-   Kind, kubectl y Helm instalados en Ubuntu/WSL2.
-   Conectividad a Internet para descargar imágenes y el chart de Envoy
    Gateway.

Ejecutar los comandos `kind`, `helm` y `kubectl` desde Ubuntu/WSL2,
salvo que se indique PowerShell.

## 5. Instalación paso a paso

### Paso 1. Crear el clúster Kind

El archivo `kind-config.yaml` debe incluir el mapeo de puertos que
permite acceder desde Windows:

``` yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
name: mi-cluster-prueba
nodes:
  - role: control-plane
    extraPortMappings:
      - containerPort: 30080
        hostPort: 8080
        protocol: TCP
      - containerPort: 30443
        hostPort: 8443
        protocol: TCP
```

Crear el clúster:

``` bash
kind create cluster --config kind-config.yaml
kubectl get nodes -o wide
```

**¿Qué hace?** Kind crea un clúster Kubernetes local.
`extraPortMappings` publica el puerto 30080 del nodo en el puerto 8080
de Windows. Por eso el navegador puede acceder mediante
`http://localhost:8080`.

### Paso 2. Instalar Envoy Gateway

``` bash
helm install eg oci://docker.io/envoyproxy/gateway-helm \
  --version v1.10.0 \
  -n envoy-gateway-system \
  --create-namespace
```

Verificar:

``` bash
kubectl get pods -n envoy-gateway-system
```

**¿Qué hace Envoy Gateway?**

-   Instala el controlador que observa los recursos Gateway API.
-   Administra la configuración del proxy Envoy.
-   Permite definir un punto de entrada compartido para varias
    aplicaciones.
-   Procesa los recursos `GatewayClass`, `Gateway` y `HTTPRoute`.
-   Configura el enrutamiento HTTP hacia los Services de Kubernetes.

El controlador y el proxy Envoy cumplen funciones diferentes: el
controlador administra la configuración y el proxy procesa el tráfico de
las aplicaciones.

### Paso 3. Instalar MetalLB

Instalar MetalLB:

``` bash
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.15.2/config/manifests/metallb-native.yaml
```

Esperar a que sus Pods estén listos:

``` bash
kubectl wait --namespace metallb-system \
  --for=condition=Ready pod \
  --all --timeout=120s
```

Aplicar el pool de direcciones:

``` bash
kubectl apply -f metallb-config.yaml
```

Verificar:

``` bash
kubectl get pods -n metallb-system
kubectl get ipaddresspool,l2advertisement -n metallb-system
```

**¿Qué hace MetalLB?**

-   Proporciona implementación de balanceo para Services Kubernetes tipo
    `LoadBalancer` en entornos donde no existe un balanceador cloud
    nativo.
-   `IPAddressPool` define el rango de direcciones que MetalLB puede
    asignar.
-   `L2Advertisement` anuncia esas direcciones en la red de capa 2.

En este laboratorio el pool configurado es
`172.18.255.200-172.18.255.250`.

### Paso 4. Crear el Gateway compartido

Aplicar:

``` bash
kubectl apply -f gateway-infra.yaml
```

Verificar:

``` bash
kubectl get gatewayclass,gateway -A
```

Esperar a que el Gateway muestre `PROGRAMMED=True`:

``` bash
kubectl get gateway gateway-infra -n default -w
```

**¿Qué contiene `gateway-infra.yaml`?**

-   `GatewayClass`: identifica el controlador que administra los
    Gateways. En este laboratorio usa
    `gateway.envoyproxy.io/gatewayclass-controller`.
-   `Gateway`: define el punto de entrada compartido `gateway-infra`,
    con un listener HTTP en el puerto 80.
-   `allowedRoutes.namespaces.from: All`: permite que HTTPRoutes de
    otros namespaces se asocien a este Gateway.

El Gateway se crea una sola vez y puede recibir rutas de varias
aplicaciones. No se crea un Gateway por cada aplicación.

### Paso 5. Verificar y configurar el Service proxy de Envoy

Identificar el Service creado por Envoy:

``` bash
kubectl get svc -n envoy-gateway-system
```

En este laboratorio, el Service proxy de Envoy es de tipo `LoadBalancer`
y MetalLB le asignó la IP `172.18.255.200`.

El puerto NodePort debe ser `30080`, porque ese es el puerto configurado
en `kind-config.yaml`:

``` bash
kubectl get svc -n envoy-gateway-system
```

El puerto esperado se muestra como:

``` text
80:30080/TCP
```

Si el Service tiene otro NodePort, editar el Service con:

``` bash
kubectl edit svc <nombre-del-service-proxy> -n envoy-gateway-system
```

Cambiar el valor `nodePort` del puerto HTTP a `30080` y guardar.

**Importante:** el nombre del Service proxy puede contener un sufijo
generado por Envoy Gateway. Obtener siempre el nombre real con
`kubectl get svc`; no asumir que será idéntico en todos los despliegues.

### Paso 6. Desplegar la aplicación Pagos

Aplicar:

``` bash
kubectl apply -f pagos-app.yaml
```

Verificar:

``` bash
kubectl get pods,svc,httproute -n pagos
kubectl describe httproute pagos-route -n pagos
```

**¿Qué hace `pagos-app.yaml`?**

1.  `Namespace pagos`: separa los recursos de Pagos de las demás
    aplicaciones.
2.  `ConfigMap pagos-web`: contiene el HTML de la página de Pagos.
3.  `Deployment pagos-app`: ejecuta dos réplicas NGINX y monta el HTML
    en `/usr/share/nginx/html/index.html`.
4.  `Service pagos-service`: expone los Pods dentro del clúster mediante
    una IP `ClusterIP` y el puerto 80.
5.  `HTTPRoute pagos-route`: asocia la ruta `/pagos` con el Gateway
    `gateway-infra` del namespace `default`. El filtro `URLRewrite`
    reemplaza el prefijo `/pagos` por `/` antes de enviar la solicitud
    al backend.

La IP `10.96.126.100` que aparece en la página corresponde a la
ClusterIP observada durante este laboratorio. Es un dato ilustrativo
fijo del HTML; Kubernetes asigna la ClusterIP al crear el Service y
puede cambiar si se recrea.

### Paso 7. Desplegar la aplicación Compras

Aplicar:

``` bash
kubectl apply -f compras-app.yaml
```

Verificar:

``` bash
kubectl get pods,svc,httproute -n compras
kubectl describe httproute compras-route -n compras
```

**¿Qué hace `compras-app.yaml`?**

1.  `Namespace compras`: mantiene Compras aislada de Pagos.
2.  `ConfigMap compras-web`: contiene la página HTML de Compras.
3.  `Deployment compras-app`: ejecuta dos réplicas NGINX.
4.  `Service compras-service`: expone las réplicas dentro del clúster
    mediante ClusterIP y puerto 80.
5.  `HTTPRoute compras-route`: asocia `/compras` al mismo Gateway
    `gateway-infra`. El filtro `URLRewrite` cambia `/compras` por `/` al
    reenviar la petición al servicio.

La IP `10.96.84.235` que aparece en la página es la ClusterIP observada
durante este laboratorio y está escrita de forma fija en el HTML. Si se
recrea el Service, comprobar su nueva IP y actualizar el
ConfigMap/manifiesto.

## 6. Acceso desde Chrome

Las dos aplicaciones utilizan el mismo punto de entrada:

  -----------------------------------------------------------------------------------------
  Aplicación        URL                               Namespace         Service
  ----------------- --------------------------------- ----------------- -------------------
  Pagos             `http://localhost:8080/pagos`     `pagos`           `pagos-service`

  Compras           `http://localhost:8080/compras`   `compras`         `compras-service`
  -----------------------------------------------------------------------------------------

No se requiere modificar el archivo `hosts` de Windows porque las rutas
se distinguen por el path URL y no por hostname.

## 7. ¿Cómo funciona el enrutamiento en Envoy?

1.  Chrome envía una solicitud a `localhost:8080`.
2.  Kind recibe el tráfico mediante el mapeo `hostPort 8080` hacia el
    `containerPort 30080` del nodo.
3.  El NodePort `30080` entrega el tráfico al Service proxy de Envoy.
4.  Envoy Gateway recibe la solicitud HTTP.
5.  El Gateway `gateway-infra` acepta las HTTPRoutes de los namespaces
    `pagos` y `compras`.
6.  Envoy compara el path de la solicitud:
    -   `/pagos` coincide con `pagos-route`.
    -   `/compras` coincide con `compras-route`.
7.  El filtro `URLRewrite` elimina el prefijo de la aplicación y deja
    `/`.
8.  Envoy reenvía la solicitud al Service correspondiente.
9.  El Service distribuye el tráfico entre los Pods NGINX seleccionados.

De esta forma, se mantiene un único Gateway compartido y cada equipo o
aplicación puede administrar sus recursos en su propio namespace, sujeto
a las políticas y permisos definidos por la plataforma.

## 8. Comandos de diagnóstico

Ver los nodos:

``` bash
kubectl get nodes -o wide
```

Ver los Pods de todos los namespaces:

``` bash
kubectl get pods -A
```

Ver Gateways:

``` bash
kubectl get gatewayclass,gateway -A
```

Ver rutas:

``` bash
kubectl get httproute -A
```

Ver los Services de Envoy:

``` bash
kubectl get svc -n envoy-gateway-system
```

Ver los recursos de Pagos y Compras:

``` bash
kubectl get all -n pagos
kubectl get httproute -n pagos
kubectl get all -n compras
kubectl get httproute -n compras
```

Ver las condiciones de las rutas:

``` bash
kubectl describe httproute pagos-route -n pagos
kubectl describe httproute compras-route -n compras
```

Las condiciones esperadas de cada HTTPRoute son:

-   `Accepted=True`
-   `ResolvedRefs=True`

Probar desde PowerShell:

``` powershell
curl.exe http://localhost:8080/pagos
curl.exe http://localhost:8080/compras
```

## 9. Consideraciones del laboratorio

-   MetalLB y Envoy Gateway tienen responsabilidades distintas. MetalLB
    asigna y anuncia direcciones para Services `LoadBalancer`; Envoy
    Gateway administra el enrutamiento HTTP.
-   Las aplicaciones Pagos y Compras utilizan Services `ClusterIP`; no
    requieren una IP externa individual.
-   Las IPs ClusterIP son internas al clúster y no deben confundirse con
    la IP externa asignada por MetalLB.
-   El acceso desde Windows se realiza mediante el mapeo de puertos de
    Kind y el NodePort de Envoy.
-   Las IPs fijas que aparecen en las páginas son valores del
    laboratorio. Si se recrea un Service, verificar la IP asignada antes
    de actualizar el HTML.
-   Si se elimina y recrea el clúster Kind, será necesario volver a
    instalar Envoy Gateway y MetalLB y aplicar nuevamente los
    manifiestos.
-   Para un entorno productivo se deben revisar alta disponibilidad,
    seguridad, TLS, políticas de acceso, observabilidad, capacidad y
    gestión de certificados.

## 10. Orden resumido de instalación

``` bash
# 1. Crear clúster
kind create cluster --config kind-config.yaml

# 2. Instalar Envoy Gateway
helm install eg oci://docker.io/envoyproxy/gateway-helm \
  --version v1.10.0 \
  -n envoy-gateway-system \
  --create-namespace

# 3. Instalar MetalLB
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.15.2/config/manifests/metallb-native.yaml

# 4. Configurar MetalLB
kubectl apply -f metallb-config.yaml

# 5. Crear Gateway compartido
kubectl apply -f gateway-infra.yaml

# 6. Desplegar Pagos
kubectl apply -f pagos-app.yaml

# 7. Desplegar Compras
kubectl apply -f compras-app.yaml
```
