# Material provisto – Actividad N°4

| Carpeta | Contenido |
|---|---|
| `app/` | API de ranking de GameCloud, ya instrumentada: `GET /ranking`, `POST /scores`, `/healthz`, `/metrics` y logs en JSON |
| `charts/gamecloud-ranking/` | Chart de Helm de la API: Deployment, Service y un generador de tráfico. No incluye el ServiceMonitor |
| `escenarios/` | Scripts para provocar fallas en la API y volver a la normalidad |

## Desplegar la API

El chart y los escenarios esperan el release `gamecloud-ranking` en el namespace `gamecloud`.

```bash
# Construir la imagen dentro de minikube (el chart usa imagePullPolicy: Never)
minikube image build -t gamecloud-ranking:1.0 app/

helm upgrade --install gamecloud-ranking charts/gamecloud-ranking \
  -n gamecloud --create-namespace --wait
```

## Métricas que expone la API

| Métrica | Tipo | Labels |
|---|---|---|
| `http_requests_total` | Counter | `method`, `ruta`, `status` |
| `http_request_duration_seconds` | Histogram | `method`, `ruta` |

## Formato de los logs

```json
{"ts": "2026-10-22T03:51:28.393+00:00", "level": "info", "msg": "pedido atendido", "pod": "gamecloud-ranking-7d9f-abcde", "method": "POST", "path": "/scores", "status": 201, "duracion_ms": 0.3}
```

`level` vale `info`, `warning` (respuestas 4XX), `error` (respuestas 5XX) o `critical`.
