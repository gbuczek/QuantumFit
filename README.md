# Quantum Fit

Repositorio con el **frontend público** de Quantum Fit (gimnasio): la landing y el panel de administración.

| Proyecto | Stack | Puerto local | Deploy |
| --- | --- | --- | --- |
| [`quantum-fit-landing/`](./quantum-fit-landing) | Next.js 16 (App Router) + Tailwind 4 | `3001` | Vercel |
| [`quantum-fit-admin/`](./quantum-fit-admin) | Vite + React 19 + React Router 7 | `5173` | Vercel |

> El **backend** (Express + Socket.IO + Prisma + PostgreSQL) y la **app móvil** (Expo) viven en
> repositorios aparte, por ser servicios de larga duración con WebSockets y base de datos.

---

## Levantar en local

```bash
npm run install:all      # instala dependencias de landing + admin
npm run dev              # levanta landing (3001) y admin (5173) en paralelo
```

O por proyecto:

```bash
cd quantum-fit-landing && npm install && npm run dev
cd quantum-fit-admin  && npm install && npm run dev
```

Ambos frontends consumen la API del backend. Configurá el destino en los archivos `.env.local`
(copiando los `.env.example`):

```bash
cp quantum-fit-landing/.env.example quantum-fit-landing/.env.local
cp quantum-fit-admin/.env.example  quantum-fit-admin/.env.local
```

---

## Variables de entorno

| Landing | Admin |
| --- | --- |
| `NEXT_PUBLIC_API_URL` | `VITE_API_URL` |
| | `VITE_LANDING_URL` |

- `NEXT_PUBLIC_API_URL` / `VITE_API_URL` → URL base de la API del backend, terminado en `/api`.
- `VITE_LANDING_URL` → URL pública de la landing (usada por el admin para enlaces externos).

En Vercel, `NEXT_PUBLIC_*` y `VITE_*` se **inlinean en el bundle durante el build**: si la variable
no está definida en el dashboard, queda fijo el valor por defecto del código y los cambios de URL
no se aplican hasta el próximo deploy.

---

## Deploy en Vercel

Un solo repositorio, **dos proyectos de Vercel**, cada uno con su propio *Root Directory*:

| Proyecto Vercel | Root Directory | Framework | Output Directory |
| --- | --- | --- | --- |
| `quantum-fit-landing` | `quantum-fit-landing` | Next.js | (default) |
| `quantum-fit-admin` | `quantum-fit-admin` | Vite | `dist` |

El admin ya incluye [`quantum-fit-admin/vercel.json`](./quantum-fit-admin/vercel.json) con el
`rewrite` de SPA (`/(.*)` → `/index.html`) para que el React Router funcione en rutas profundas.

### Build local (verificación previa)

```bash
cd quantum-fit-landing && npm run build     # genera .next/
cd quantum-fit-admin  && npm run build     # genera dist/
```

---

## Notas

- Las imágenes que sube el admin se almacenan en el backend (Multer → `uploads/`). Ese directorio
  necesita un **volume persistente** en Railway; en el deploy, el volume debe montarse en
  `/app/uploads` (el `WORKDIR` del Dockerfile del backend).
- La landing consume la API **desde el navegador** (client-side `fetch`), no desde el server. Por
  eso su build en Vercel no depende de que el backend esté online.
