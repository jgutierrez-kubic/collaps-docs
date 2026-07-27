# Cloud Run — Motor backend (`bttf-engine`)

Despliegue del servicio Python **Condenser CORE** (`collaps-C`) a Google Cloud Run mediante build desde fuente.

## Servicio

| Atributo | Valor |
|---|---|
| Nombre del servicio | `bttf-engine` |
| Proyecto GCP | `collaps-prod` |
| Región | `us-central1` |
| Runtime | Contenedor generado desde `Dockerfile` (Python 3.10-slim + Uvicorn) |
| Entrada HTTP | `uvicorn main:app --host 0.0.0.0 --port ${PORT}` |
| Puerto | `8080` (variable `PORT`) |

La URL pública del servicio es la que consume `CollapsBttfTrigger` al llamar:

```text
POST https://bttf-engine-....us-central1.run.app/api/v1/condenser/job
```

## Comando de despliegue

Desde el directorio raíz de `collaps-C`:

```powershell
gcloud run deploy bttf-engine `
  --source . `
  --project collaps-prod `
  --region us-central1 `
  --allow-unauthenticated
```

Equivalente en una sola línea:

```bash
gcloud run deploy bttf-engine --source . --project collaps-prod --region us-central1 --allow-unauthenticated
```

### Qué hace `--source .`

1. Empaqueta el directorio actual.
2. Construye la imagen usando el `Dockerfile` del repo (Cloud Build managed).
3. Publica una nueva revisión en el servicio `bttf-engine`.
4. Enruta tráfico a la revisión nueva.

### Flags relevantes

| Flag | Efecto |
|---|---|
| `--source .` | Build + deploy desde el código local (no requiere push previo a Artifact Registry) |
| `--project collaps-prod` | Proyecto GCP destino |
| `--region us-central1` | Región del servicio |
| `--allow-unauthenticated` | Permite invocación HTTP sin identidad IAM (adecuado si n8n llama por URL pública) |

### Configurar `DATABASE_URL` en el servicio

El deploy anterior **no** inyecta secretos por sí solo. Tras (o junto con) el deploy, configura la variable:

```powershell
gcloud run services update bttf-engine `
  --project collaps-prod `
  --region us-central1 `
  --set-env-vars "DATABASE_URL=postgresql://DB_USER:DB_PASSWORD@DB_HOST:5432/DB_NAME"
```

O vía Secret Manager (recomendado en producción):

```powershell
gcloud run services update bttf-engine `
  --project collaps-prod `
  --region us-central1 `
  --update-secrets=DATABASE_URL=DATABASE_URL:latest
```

Detalle de variables: [Variables de entorno](variables-entorno.md).

## Imagen Docker del motor

Resumen del `Dockerfile` de `collaps-C`:

| Paso | Descripción |
|---|---|
| Base | `python:3.10-slim` |
| Deps | `pip install -r requirements.txt` |
| Usuario | no-root (`app`) |
| CMD | `uvicorn main:app --host 0.0.0.0 --port ${PORT} --workers 1` |

`--workers 1` es coherente con `BackgroundTasks` en el mismo proceso: más workers multiplicarían procesos sin cola compartida de jobs.

## Verificación post-deploy

```powershell
# Health / OpenAPI
curl https://bttf-engine-XXXX.us-central1.run.app/docs

# Smoke del endpoint (esperar 202 o 422 de validación)
curl -X POST https://bttf-engine-XXXX.us-central1.run.app/api/v1/condenser/job `
  -H "Content-Type: application/json" `
  -d "{}"
```

Un body `{}` debe devolver **422** (campos requeridos faltantes). Eso confirma que el servicio está vivo y validando con Pydantic.

## Relación con n8n

| Pieza | Despliegue |
|---|---|
| Motor Python | Este documento (`bttf-engine`, `--source .`) |
| Nodos + runtime n8n | [Docker / Cloud Build](docker-cloud-build.md) (Kaniko → Artifact Registry → `n8n-collaps`) |

## Notas operativas

1. `--allow-unauthenticated` expone el API públicamente: limita superficie (solo rutas necesarias) y considera auth en evoluciones futuras.
2. Un redeploy reinicia el proceso: jobs en `BackgroundTasks` en curso pueden perderse.
3. Tras rotar `DATABASE_URL`, despliega/actualiza el servicio; el `@lru_cache` del engine no se invalida en caliente.
