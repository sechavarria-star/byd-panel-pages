# byd-panel-pages — Panel BYD (appbyd.giwa-ia.com)

Panel del agente de WhatsApp de BYD: interacciones, clientes, modelos consultados y agendas,
separado en **Pre Venta** y **Post Venta**. La base y el resto del proyecto están en
`../byd/CLAUDE.md`.

Es el **primer proyecto de GIWA con Supabase Auth**.

## Cómo funciona

```
appbyd.giwa-ia.com (GitHub Pages, este index.html)
   │ supabase-js + publishable key (pública por diseño)
   │ login email + contraseña → sesión (JWT) que supabase-js guarda y renueva sola
   ▼
Supabase Giwa_Database, schema byd (expuesto en la API)
   ├─ lecturas: directo a las vistas v_*, con RLS por rol (../byd/supabase/sql/02-panel.sql)
   └─ usuarios: Edge Function byd-panel-usuarios (tiene la service_role)
```

- **No hay API intermedia para leer**: el navegador consulta las vistas y la seguridad la
  ponen las policies. Sin fila activa en `byd.usuarios_panel` no se ve nada, aunque el
  login ande (Auth es compartido por todo Giwa_Database).
- **Roles:** `admin` (todo + Usuarios), `preventa` (Resumen, Interesados, Modelos,
  Test drives, Conversaciones), `posventa` (Resumen, Vehículos, Taller, Conversaciones).
  El menú (`VISTAS`, `puede()`) solo acomoda la navegación: **el que corta es RLS**
  (`byd.puede_ver`). Si se agrega una vista, agregarle la policy en la base.
- **Solo lectura** en esta versión. Escribir (marcar test drive o service realizado)
  es la fase 2: va como funciones `security definer` que validen `byd.rol_actual()`.
- **Tres formas de entrar: Google, Microsoft (`azure`) o email + contraseña.** El registro
  libre está cerrado, así que Google/Microsoft **no crean usuarios**: un admin da de alta el
  email desde **Usuarios** (la contraseña es opcional; vacía = solo Google/Microsoft) y
  Supabase vincula la cuenta a ese usuario porque el email coincide y está confirmado. Una cuenta de Google no dada de alta
  vuelve con `?error_description=Signups not allowed…` y el login lo traduce.
- `flowType: 'pkce'`: Google vuelve con `?code=`, no con `#access_token=`, que chocaría con
  el ruteo por hash.
- **Mi contraseña** (pie del menú): cada usuario cambia la suya con `sb.auth.updateUser`, con su
  propia sesión, sin pasar por la Edge Function. Si entró solo con Google/Microsoft, le crea una
  y desde ahí puede entrar también con email.
- "Olvidé mi contraseña" por mail no está: el SMTP de Supabase manda muy pocos mails por
  hora. Si alguien la olvida, la resetea un admin desde Usuarios.
- **Google OAuth:** cliente "Web" en Google Cloud con redirect URI
  `https://ofhiogczijhzkuqdjwvz.supabase.co/auth/v1/callback`; el Client ID y el secret
  viven **solo** en Supabase (Authentication → Sign In / Providers → Google), nunca acá.
- **Microsoft OAuth:** App registration en Azure (Microsoft Entra), cuentas "de cualquier
  directorio y personales", redirect URI (Web) la misma de arriba. Client secret → Supabase
  (Providers → Azure, Tenant URL vacía = `common`). El front pide el scope `email`.
  **Token configuration → optional claims (ID token): `email` y `xms_edov`.** Sin `xms_edov`
  Supabase toma el email como no verificado, no lo vincula al usuario dado de alta y, con el
  registro cerrado, rechaza el ingreso.

## Vistas

| Vista | Fuente | Chips |
|---|---|---|
| Resumen | todas según rol + mapa de calor "Cuándo escriben" (`v_mensajes` de clientes, día × hora de Argentina, mismo diseño que Grun) | **filtro cruzado**: recuadros, celdas y modelos (ver abajo) |
| Interesados | `v_leads` | estado, motor, carrocería, presupuesto, uso |
| Modelos consultados | `v_modelos_consultados` | cómo (ficha / recomendado / elegido / test drive), motor, carrocería |
| Agenda de test drives | `v_testdrives` | sucursal, modelo, estado |
| Clientes y vehículos | `v_vehiculos` | estado del service, modelo, motor, turno |
| Agenda de taller | `v_turnos_service` | sucursal, estado |
| Conversaciones | `v_conversaciones` + `v_mensajes` | intención, etapa, tipo |
| Usuarios | Edge Function | — |

**Filtro cruzado del Resumen** (`RES`, `vResumen`): cada recuadro, celda del mapa de calor y
modelo del ranking es un grupo de personas (`wa_id`); los filtros se combinan (Y) y cada bloque
se calcula con los filtros de los *otros* bloques, así lo elegido queda resaltado sobre el resto
apagado. "Ver en…" lleva esas personas a Interesados, Test drives, Vehículos, Taller o
Conversaciones (`PERSONAS`, barra "Desde el Resumen"); "Limpiar" lo suelta. El cupo del taller
se cuenta siempre sobre todos los turnos, no sobre el filtro.

Los chips muestran cuántas filas quedan si se elige cada uno (idea del panel de Grun). Las
agendas son semanales, de lunes a sábado (idea de la agenda del Hospital).

## Estilos

Tokens y componentes copiados de `_Estilos.html` (design system GIWA): una página estática
no puede incluir el archivo compartido. Mismos nombres y valores; lo propio del panel está
al final del `<style>`.

## Probar en local

`preview_start` con `byd-panel-pages` (puerto **8800**; el 8080 es de ypf-panel-pages).
`http://localhost:8800` ya está en los orígenes de la Edge Function y en las Redirect URLs
de Auth. Antes de publicar: extraer el `<script>` y correr `node --check`.

## Deploy

Repo público `sechavarria-star/byd-panel-pages`, GitHub Pages desde `main` /root, `CNAME`
`appbyd.giwa-ia.com`. DNS en Squarespace: `CNAME appbyd → sechavarria-star.github.io`.
El repo puede ser público: la publishable key no da acceso a nada sin un usuario habilitado.
