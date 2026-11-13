# TP4 – Observabilidad: GameCloud en Kubernetes

## Descripción

Este repositorio contiene los archivos de configuración utilizados para desarrollar el trabajo práctico de observabilidad de una aplicación desplegada en Kubernetes.

El objetivo es desplegar la API GameCloud y supervisar su funcionamiento mediante herramientas de monitoreo y observabilidad, utilizando métricas y logs para analizar el comportamiento del sistema e identificar posibles fallas.

La solución utiliza:

- **Minikube:** entorno local de Kubernetes.
- **Helm:** instalación y administración de los componentes.
- **Prometheus:** recolección y consulta de métricas.
- **Grafana:** visualización de métricas y logs mediante dashboards.
- **Loki:** almacenamiento y consulta de logs.
- **Alloy:** recolección, procesamiento y envío de logs hacia Loki.


## Requisitos previos

Para reproducir la práctica se necesita tener instalado:

- Docker Desktop.
- Minikube.
- kubectl.
- Helm.

Los comandos de esta guía están preparados para ejecutarse desde **Windows PowerShell**.

## 1. Preparar el entorno Kubernetes

Iniciar Minikube con los recursos utilizados durante la práctica:

```powershell
minikube start --cpus=4 --memory=8192 --driver=docker
```

Comprobar que el nodo se encuentre disponible:

```powershell
kubectl get nodes
```

Agregar los repositorios Helm necesarios:

```powershell
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update
```

Crear los namespaces:

```powershell
kubectl create namespace gamecloud
kubectl create namespace observabilidad
```

## 2. Desplegar la aplicación GameCloud

Desde la raíz del repositorio, construir la imagen de la API dentro del entorno Minikube:

```powershell
minikube image build -t gamecloud-ranking:1.0 app/
```

Instalar la aplicación mediante Helm:

```powershell
helm upgrade --install gamecloud-ranking .\charts\gamecloud-ranking -n gamecloud --wait
```

Comprobar el estado de los pods:

```powershell
kubectl get pods -n gamecloud
```

Para acceder a la API desde el equipo local, ejecutar:

```powershell
kubectl port-forward -n gamecloud svc/gamecloud-ranking 8080:80
```

Con el port-forward activo, se pueden consultar los siguientes endpoints:

- API de ranking: `http://localhost:8080/ranking`
- Estado de la API: `http://localhost:8080/healthz`
- Métricas: `http://localhost:8080/metrics`

## 3. Instalar Prometheus y Grafana

Instalar kube-prometheus-stack utilizando el archivo de valores propio:

```powershell
helm upgrade --install kube-prometheus-stack prometheus-community/kube-prometheus-stack --version 92.11.0 -n observabilidad -f .\observabilidad\values-kube-prometheus-stack.yaml --wait --timeout 10m
```

Acceder a Grafana:

```powershell
kubectl port-forward -n observabilidad svc/kube-prometheus-stack-grafana 3000:80
```

Abrir `http://localhost:3000` en el navegador.

El usuario inicial es `admin`. La contraseña se obtiene del Secret creado por Helm:

```powershell
kubectl get secret kube-prometheus-stack-grafana -n observabilidad -o jsonpath="{.data.admin-password}"
```

El valor obtenido está codificado en Base64 y debe decodificarse para utilizarlo como contraseña.

## 4. Configurar la recolección de métricas de GameCloud

La aplicación incluye un ServiceMonitor en:

```text
charts/gamecloud-ranking/templates/servicemonitor.yaml
```

Este recurso permite que Prometheus descubra los endpoints de métricas de la API.

Aplicar la configuración mediante una actualización del despliegue:

```powershell
helm upgrade --install gamecloud-ranking .\charts\gamecloud-ranking -n gamecloud --wait
```

Se puede comprobar la existencia del ServiceMonitor con:

```powershell
kubectl get servicemonitors -n gamecloud
```

Para verificar la recolección, acceder a Prometheus:

```powershell
kubectl port-forward -n observabilidad svc/kube-prometheus-stack-prometheus 9090:9090
```

Abrir `http://localhost:9090` y consultar, por ejemplo:

```promql
http_requests_total
```

El target correspondiente a GameCloud debe figurar como `UP`.

## 5. Instalar Loki

Instalar Loki utilizando el archivo de valores preparado para el despliegue SingleBinary con almacenamiento filesystem y sin multitenancy:

```powershell
helm upgrade --install loki grafana/loki --version 7.5.0 -n observabilidad -f .\observabilidad\values-loki.yaml --wait --timeout 10m
```

Comprobar el estado de los pods:

```powershell
kubectl get pods -n observabilidad
```

## 6. Instalar y configurar Alloy

Instalar Alloy utilizando su archivo de configuración:

```powershell
helm upgrade --install alloy grafana/alloy --version 1.14.2 -n observabilidad -f .\observabilidad\values-alloy.yaml --wait --timeout 10m
```

La configuración permite:

- Descubrir los pods de Kubernetes.
- Incorporar los labels `namespace`, `pod`, `container` y `app`.
- Procesar los logs en formato JSON.
- Extraer el nivel del log (`level`) como etiqueta.
- Enviar los registros a Loki.

En Grafana, Loki queda disponible como fuente de datos con la dirección:

```text
http://loki.observabilidad.svc.cluster.local:3100
```

Una consulta de ejemplo para visualizar los logs de la API es:

```logql
{namespace="gamecloud", app="gamecloud-ranking"}
```

Para consultar solamente los registros de error:

```logql
{namespace="gamecloud", app="gamecloud-ranking", level="error"}
```

## 7. Dashboard de observabilidad

El archivo `dashboards/dashboard-gamecloud.json` contiene el dashboard desarrollado para la práctica.

Para importarlo:

1. Ingresar a Grafana.
2. Acceder a **Dashboards** y seleccionar **New → Import**.
3. Cargar el archivo `dashboard-gamecloud.json`.
4. Seleccionar las fuentes de datos correspondientes: Prometheus y Loki.
5. Confirmar la importación.

El dashboard incluye paneles para visualizar:

- Pedidos por segundo, agrupados por ruta.
- Porcentaje de errores HTTP 5XX.
- Latencia p95 por ruta.
- Pods disponibles.
- Reinicios por pod.
- Consumo de CPU por pod y sus límites.
- Consumo de memoria por pod y sus límites.
- Logs de error de la API.

## 8. Escenarios de falla

Los escenarios se encuentran definidos en la carpeta `escenarios/` del material original del profesor. Para ejecutarlos desde PowerShell se utilizaron los siguientes comandos, desde la raíz del repositorio.

**Escenario 1 – Errores HTTP 500:**

```powershell
helm upgrade gamecloud-ranking .\charts\gamecloud-ranking -n gamecloud --reuse-values --set fallas.errorRate=0.2
```

**Escenario 2 – Incremento de latencia:**

```powershell
helm upgrade gamecloud-ranking .\charts\gamecloud-ranking -n gamecloud --reuse-values --set fallas.latenciaMs=2000
```

**Escenario 3 – Fallo al iniciar los pods:**

```powershell
helm upgrade gamecloud-ranking .\charts\gamecloud-ranking -n gamecloud --reuse-values --set fallas.fallarAlIniciar=true
```

**Escenario 4 – Puerto incorrecto en el Service:**

```powershell
kubectl -n gamecloud patch service gamecloud-ranking --type json -p '[{"op":"replace","path":"/spec/ports/0/targetPort","value":9999}]'
```

Para restaurar el estado normal después de las pruebas, ejecutar:

```powershell
helm upgrade gamecloud-ranking .\charts\gamecloud-ranking -n gamecloud --set fallas.errorRate=0 --set fallas.latenciaMs=0 --set fallas.fallarAlIniciar=false

kubectl -n gamecloud patch service gamecloud-ranking --type json -p '[{"op":"replace","path":"/spec/ports/0/targetPort","value":"http"}]'
```

## 9. Finalizar el entorno

Para detener Minikube:

```powershell
minikube stop
```

Para eliminar el entorno completo y sus recursos:

```powershell
minikube delete
```

**Nota:** `minikube delete` elimina el clúster y los recursos desplegados en él. No es necesario ejecutarlo para detener temporalmente la práctica.