# Hospital Ramón Exeni · Turnos de Neurología Infantil — Contexto técnico

Sistema de turnos del Hospital de Niños de San Justo: las familias sacan, ven y cancelan turnos
por **WhatsApp** y **Telegram**, y el hospital los gestiona desde un **backoffice web**.

Este documento es el punto de partida para cualquier persona que desarrolle sobre el sistema.
Leerlo entero antes del primer cambio.

> **Producción en uso.** El sistema atiende pacientes reales todos los días. Ningún cambio llega a
> producción sin pasar por el circuito de [Cómo publicar](#cómo-publicar).

---

## 1. Piezas y dónde vive cada una

| Pieza | Tecnología | Dónde corre | Quién la mantiene |
|---|---|---|---|
| Dominio `hospitalramonexeni.giwa-ia.com` | HTML estático (login Google + iframe) | GitHub Pages, **este repo** (`index.html`, `CNAME`) | Backoffice |
| Backoffice web | Google Apps Script | Web app de Apps Script | Backoffice |
| Conversor HTML → PDF | Google Apps Script | Web app de Apps Script | Backoffice |
| Base de datos | Supabase (Postgres 17), schema `wa` | Proyecto Supabase `hospital-neurologia` | Compartida |
| Agente WhatsApp + Flows | Supabase Edge Functions (Deno) + WhatsApp Flows de Meta | Supabase / Meta | Agente |
| Agente Telegram | Supabase Edge Function (Deno) | Supabase | Agente |
| Tareas programadas | n8n Cloud | n8n, proyecto Produccion | Agente / n8n |

Regla general: **cada pieza tiene un dueño**. Para tocar una pieza que no es tuya, coordinar antes
con quien la mantiene.

---

## 2. Arquitectura

```
Familias ──WhatsApp──▶ whatsapp-webhook ──▶ Flows (flow-endpoint) ─┐
         ──Telegram──▶ telegram-webhook ───────────────────────────┤
                                                                   ▼
                                                    Supabase · schema wa
                                                    (pacientes, turnos, …)
                                                      ▲            │ insert en wa.notificaciones
Hospital ──navegador──▶ hospitalramonexeni.giwa-ia.com │            ▼
                         └─ iframe ─▶ Backoffice (Apps Script)   enviar-notificacion ─▶ Telegram / WhatsApp
                                         │
                                         └─ responder-handoff ─▶ respuesta al paciente

n8n (horarios fijos): genera turnos · libera turnos · encola recordatorios · importa padrón ·
                       manda la agenda del día a cada médico (PDF por el conversor)
```

---

## 3. Base de datos (schema `wa`)

Tablas principales:

| Tabla | Qué guarda |
|---|---|
| `pacientes` | Padrón (≈151 mil del hospital + altas por WhatsApp/Telegram/web). `origen_alta`: `hospital`, `whatsapp`, `telegram`, `web_hospital` |
| `turnos` | Agenda. `estado`: Libre, Libre Hospital, Reservado, Confirmado, Presente, Atendido, Cancelado, Ausente, Bloqueado. `canal`: `whatsapp`, `telegram`, `web` |
| `lineas_pacientes` | Qué pacientes maneja cada línea (número de WhatsApp o `tg:<id>`) |
| `profesionales`, `especialidades`, `config_agendas` | Quién atiende, qué y cuándo |
| `licencias`, `feriados` | Días sin atención |
| `notificaciones` | Cola de avisos a familias (la consume `enviar-notificacion`) |
| `conversaciones`, `mensajes` | Hilos de chat de los dos canales (`canal` es columna generada) |
| `flow_sesiones`, `telegram_sesiones` | Estado de los formularios de cada canal |
| `usuarios`, `log_acciones` | Usuarios del backoffice y auditoría |

El backoffice lee algunas tablas a través de vistas `v_*_compat`. Hay vistas operativas como
`v_bandeja_handoff`, `v_avisos_pendientes` y `v_ciclo_confirmacion`.

**Cambios de esquema:** siempre como migración (`supabase/migrations` o `apply_migration`), aditivos y
coordinados con el dueño del agente. Varios objetos viejos se crearon desde el SQL Editor y no figuran
en el historial de migraciones.

---

## 4. Canales

### WhatsApp
- Entrada: `whatsapp-webhook` (agente de IA + derivación a los Flows de reserva, consulta y cancelación).
- Los avisos usan plantillas aprobadas por Meta. Fuera de la ventana de 24 h no se puede escribir texto libre.

### Telegram (desde noviembre 2026)
- Entrada: `telegram-webhook`. Menú con botones para reservar, ver y cancelar; mismo agente de IA.
- Identificador de línea: `tg:<telegram_user_id>`, en `pacientes.telegram_chat_id` y
  `lineas_pacientes`. **`pacientes.chat_id` es sólo WhatsApp.**
- Al compartir el contacto, si el teléfono ya era una línea de WhatsApp, se copian sus pacientes.

### Avisos y recordatorios (`enviar-notificacion`)
- Si el paciente tiene Telegram, el aviso va por Telegram; si falla, por WhatsApp.
- Recordatorio de 24 h con botones Confirmar / Cancelar en los dos canales.

### Atención humana (handoff)
- WhatsApp y Telegram derivan a la bandeja **Handoff** del backoffice; se responde desde ahí por
  `responder-handoff`.
- En Telegram sólo se activa con una palabra clave, y cada conversación también tiene un tema en un
  grupo privado del staff.

---

## 5. Backoffice (Apps Script)

Código: `backend/backoffice/` (cuando se suba a este repo).

| Archivo | Qué es |
|---|---|
| `Código.js` | Servidor: acceso a Supabase y todas las funciones que llama la pantalla |
| `Index.html` | La aplicación entera (HTML + CSS + JS) |
| `Ayuda.html` | Manual de uso, que se muestra en la pestaña Ayuda |
| `test/test_auth.js` | Prueba del control de acceso (Node, sin red) |

### Regla de seguridad (no negociable)

La web app es pública, y en Apps Script **cualquier función que no termine en `_` se puede llamar desde
el navegador**. Por eso:

1. Toda función sin `_` recibe `sid` como último parámetro y empieza con un control:
   - `_exigirAdmin_(sid);` — sólo usuarios con rol **admin**. Es el **default** para cualquier función.
   - `_exigirUsuario_(sid);` — cualquier usuario activo, también **recepción**. Sólo para lo que usa
     recepción, que ve únicamente el kanban de Estados: leer la agenda y cambiar el estado de un turno.
   Una función nueva va con `_exigirAdmin_`, salvo que recepción la necesite; en ese caso se agrega
   también a la lista `ABIERTAS` de `test/test_auth.js`.
2. El acceso a la base (`sbGet_`, `sbPost_`, `sbPatch_`, `sbDelete_`, `sbRpc_`) y todos los helpers
   terminan en `_`.
3. Toda llamada del front manda `window.__SID__` como último argumento.
4. Antes de publicar: `node test/test_auth.js .` tiene que dar `TODO OK` (prueba sin sesión, usuario
   ajeno, admin y recepción).

El rol se cachea 5 minutos: dar de baja a alguien o cambiarle el rol tarda hasta 5 minutos en aplicarse.

Las credenciales de Supabase se leen de las **Propiedades del script**, nunca del código.

---

## 6. Cómo trabajar

### Git
- `main` publica el sitio `hospitalramonexeni.giwa-ia.com`: **nadie empuja directo a `main`**.
- Una rama por cambio (`feat/…`, `fix/…`), pull request y revisión de otra persona.
- No tocar `index.html` ni `CNAME` salvo que el cambio sea del login o del dominio.

### Apps Script (clasp)
- Un solo Apps Script compartido. **Una persona a la vez** sube a HEAD: avisar en el equipo antes de
  hacer `clasp push`.
- Antes de editar: `clasp pull` y comparar con la versión que está en producción
  (`clasp list-deployments`, `clasp pull --versionNumber <n>`).
- El código del repo y el de Apps Script tienen que coincidir: lo que se publica sale del repo.

---

## 7. Cómo publicar

1. Cambio en una rama, con revisión.
2. `node test/test_auth.js .` → `TODO OK`. Extraer el JS de `Index.html` y correr `node --check`.
3. `clasp push` → sube a **HEAD**. Producción **no cambia**: está fija en su versión.
4. Probar en la URL de prueba (deployment `@HEAD`, sólo editores). **Usa la base real**: navegar y
   consultar; no escribir sobre datos reales.
5. Con el OK del responsable: `clasp create-deployment -i <deployment de producción> -d "qué cambió"`.
6. Volver atrás, si hace falta: `clasp create-deployment -i <deployment de producción> -V <versión anterior>`.
7. Merge del pull request a `main`.

---

## 8. Secretos

Ningún secreto va al repo. Dónde vive cada uno (sólo nombres):

| Dónde | Nombres |
|---|---|
| Propiedades del script (backoffice) | `SUPABASE_URL`, `SUPABASE_KEY` |
| Propiedades del script (conversor PDF) | `TOKEN` |
| Secrets de Supabase (Edge Functions) | `WHATSAPP_TOKEN`, `WHATSAPP_PHONE_NUMBER_ID`, `TELEGRAM_BOT_TOKEN`, `TELEGRAM_WEBHOOK_SECRET`, `TELEGRAM_STAFF_CHAT_ID`, `ANTHROPIC_API_KEY`, `BACKOFFICE_SECRET`, `NOTIF_WEBHOOK_SECRET`, `TPL_*` |
| Credenciales de n8n | Postgres, Gmail, Telegram de avisos internos |

Si aparece un secreto en el código o en un commit: avisar, **rotarlo** y sacarlo.

---

## 9. Datos sensibles

El sistema maneja datos de **menores** (nombre, DNI, fecha de nacimiento, contacto de la familia).

- Nunca subir al repo exportaciones, planillas, capturas con datos reales ni logs con datos de pacientes.
- Para probar, usar registros de prueba creados a propósito y borrarlos después.

---

## 10. Pendientes conocidos

- **Telegram en el backoffice:** ya están el contador y el filtro de origen, los colores por origen y
  el contacto "Tg: / Ws:" en Handoff. Falta: el canal de cada turno, el contacto según canal en Turnos
  y en la agenda impresa, el motivo de cancelación por Telegram, y en Handoff el encabezado del chat y
  no mostrar "Ventana 24h" en chats de Telegram.
- **Licencias de último momento:** al guardar una licencia, el front llama a `dispararFlujoN8n`, que
  no existe en el servidor de la v2. Confirmar con quien mantiene el agente qué cancela y avisa los
  turnos afectados.

### Decisiones tomadas

- Handoff: en el hilo se muestra "Operador", no el nombre de quien respondió.
- Handoff: "Devolver al bot" no le avisa al paciente de Telegram.
- Tokens escritos dentro de algunos workflows de n8n: pasarlos a credenciales y rotarlos.
