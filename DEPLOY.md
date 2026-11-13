# Quantum Fit — Guía de despliegue

Documentación operativa de los tres componentes de Quantum Fit.

```
┌─────────────────┐      ┌──────────────────┐      ┌─────────────────────────┐
│  App móvil      │      │  Landing         │      │  Panel admin            │
│  (Expo)         │─────►│  quantum-fit-    │─────►│  quantum-fit-admin      │
│                 │      │  landing.vercel  │      │  .vercel.app            │
└─────────────────┘      └──────────────────┘      └─────────────────────────┘
                                   │                              │
                                   └──────────────┬───────────────┘
                                                  │  HTTPS + Socket.IO
                                                  ▼
                                   ┌────────────────────────────┐
                                   │  Backend (Railway)          │
                                   │  backend-production-0360e674│
                                   │  .up.railway.app            │
                                   │  Express + Socket.IO        │
                                   └──────┬──────────────┬────────┘
                                          │              │
                                   ┌──────▼─────┐  ┌─────▼──────────┐
                                   │ PostgreSQL │  │ Volume uploads │
                                   │  (plugin)  │  │  /app/uploads  │
                                   └────────────┘  └────────────────┘
```

---

## URLs de producción

| Componente | URL |
| --- | --- |
| Landing | https://quantum-fit-landing.vercel.app |
| Panel admin | https://quantum-fit-admin.vercel.app |
| API backend | https://backend-production-0360e674.up.railway.app |
| Health check | https://backend-production-0360e674.up.railway.app/health |

El prefijo `/api` se agrega a la URL del backend en las variables de entorno.

---

## 1. Vercel (landing + panel)

Dos proyectos de Vercel sobre **el mismo repositorio**, cada uno con su propio
*Root Directory*.

| | Proyecto | Root Directory | Framework | Output |
| --- | --- | --- | --- | --- |
| 1 | `quantum-fit-landing` | `quantum-fit-landing` | Next.js | (default) |
| 2 | `quantum-fit-admin` | `quantum-fit-admin` | Vite | `dist` |

### Variables de entorno

| Proyecto | Variable | Valor |
| --- | --- | --- |
| `quantum-fit-landing` | `NEXT_PUBLIC_API_URL` | `https://backend-production-0360e674.up.railway.app/api` |
| `quantum-fit-admin` | `VITE_API_URL` | `https://backend-production-0360e674.up.railway.app/api` |
| `quantum-fit-admin` | `VITE_LANDING_URL` | `https://quantum-fit-landing.vercel.app` |

> ⚠️ `NEXT_PUBLIC_*` y `VITE_*` se **inlinean en el bundle durante el build**.
> Cambiar la variable no alcanza: hay que volver a desplegar.

### Conectar el repositorio con GitHub (deploy automático)

1. Instalá la **Vercel GitHub App** en la cuenta que es dueña del repo
   (`gbuczek`): https://github.com/apps/vercel → *Install* → elegí la cuenta →
   *Repository access: Only select repositories* → marcá `QuantumFit`.
2. En cada proyecto: **Settings → Git → Connect Git Repository** → `gbuczek/QuantumFit`.
3. Verificá que el *Root Directory* de cada proyecto sea el de la tabla de arriba.

A partir de ahí, cada `git push` a `main` despliega automáticamente.

### Despliegue manual (sin conectar GitHub)

```bash
# Desde la raíz del repo, apuntando al proyecto correcto
cat > .vercel/project.json <<'EOF'
{"projectId":"prj_X3rGaltPB3Tw93Qg312fX7sewb3k","orgId":"team_BDlV9SdySg8UXG6OLjViYy4R","projectName":"quantum-fit-landing"}
EOF
npx vercel --prod --yes --scope buczek-guillermo-sebastians-projects
```

> Desplegando **desde la raíz del repo** el *Root Directory* del proyecto se
> respeta. Si desplegás desde dentro de la carpeta, Vercel busca un
> subdirectorio con ese nombre dentro de lo subido y falla con
> *"The specified Root Directory does not exist"*.

---

## 2. Railway (backend + base de datos)

### Servicios del proyecto `QuantumFit`

| Servicio | Tipo | Volumen | Montaje |
| --- | --- | --- | --- |
| `backend` | Node (Dockerfile) | `uploads` | `/app/uploads` |
| `Postgres` | Plugin PostgreSQL 16 | `postgres-volume` | `/var/lib/postgresql/data` |

### Límites de recursos (techo de gasto)

| Servicio | RAM | vCPU |
| --- | --- | --- |
| `backend` | 1 GB | 1 vCPU |
| `Postgres` | 2 GB | 2 vCPU |

El consumo real se factura por segundo; estos límites son solo un techo para
que un memory leak no dispare la factura.

### Variables de entorno del backend

Se setean en el dashboard o por API. Las que importan:

| Variable | Valor / origen |
| --- | --- |
| `DATABASE_URL` | `${{Postgres.DATABASE_URL}}` (referencia al plugin) |
| `ALLOWED_ORIGINS` | `https://quantum-fit-landing.vercel.app,https://quantum-fit-admin.vercel.app` |
| `EXPO_PUBLIC_API_URL` | `https://backend-production-0360e674.up.railway.app` |
| `MERCADOPAGO_NOTIFICATION_URL` | `https://backend-production-0360e674.up.railway.app/api/mercadopago/webhook` |
| `NODE_ENV` | `production` |
| `PORT` | **no setear** — Railway lo inyecta |

> `ALLOWED_ORIGINS` es la lista de orígenes CORS permitidos. Si agregás un
> dominio nuevo (por ejemplo un custom domain), hay que actualizarlo **y**
> redeployar.

### El volumen de `uploads` es crítico

El admin sube imágenes con Multer, que escribe en `process.cwd()/uploads`.
Como el `Dockerfile` usa `WORKDIR /app`, el path real es `/app/uploads`.
El volume **debe** montarse ahí:

```bash
railway volume add --mount-path /app/uploads
```

Si se monta en otro lado, cada redeploy borra las fotos.

### Migraciones de base de datos

El `CMD` del Dockerfile corre `prisma migrate deploy` en cada arranque, así que
las migrations se aplican solas al desplegar.

> Ya pasó: la migration `20260828000000_add_crystal_membership_link` agregó
> `crystalEmail` y `crystalTokenEncrypted`, pero **no** `crystalData`, que sí
> estaba en `schema.prisma`. El login fallaba con *"column User.crystalData
> does not exist"*. Corregido con
> `prisma/migrations/20261112000000_add_crystal_data/migration.sql`.
>
> **Regla:** después de tocar `schema.prisma`, generar la migration con
> `npx prisma migrate dev --name <nombre>` y commitearla. Si se edita el schema
> sin migration, el deploy falla en runtime, no en build.

### Backups

| Volumen | Frecuencia | Retención |
| --- | --- | --- |
| `postgres-volume` | Diaria + Semanal | 6 días / 27 días |
| `uploads` | Diaria + Semanal | 6 días / 27 días |

Se configuran en **Railway → servicio → Backups**. Requieren un token de
**cuenta** (no de workspace) para gestionarlos por API.

### Despliegue manual del backend

```bash
export RAILWAY_API_TOKEN=<token>
railway up --detach --yes
```

El backend se despliega **desde GitHub** cuando el Railway GitHub App esté
instalado. Para eso hay que instalar la app en la cuenta de GitHub que tenga el
repo del backend y autorizar el workspace de Railway del cliente.

---

## 3. Cuenta de usuario inicial

El seed (`prisma/seed.ts`) se ejecuta en cada arranque y usa `upsert`, así que
es idempotente. Crea:

| Email | Contraseña | Rol |
| --- | --- | --- |
| `admin@quantumfit.com` | `Admin123!` | ADMIN (`Staff`) |
| `demo@quantumfit.com` | `Demo123!` | USER |

> 🔴 **Cambiar la contraseña del admin antes de entregar a la clienta.**

---

## 4. Verificación end-to-end

```bash
# Backend arriba
curl https://backend-production-0360e674.up.railway.app/health
# {"success":true,"message":"QUANTUM FIT API is running",...}

# API con datos reales
curl https://backend-production-0360e674.up.railway.app/api/landing/site-config

# Login del admin
curl -X POST https://backend-production-0360e674.up.railway.app/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"admin@quantumfit.com","password":"Admin123!"}'

# CORS: el origen de Vercel debe reflejarse en la respuesta
curl -D - -o /dev/null -H "Origin: https://quantum-fit-admin.vercel.app" \
  https://backend-production-0360e674.up.railway.app/api/landing/gyms
```

---

## 5. Pendientes conocidos

| # | Tarea | Prioridad |
| --- | --- | --- |
| 1 | Renovar `EXTERNAL_API_TOKEN` de Crystal (el actual devuelve 401 en toda la API) | 🔴 Alta |
| 2 | Cambiar la contraseña de `admin@quantumfit.com` | 🔴 Alta |
| 3 | `MERCADOPAGO_ACCESS_TOKEN` es un token **de prueba** (`TEST-…`): los pagos no cobran real | 🔴 Alta |
| 4 | Activar backups automáticos en los dos volumes | 🟡 Media |
| 5 | Configurar un límite de gasto en la cuenta (Billing → Usage limits) | 🟡 Media |
| 6 | Conectar el backend a GitHub para deploy automático | 🟡 Media |
| 7 | Migrar `railway.json` → `.railway/railway.ts` (Config as Code está deprecado, se corta el **2026-12-01**) | 🟡 Media |
| 8 | `syncAllUsers` recorre los usuarios en serie: con 1000 socios tarda 10-25 min en un solo request. Hay que volverlo un job en background | 🟡 Media |
| 9 | El endpoint `/health` no consulta la base: si Postgres cae, Railway sigue creyendo sano el servicio | 🟢 Baja |

---

## 6. Modelo de despliegue

| Plataforma | Qué vive ahí | Por qué |
| --- | --- | --- |
| **Vercel** | Landing (Next.js) y panel (Vite) | Son frontends estáticos/hybrid. Render en el edge, CDN global, 0 mantenimiento de servidores. Las llamadas a la API se hacen **desde el navegador**, no desde el servidor: por eso sus builds no dependen de que el backend esté online. |
| **Railway** | Backend (Express + Socket.IO) y PostgreSQL | Es un proceso Node de larga duración. Necesita **WebSockets persistentes** (notificaciones en vivo de la app móvil), **disco persistente** para las imágenes y una **base de datos relacional** en la misma red privada. Vercel no puede dar las tres cosas. |
