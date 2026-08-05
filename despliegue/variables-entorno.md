# Variables de entorno

Configuración runtime del motor Python (`bttf-engine` / `collaps-C`) tras el hotfix anti-OOM / pool SQL (`v0.1.0-hotfix`).

## Variables del motor Python

| Variable | Obligatoria | Default / valor prod | Rol |
|---|---|---|---|
| `DATABASE_URL` | **Sí** (para `/job`) | — | Connection string PostgreSQL |
| `SQL_CHUNK_SIZE` | No | **`10000`** (recomendado) | Filas por página `LIMIT/OFFSET` en el análisis |
| `DB_POOL_CPU_COUNT` | No | `os.cpu_count()`; **prod: `2`** | Factor CPU en el tamaño del pool |
| `DB_POOL_DISK_COUNT` | No | `1`; **prod: `1`** | Factor disco en el tamaño del pool |
| `GCS_BUCKET_NAME` | No | `bim-saas-storage-collaps-prod` | Bucket del endpoint `/upload` |
| `PORT` | No | `8080` | Puerto Uvicorn |

### Pool SQL (fail-fast)

Fórmula en `app/core/db.py`:

$$
\text{pool\_size} = (\text{DB\_POOL\_CPU\_COUNT} \times 2) + \text{DB\_POOL\_DISK\_COUNT}
$$

Con la configuración actual de Cloud Run (2 vCPU, 1 disco):

$$
\text{pool\_size} = (2 \times 2) + 1 = 5
$$

| Parámetro SQLAlchemy | Valor | Efecto |
|---|---|---|
| `pool_size` | fórmula arriba | Conexiones máximas en el pool |
| `max_overflow` | **`0`** | Sin conexiones extra por encima del pool |
| `pool_timeout` | **`5`** s | Fail-fast si no hay conexión libre |
| `pool_pre_ping` | `true` | Descarta conexiones muertas |
| `pool_recycle` | `300` s | Recicla conexiones antiguas |

> Alinea `DB_POOL_CPU_COUNT` con los **vCPUs** del servicio Cloud Run. Si subes CPU, actualiza esta variable y redespliega.

### Ejemplo `.env` / Cloud Run (placeholders)

```env
DATABASE_URL=postgresql://DB_USER:DB_PASSWORD@DB_HOST:5432/DB_NAME
SQL_CHUNK_SIZE=10000
DB_POOL_CPU_COUNT=2
DB_POOL_DISK_COUNT=1
GCS_BUCKET_NAME=bim-saas-storage-collaps-prod
PORT=8080
```

### Eliminado del motor (desacoplamiento CMS)

| Variable / mecanismo | Estado |
|---|---|
| `DIRECTUS_URL`, tokens Directus en env | **No aplica** |
| Lectura de `portal_projects` para auto-registro | **Eliminada** |
| Credenciales NocoDB en el motor | **No aplica** |

Visores → [Sync de visores](../orquestador/sync-visores.md).  
NocoDB → [NocoDB](../infraestructura/nocodb.md).

### `DATABASE_URL`

```text
postgresql://USER:PASSWORD@HOST:5432/DATABASE
```

| Práctica | Detalle |
|---|---|
| Host correcto | IP **privada/VPC** preferible; pública solo con firewall autorizado |
| No versionar `.env` | Entrada explícita en `.gitignore` |
| Logs seguros | Solo `host:port/database` |
| Rotación | Redeploy Cloud Run tras cambiar `DATABASE_URL` (`@lru_cache` del engine) |

Ver checklist de red en [Cloud Run](cloud-run.md#red-y-database_url-obligatorio).

---

## Variables típicas de n8n (`n8n-collaps`)

| Variable | Rol |
|---|---|
| `WEBHOOK_URL` | Base pública para Wait / `resumeUrl` / callbacks |
| `N8N_HOST` / `N8N_PROTOCOL` / `N8N_PORT` | Hosting del editor |
| Vars DB n8n | Persistencia de workflows (Cloud SQL) |

---

## Checklist pre-deploy (motor)

1. `DATABASE_URL` correcto y con permisos SELECT/INSERT/ALTER.  
2. `SQL_CHUNK_SIZE=10000`, `DB_POOL_CPU_COUNT=2`, `DB_POOL_DISK_COUNT=1` alineados con Cloud Run.  
3. Recursos Cloud Run: 2 vCPU / 4 GiB / concurrency 2 (ver [Cloud Run](cloud-run.md)).  
4. Ningún secreto real en el repositorio.
