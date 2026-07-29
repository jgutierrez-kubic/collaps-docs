# Variables de entorno

Configuración runtime del motor Python (`bttf-engine` / `collaps-C`) y notas sobre secretos.

## Variables del motor Python

| Variable | Obligatoria | Default | Rol |
|---|---|---|---|
| `DATABASE_URL` | **Sí** (para `/job`) | — | Connection string PostgreSQL |
| `GCS_BUCKET_NAME` | No | `bim-saas-storage-collaps-prod` | Bucket del endpoint `/upload` |
| `PORT` | No | `8080` | Puerto Uvicorn |

### Eliminado del motor (desacoplamiento CMS)

| Variable / mecanismo | Estado |
|---|---|
| `DIRECTUS_URL`, tokens Directus en env | **No aplica** |
| Lectura de `portal_projects` para auto-registro | **Eliminada** del código del engine |
| Credenciales NocoDB en el motor | **No aplica** |

La actualización de visores es responsabilidad de n8n → [Sync de visores](../orquestador/sync-visores.md).  
NocoDB tiene su propio servicio y env vars → [NocoDB](../infraestructura/nocodb.md).

### `DATABASE_URL`

```text
postgresql://USER:PASSWORD@HOST:5432/DATABASE
```

`postgres://` se normaliza a `postgresql://`.

```env
DATABASE_URL=postgresql://DB_USER:DB_PASSWORD@DB_HOST:5432/DB_NAME
GCS_BUCKET_NAME=bim-saas-storage-collaps-prod
PORT=8080
```

| Práctica | Detalle |
|---|---|
| No versionar `.env` | Entrada explícita en `.gitignore` |
| Logs seguros | Solo `host:port/database` |
| Rotación | Redeploy Cloud Run tras cambiar `DATABASE_URL` (`@lru_cache`) |

---

## Variables típicas de n8n (`n8n-collaps`)

| Variable | Rol |
|---|---|
| `WEBHOOK_URL` | Base pública para Wait / `resumeUrl` / callbacks |
| `N8N_HOST` / `N8N_PROTOCOL` / `N8N_PORT` | Hosting del editor |
| Vars DB n8n | Persistencia de workflows (Cloud SQL) |

Tokens de **NocoDB** (`xc-token`) y **Directus** (Bearer) viven en credenciales del sub-workflow de sync, no en el motor Python.

---

## Checklist pre-deploy (motor)

1. `DATABASE_URL` correcto y con permisos SELECT/INSERT/ALTER en tablas usadas.  
2. **No** se requieren env vars de Directus/NocoDB en `bttf-engine`.  
3. El flujo n8n encadena Sync Visores tras el callback si hace falta refrescar UIs.  
4. Ningún secreto real en el repositorio.
