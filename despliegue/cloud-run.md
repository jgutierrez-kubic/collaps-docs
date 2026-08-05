# Cloud Run — Motor backend (`bttf-engine`)

Despliegue del servicio Python **Condenser CORE** (`collaps-C`) a Google Cloud Run.  
Perfil **Release Stable**: anti-OOM, pool fail-fast y CPU sin throttling para `BackgroundTasks`.

## Servicio

| Atributo | Valor |
|---|---|
| Nombre del servicio | `bttf-engine` |
| Proyecto GCP | `collaps-prod` |
| Región | `us-central1` |
| Runtime | Contenedor desde `Dockerfile` (Python 3.10-slim + Uvicorn) |
| Entrada HTTP | `uvicorn main:app --host 0.0.0.0 --port ${PORT}` |
| Puerto | `8080` (`PORT`) |

La URL pública la consume `CollapsBttfTrigger`:

```text
POST https://bttf-engine-....us-central1.run.app/api/v1/condenser/job
```

---

## Capacidad y escalado (Release Stable)

| Recurso | Valor | Motivo |
|---|---|---|
| CPU | **2 vCPU** | Alineado con `DB_POOL_CPU_COUNT=2` |
| Memoria | **4 GiB** | Margen para chunks Pandas + Polars |
| Concurrencia por instancia | **2** | Evita colapso vertical / agotar el pool SQL |
| Máx. instancias | **15** | Escala **horizontal** ante picos |
| Workers Uvicorn | **1** | Un proceso; jobs en `BackgroundTasks` |
| **CPU throttling** | **Desactivado** (`--no-cpu-throttling`) | **Obligatorio** |

### Por qué `--no-cpu-throttling` es obligatorio

El endpoint responde `202` de inmediato y deja el análisis en `BackgroundTasks`. Con el throttling por defecto, GCP **reduce la CPU** cuando no hay request HTTP activa → el motor asíncrono se asfixia (jobs lentísimos o “colgados”).

> Todo deploy de `bttf-engine` debe incluir `--no-cpu-throttling`. Sin este flag el servicio no se considera estable en producción.

Ejemplo de update de capacidad:

```bash
gcloud run services update bttf-engine \
  --project collaps-prod \
  --region us-central1 \
  --cpu=2 \
  --memory=4Gi \
  --concurrency=2 \
  --max-instances=15 \
  --no-cpu-throttling
```

Variables de pool / chunk recomendadas en el mismo servicio:

```bash
gcloud run services update bttf-engine \
  --project collaps-prod \
  --region us-central1 \
  --update-env-vars "SQL_CHUNK_SIZE=10000,DB_POOL_CPU_COUNT=2,DB_POOL_DISK_COUNT=1"
```

Detalle: [Variables de entorno](variables-entorno.md).

---

## Comando de despliegue

Desde el directorio raíz de `collaps-C`:

```powershell
gcloud run deploy bttf-engine `
  --source . `
  --project collaps-prod `
  --region us-central1 `
  --allow-unauthenticated `
  --cpu=2 `
  --memory=4Gi `
  --concurrency=2 `
  --max-instances=15 `
  --no-cpu-throttling
```

### Qué hace `--source .`

1. Empaqueta el directorio actual.  
2. Construye la imagen con el `Dockerfile` (Cloud Build managed).  
3. Publica una nueva revisión en `bttf-engine`.  
4. Enruta tráfico a la revisión nueva.

### Flags relevantes

| Flag | Efecto |
|---|---|
| `--source .` | Build + deploy desde código local |
| `--project collaps-prod` | Proyecto GCP |
| `--region us-central1` | Región |
| `--allow-unauthenticated` | Invocación HTTP sin IAM (n8n por URL pública) |
| `--cpu` / `--memory` | Capacidad del contenedor |
| `--concurrency` | Peticiones simultáneas por instancia |
| `--max-instances` | Techo de escala horizontal |
| `--no-cpu-throttling` | **Obligatorio** — CPU plena durante BackgroundTasks |

### Red y `DATABASE_URL` (obligatorio)

| Requisito | Detalle |
|---|---|
| Destino | Apuntar `DATABASE_URL` a la **IP correcta** de Cloud SQL |
| Preferido | Conectividad **VPC / IP privada** (Serverless VPC Access / Direct VPC) |
| Alternativa | IP pública **solo** si el firewall / authorized networks lo permiten |
| Firewall | Asegurar que el rango/egress de Cloud Run puede alcanzar el puerto PG (5432) |

Un `DATABASE_URL` con host incorrecto o bloqueado por firewall se manifiesta como timeouts de pool (`pool_timeout=5`) o fallos de `connect_timeout`.

### Secretos / `DATABASE_URL`

```powershell
gcloud run services update bttf-engine `
  --project collaps-prod `
  --region us-central1 `
  --update-secrets=DATABASE_URL=DATABASE_URL:latest
```

---

## Imagen Docker del motor

| Paso | Descripción |
|---|---|
| Base | `python:3.10-slim` |
| Deps | `pip install -r requirements.txt` |
| Usuario | no-root (`app`) |
| CMD | `uvicorn main:app --host 0.0.0.0 --port ${PORT} --workers 1` |

`--workers 1` evita multiplicar pools SQL y jobs en background sin cola compartida.

---

## Verificación post-deploy

```powershell
curl https://bttf-engine-XXXX.us-central1.run.app/docs

curl -X POST https://bttf-engine-XXXX.us-central1.run.app/api/v1/condenser/job `
  -H "Content-Type: application/json" `
  -d "{}"
```

Un body `{}` debe devolver **422** (validación Pydantic OK).

## Relación con n8n

| Pieza | Despliegue |
|---|---|
| Motor Python | Este documento |
| Nodos + runtime n8n | [Docker / Cloud Build](docker-cloud-build.md) |

## Notas operativas

1. **Nunca** despliegues sin `--no-cpu-throttling`.  
2. Concurrencia baja + max instances alto = escala horizontal controlada.  
3. Redeploy mata jobs en `BackgroundTasks` en curso.  
4. Tras rotar `DATABASE_URL` o vars de pool, redespliega (engine en `@lru_cache`).  
5. Si cambias vCPU, actualiza `DB_POOL_CPU_COUNT` en la misma revisión.  
6. La imagen debe incluir `polars` y `pyarrow` (ver `requirements.txt`).
