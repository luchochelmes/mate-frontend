# Mate — Guía para llevarlo a producción

Este doc explica cómo pasar Mate de "prototipo local con localStorage" a "app real con base de datos multiusuario en Vercel".

**Objetivo:** que cualquier usuario pueda crear cuenta, loguearse desde cualquier navegador/dispositivo, y ver sus mismos datos (materias, apuntes, chat, etc).

**Trabajo total estimado:** 5-7 horas repartidas en 4 fases. Se puede hacer en varias sesiones.

**Costo:** $0/mes mientras estés en free tiers (Supabase free + Vercel Hobby + Groq gratis).

---

## Estado actual (localStorage)

Hoy, Mate guarda **todo por usuario y por navegador** usando `localStorage`. Claves:

- `mate_accounts` — array de usuarios registrados (con password en texto plano ❌ no seguro)
- `mate_current_email` — sesión activa
- `mate_[email]_materias`
- `mate_[email]_notas`
- `mate_[email]_uploads` — archivos en base64
- `mate_[email]_chatConvos`
- `mate_[email]_eventos`
- `mate_[email]_horarios`
- `mate_[email]_pomoHours`
- `mate_[email]_cuatri`
- `mate_backend_url` — URL global del backend de chat

**Problemas de este modelo:**

- Los datos viven **solo en tu navegador**. Cambiás de dispositivo → perdés todo.
- Los passwords están en texto plano (nunca hacer esto en producción).
- No hay multiusuario real; cada persona ve sus datos solo si usa el mismo navegador.
- Los archivos base64 en LS son pesados (cuota ~5-10 MB por dominio).

---

## Fase A — Crear la base de datos en Supabase

**Tiempo estimado:** 30-45 min · Complejidad: baja

### A.1 Crear el proyecto

1. Andá a https://supabase.com/dashboard
2. **New project** (usá la misma cuenta que ILLIOS o creá una nueva)
3. Nombre: `mate` · Region: `sa-east-1` (São Paulo) · Password del proyecto: guardala en un lugar seguro
4. Esperá 2-3 min a que aparezca el proyecto

### A.2 Crear las tablas (schema)

En Supabase Dashboard → **SQL Editor** → **New query** → pegar y ejecutar:

```sql
-- ============================================
-- MATE — schema inicial
-- ============================================

-- Perfiles (extienden auth.users con datos custom)
create table public.profiles (
  id uuid primary key references auth.users(id) on delete cascade,
  nombre text,
  apellido text,
  apodo text,
  carrera text,
  universidad text,
  created_at timestamp with time zone default now()
);

-- Materias
create table public.materias (
  id uuid primary key default gen_random_uuid(),
  user_id uuid not null references auth.users(id) on delete cascade,
  nombre text not null,
  catedra text default '',
  color text default '#4E6B33',
  estado text default 'cursando',
  final numeric,
  asis_on boolean default false,
  asis_p int default 0,
  asis_t int default 0,
  asis_min int default 75,
  created_at timestamp with time zone default now()
);
create index on materias(user_id);

-- Notas por materia
create table public.notas_materia (
  id uuid primary key default gen_random_uuid(),
  materia_id uuid not null references materias(id) on delete cascade,
  user_id uuid not null references auth.users(id) on delete cascade,
  nombre text not null,
  valor numeric not null,
  created_at timestamp with time zone default now()
);
create index on notas_materia(materia_id);

-- Apuntes por materia (texto largo)
create table public.apuntes (
  materia_id uuid primary key references materias(id) on delete cascade,
  user_id uuid not null references auth.users(id) on delete cascade,
  texto text default '',
  updated_at timestamp with time zone default now()
);

-- Eventos (parciales, finales, TPs)
create table public.eventos (
  id uuid primary key default gen_random_uuid(),
  user_id uuid not null references auth.users(id) on delete cascade,
  materia_id uuid references materias(id) on delete set null,
  fecha date not null,
  tipo text not null,
  titulo text not null,
  created_at timestamp with time zone default now()
);
create index on eventos(user_id, fecha);

-- Horarios (bloques semanales)
create table public.horarios (
  id uuid primary key default gen_random_uuid(),
  user_id uuid not null references auth.users(id) on delete cascade,
  dia int not null,
  hora_ini text not null,
  hora_fin text not null,
  materia text not null,
  aula text default '',
  color text default '#565B47',
  created_at timestamp with time zone default now()
);
create index on horarios(user_id);

-- Uploads (metadata; los binarios van en Storage)
create table public.uploads (
  id uuid primary key default gen_random_uuid(),
  user_id uuid not null references auth.users(id) on delete cascade,
  materia_id uuid references materias(id) on delete set null,
  materia_nombre text,
  catedra text,
  tipo text,
  anio text,
  filename text,
  storage_path text not null,
  mime text,
  descargas int default 0,
  created_at timestamp with time zone default now()
);
create index on uploads(user_id);

-- Conversaciones del chat
create table public.chat_convos (
  id uuid primary key default gen_random_uuid(),
  user_id uuid not null references auth.users(id) on delete cascade,
  materia_id uuid references materias(id) on delete cascade,
  role text not null check (role in ('user','assistant')),
  content text not null,
  provider text,
  pdf_name text,
  created_at timestamp with time zone default now()
);
create index on chat_convos(user_id, materia_id, created_at);

-- Horas de Pomodoro por materia
create table public.pomo_hours (
  materia_id uuid primary key references materias(id) on delete cascade,
  user_id uuid not null references auth.users(id) on delete cascade,
  horas numeric default 0
);

-- Config del cuatrimestre
create table public.cuatri (
  user_id uuid primary key references auth.users(id) on delete cascade,
  inicio date not null,
  fin date not null
);
```

### A.3 Habilitar Row Level Security (RLS)

**Crítico:** sin RLS, cualquier usuario podría leer datos de otros. Pegar y ejecutar:

```sql
-- Activar RLS en todas las tablas
alter table profiles       enable row level security;
alter table materias       enable row level security;
alter table notas_materia  enable row level security;
alter table apuntes        enable row level security;
alter table eventos        enable row level security;
alter table horarios       enable row level security;
alter table uploads        enable row level security;
alter table chat_convos    enable row level security;
alter table pomo_hours     enable row level security;
alter table cuatri         enable row level security;

-- Policy generica: cada usuario solo ve/modifica sus propias filas
create policy "own rows profiles"    on profiles      for all using (auth.uid() = id);
create policy "own rows materias"    on materias      for all using (auth.uid() = user_id);
create policy "own rows notas"       on notas_materia for all using (auth.uid() = user_id);
create policy "own rows apuntes"     on apuntes       for all using (auth.uid() = user_id);
create policy "own rows eventos"     on eventos       for all using (auth.uid() = user_id);
create policy "own rows horarios"    on horarios      for all using (auth.uid() = user_id);
create policy "own rows uploads"     on uploads       for all using (auth.uid() = user_id);
create policy "own rows chat"        on chat_convos   for all using (auth.uid() = user_id);
create policy "own rows pomo"        on pomo_hours    for all using (auth.uid() = user_id);
create policy "own rows cuatri"      on cuatri        for all using (auth.uid() = user_id);
```

### A.4 Crear el bucket de Storage para archivos

Dashboard → **Storage** → **New bucket**

- Name: `uploads`
- Public: **NO** (privado)
- File size limit: 4 MB

Después, en **Policies** del bucket, agregar:

```sql
-- Los usuarios solo pueden subir/leer archivos dentro de su propia carpeta
create policy "uploads select own"
  on storage.objects for select
  using (bucket_id = 'uploads' and auth.uid()::text = (storage.foldername(name))[1]);

create policy "uploads insert own"
  on storage.objects for insert
  with check (bucket_id = 'uploads' and auth.uid()::text = (storage.foldername(name))[1]);

create policy "uploads delete own"
  on storage.objects for delete
  using (bucket_id = 'uploads' and auth.uid()::text = (storage.foldername(name))[1]);
```

Convención de path: `<user_id>/<upload_id>.<ext>`.

### A.5 Habilitar Auth

Dashboard → **Authentication** → **Providers**

- **Email** habilitado. Por default pide verificación por email; para prototipo podés desactivar "Confirm email" y así los users se registran directo sin verificar.

### A.6 Copiar las credenciales

Dashboard → **Settings** → **API**

Necesitás dos valores:

- `Project URL` (algo como `https://xxx.supabase.co`)
- `anon` public key

Guardalos, los vas a usar en la Fase B.

---

## Fase B — Migrar el frontend a Supabase

**Tiempo estimado:** 4-6 horas · Complejidad: alta

Esta es la fase más pesada. Hay que reemplazar ~20 puntos del código que leen/escriben `localStorage` por llamadas a Supabase.

### B.1 Agregar el SDK de Supabase

En `Mate.dc.html`, dentro del `<head>`, agregar:

```html
<script type="module">
  import { createClient } from 'https://cdn.jsdelivr.net/npm/@supabase/supabase-js@2/+esm';
  window.mateSb = createClient('__SUPABASE_URL__', '__SUPABASE_ANON_KEY__');
</script>
```

En Vercel, esos placeholders se reemplazan en build/runtime con las env vars (ver Fase C).

### B.2 Reemplazar el auth fake

Localizar `doAuth` en el DCLogic. Reemplazar la lógica de `localStorage.getItem('mate_accounts')` por:

```js
// Registro
const { data, error } = await window.mateSb.auth.signUp({
  email: f.email,
  password: f.password,
  options: { data: { nombre: f.nombre, apellido: f.apellido, apodo: f.apodo, carrera: f.carrera, universidad: f.universidad } }
});
// Después: insert en profiles con esos datos

// Login
const { data, error } = await window.mateSb.auth.signInWithPassword({ email: f.email, password: f.password });

// Logout
await window.mateSb.auth.signOut();

// Chequear sesión al mount
const { data: { user } } = await window.mateSb.auth.getUser();
```

### B.3 Reemplazar los CRUD

Para cada entidad, cambiar `localStorage` por Supabase:

**Materias** (`loadUserData` / `guardarMat` / `confirmarDelMat`):
```js
// Leer
const { data } = await window.mateSb.from('materias').select('*, notas_materia(*)').eq('user_id', user.id);
// Crear
await window.mateSb.from('materias').insert({ user_id: user.id, nombre, catedra, color, estado, asis_on, asis_p, asis_t, asis_min });
// Editar
await window.mateSb.from('materias').update({...}).eq('id', id);
// Borrar
await window.mateSb.from('materias').delete().eq('id', id);
```

**Eventos, Horarios, Chat, Cuatri, Notas, Pomo hours** — mismo patrón.

**Apuntes** — `upsert` con `materia_id` como PK.

**Uploads** — usar Supabase Storage:
```js
// Subir
const path = `${user.id}/${crypto.randomUUID()}.${ext}`;
await window.mateSb.storage.from('uploads').upload(path, file);
await window.mateSb.from('uploads').insert({ user_id, materia_id, filename, storage_path: path, mime, tipo, anio, catedra });
// Descargar
const { data } = await window.mateSb.storage.from('uploads').createSignedUrl(row.storage_path, 60);
window.open(data.signedUrl);
```

### B.4 Botón "Migrar datos locales a la nube" (opcional)

En Configuración, agregar un botón que:

1. Lee todo el `localStorage` del usuario actual
2. Para cada tabla, hace un `insert` bulk en Supabase
3. Sube los archivos base64 al bucket
4. Marca `mate_migrated_at` en LS para no correrlo dos veces

Útil solo si estuviste cargando datos en localhost y no querés perderlos al mudarte a la nube.

### B.5 Loading states + errores

Todos los reads/writes ahora son async. Hay que agregar `chatSending`-style flags en cada acción (`materiaSaving`, `eventoSaving`, etc) y manejar errores con `try/catch` mostrando mensajes claros al usuario.

---

## Fase C — Desplegar el frontend en Vercel

**Tiempo estimado:** 20 min · Complejidad: baja

### C.1 Preparar el repo

```powershell
cd C:\Users\Usuario\Desktop\mate-local
# ya está iniciado como repo git (rama redesign/inicio)
# antes de subir, merge a main
git checkout main
git merge redesign/inicio
```

### C.2 Crear repo en GitHub

1. https://github.com/new
2. Name: `mate-frontend` · Privado
3. Copiar la URL: `https://github.com/TU-USUARIO/mate-frontend.git`

### C.3 Push

```powershell
git remote add origin https://github.com/TU-USUARIO/mate-frontend.git
git push -u origin main
```

Auth: si te pide password, usá un Personal Access Token de GitHub (mismo procedimiento que hiciste para `mate-backend`).

### C.4 Import en Vercel

1. https://vercel.com/dashboard → **Add New → Project**
2. Import `mate-frontend`
3. **Framework Preset**: Other
4. **Root Directory**: `.`
5. **Build Command**: (dejar vacío)
6. **Output Directory**: (dejar vacío)
7. **Environment Variables**:
   - `SUPABASE_URL` = tu Project URL
   - `SUPABASE_ANON_KEY` = tu anon key
8. Deploy

### C.5 Inyectar env vars al HTML

Como es un sitio estático, las env vars no se leen desde el JS del navegador directamente. Dos opciones:

**Opción 1 (simple):** hacer un `/api/config.js` que devuelva las keys:

Crear `api/config.js`:
```js
export default function handler(req, res) {
  res.setHeader('Content-Type', 'application/javascript');
  res.send(`window.mateConfig = { supabaseUrl: '${process.env.SUPABASE_URL}', supabaseAnonKey: '${process.env.SUPABASE_ANON_KEY}' };`);
}
```

Y en `Mate.dc.html`, en el `<head>`:
```html
<script src="/api/config.js"></script>
<script type="module">
  import { createClient } from 'https://cdn.jsdelivr.net/npm/@supabase/supabase-js@2/+esm';
  window.mateSb = createClient(window.mateConfig.supabaseUrl, window.mateConfig.supabaseAnonKey);
</script>
```

**Opción 2:** build step con reemplazo (usá si migrás a Vite/Next después).

### C.6 Actualizar la URL del chat

En Configuración, la URL del backend de chat sigue apuntando a `mate-backend.vercel.app/api/chat`. Nada que cambiar; ya funciona con CORS `*`.

Si querés restringir CORS solo al dominio del frontend, en `mate-backend/api/chat.js` cambiar:
```js
res.setHeader('Access-Control-Allow-Origin', 'https://mate-frontend.vercel.app');
```

---

## Fase D — Post-deploy: pruebas end-to-end

**Tiempo estimado:** 15 min

1. Abrir la URL de Vercel del frontend en el navegador
2. Registrar un usuario nuevo
3. Verificar en Supabase Dashboard → **Table Editor** → `profiles` que la fila apareció
4. Cargar 2-3 materias, un evento, un apunte
5. Cerrar sesión → volver a loguearte → verificar que todo persiste
6. Abrir la app en otro navegador/dispositivo → loguearte → confirmar que ves los mismos datos
7. Chat: mandar un mensaje → confirmar que llega respuesta de Groq
8. Subir un PDF → descargarlo desde otro navegador (verifica Supabase Storage)

Si algo falla:
- Frontend: F12 → Console
- Backend chat: Vercel dashboard → mate-backend → Functions → ver logs
- Supabase: Dashboard → Logs → API/Database

---

## Post-deploy: cosas para pensar después

- **Email confirmations**: para producción real, activar "Confirm email" en Supabase Auth. Requiere configurar SMTP (Resend gratis funciona bien, como en ILLIOS).
- **Custom domain**: si querés `mate.tudominio.com`, configurar en Vercel → Domains.
- **Rate limiting** en el backend de chat para evitar abuso (`mate-backend` no tiene hoy).
- **Analytics**: Vercel Analytics es gratis, útil para ver cuántos usuarios entran.
- **Backups**: Supabase Pro tiene backups automáticos. Free tier no. Podés hacer `pg_dump` manual con la CLI.
- **Costos**: si superás 500MB de DB o 1GB de Storage, upgrade a Pro ($25/mes).
