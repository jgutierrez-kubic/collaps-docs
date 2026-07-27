# Variables de entorno

Configuración runtime del motor Python (**Condenser CORE** / `collaps-C`) y notas sobre secretos.

## Variables del motor Python (`collaps-C`)

| Variable | Obligatoria | Default | Dónde se lee | Rol |
|---|---|---|---|---|
| `DATABASE_URL` | **Sí** (para `/job`) | — | `app/core/config.py` → `DB_URL` | Connection string PostgreSQL del engine SQLAlchemy |
| `GCS_BUCKET_NAME` | No | `bim-saas-storage-collaps-prod` | `app/core/config.py` | Bucket GCS del endpoint auxiliar `/upload` |
| `PORT` | No | `8080` | Dockerfile / Cloud Run | Puerto de Uvicorn |

### `DATABASE_URL`

Sin esta variable, `get_db_engine()` lanza:

```text
RuntimeError: DATABASE_URL no está configurada.
Configure la variable de entorno antes de conectarse a PostgreSQL.
```

**Formato esperado**

```text
postgresql://USER:PASSWORD@HOST:5432/DATABASE
```

También se acepta el esquema legacy `postgres://...`; `normalize_database_url()` lo convierte a `postgresql://` para compatibilidad con SQLAlchemy/psycopg2.

**Ejemplo con placeholders (nunca uses secretos reales en docs ni Git)**

```env
DATABASE_URL=postgresql://DB_USER:DB_PASSWORD@DB_HOST:5432/DB_NAME
GCS_BUCKET_NAME=bim-saas-storage-collaps-prod
PORT=8080
```

**Buenas prácticas**

| Práctica | Detalle |
|---|---|
| No versionar `.env` | Debe estar en `.gitignore` (entrada explícita `.env`) |
| Logs seguros | `get_database_target()` imprime solo `host:port/database`, nunca usuario/contraseña |
| Rotación | `get_db_engine()` usa `@lru_cache`: tras cambiar `DATABASE_URL` hay que **reiniciar** la revisión de Cloud Run |
| Cloud Run | Preferir Secret Manager / variables del servicio frente a archivos `.env` en la imagen |

### Credenciales Directus (no son env vars)

Directus **no** se configura vía `.env` del motor. En runtime, `AnalysisEngine` lee:

```sql
SELECT directus_url, "Instance_Token"
FROM public.portal_projects
WHERE "Schema_Name" = :schema_name
```

Si no hay fila o campos vacíos, el registro de colección se omite sin fallar el job.

---

## Variables típicas del servicio n8n (`n8n-collaps`)

Documentadas aquí a nivel de despliegue (inyectadas en `gcloud run deploy`). Los valores reales son secretos de operación.

| Variable | Rol típico |
|---|---|
| `WEBHOOK_URL` | URL pública base de webhooks n8n (necesaria para Wait / resume) |
| `N8N_HOST` / `N8N_PROTOCOL` / `N8N_PORT` | Hosting HTTP del editor / webhooks (según imagen oficial) |
| `DB_*` / connection string PG de n8n | Persistencia interna de workflows (vía Cloud SQL) |
| Credenciales Cloud SQL | Acopladas con `--add-cloudsql-instances` |

Ver el pipeline completo en [Docker / Cloud Build](docker-cloud-build.md).

---

## Checklist pre-deploy (motor)

1. `DATABASE_URL` apunta al Postgres correcto (host privado/Cloud SQL o IP autorizada).
2. El usuario DB tiene permisos de `SELECT` en tablas origen y `INSERT`/`ALTER` en destino.
3. Existe fila en `public.portal_projects` si se desea auto-registro Directus.
4. Ningún secreto real está en el repositorio ni en esta documentación.
