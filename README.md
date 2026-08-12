# Registro ILATACC

Plataforma de registro de asistentes para eventos académicos, artísticos y culturales del **Instituto Latinoamericano de Ciencias Cinematográficas (ILATACC)**.

- **Repositorio:** https://github.com/ILATACC/registro
- **Producción (GitHub Pages):** https://ilatacc.github.io/registro/
- **Dominio final (pendiente de apuntar en Neubox):** https://registro.ilatacc.com.mx
- **Proyecto de Supabase:** `wvkmhezoriwbrnpicxsb`
- **Responsable del sitio web:** Cinematika Morelia, S.A. de C.V.

## Estructura

```
index.html                        → sitio público: cartelera, registro y boleto QR
admin.html                        → panel de administración (página aparte, con su propio login)
supabase/migrations/              → las 7 migraciones aplicadas a producción, en orden
supabase/functions/                → Edge Function real desplegada (correo + boleto QR)
.github/workflows/deploy.yml      → despliegue automático a Neubox por FTP
docs/guia-produccion.md            → guía completa de integración y protocolos de seguridad
```

> La URL del proyecto y la clave pública (`anon`/`publishable`) de Supabase están escritas
> directamente dentro de `index.html` y `admin.html` — es seguro tenerlas ahí porque su
> protección la da RLS, no el secretismo de la clave. La `service_role` key **nunca** vive
> en este repositorio.

## Migraciones aplicadas (en orden)

| # | Migración | Qué hace |
|---|---|---|
| 0001 | `init_schema` | Tablas, RLS, función `registrar_asistente()` |
| 0002 | `hardening_funciones` | `search_path` fijo y permisos correctos en funciones `SECURITY DEFINER` |
| 0003 | `webhook_confirmacion` | Trigger que llama a la Edge Function al registrarse |
| 0004 | `hardening_notificar_registro` | Bloquea la llamada directa al webhook interno |
| 0005 | `fix_es_admin_grants` | Corrige un bug real que rompía RLS para el público |
| 0006 | `contador_registrados` | Contador de aforo visible al público sin exponer la lista de asistentes |
| 0007 | `banners_storage` | Bucket de Storage para banners/fotos de eventos |
| 0008 | `edad_clasificacion_checkin` | Edad del asistente, clasificación cinematográfica, y check-in real (conteo en vivo) |
| 0009 | `clasificacion_obligatoria_cineteca` | Cineteca exige clasificación igual que Proyección Cinematográfica (reforzado con CHECK constraint) |
| 0010 | `auto_actualizar_updated_at` | `updated_at` ahora se refresca al editar un evento — usado para invalidar la caché de vista previa de WhatsApp/Facebook automáticamente |
| 0011 | `evento_cerrado_renta_privada` | Concepto "cerrado" (renta privada) independiente de "privado" (comunidad ILATACC) |
| 0012 | `codigo_acceso_evento_cerrado` | Código de acceso auto-generado — el auto-registro sigue abierto, pero exige este código en eventos cerrados |
| 0013 | `revoke_generar_codigo_acceso` | Bloquea la invocación directa de la función interna del disparador de códigos |
| 0014 | `evitar_duplicado_exacto_por_evento` | Evita que la misma persona (nombre+correo+teléfono exactos) se registre dos veces al mismo evento |
| 0015 | `edad_obligatoria` | La edad ahora es obligatoria también a nivel de base de datos, igual que nombre/correo/teléfono |
| 0016 | `genero_asistente` | Campo de género (opcional/sensible) — base para el futuro sistema de reportes demográficos |

## Edge Functions desplegadas

| Función | Qué hace |
|---|---|
| `enviar-confirmacion` | Correo de confirmación + boleto QR + enlace privado de videollamada cuando aplica (disparada por trigger de DB) |
| `evento-preview` | Genera etiquetas Open Graph por evento (imagen/título del banner) para que compartir en WhatsApp/Facebook/X muestre una vista previa correcta — WhatsApp no ejecuta JavaScript, así que el sitio principal (SPA) no puede generar esto por sí solo |

## Puesta en marcha en un proyecto nuevo


1. Crear un proyecto en [supabase.com](https://supabase.com).
2. Aplicar las migraciones de `supabase/migrations/` **en orden numérico** desde el SQL Editor.
   ⚠️ En `0003_webhook_confirmacion.sql`, reemplaza `REEMPLAZA_CON_TU_WEBHOOK_SECRET` por un
   secreto real antes de aplicarla (`python3 -c "import secrets; print(secrets.token_hex(24))"`).
3. Actualizar `SUPABASE_URL` y `SUPABASE_KEY` dentro de `index.html` y `admin.html` con los de tu proyecto.
4. Desplegar `supabase/functions/enviar-confirmacion/` como Edge Function (`verify_jwt = false`).
5. Configurar los secretos de esa función (Dashboard → Edge Functions → Manage secrets):
   - `WEBHOOK_SECRET` → el mismo valor que pusiste en el paso 2
   - `RESEND_API_KEY` → tu clave de [resend.com](https://resend.com)
6. Dar de alta al primer administrador: crear el usuario en *Authentication → Users*, copiar su UUID,
   e insertarlo en `perfiles` (`insert into perfiles (id, nombre, correo) values (...)`).
7. Subir el repositorio a GitHub y desplegar a Neubox (ver `docs/guia-produccion.md` §2–3).

## Seguridad

Toda la lógica de permisos vive en Row Level Security (RLS) del lado de Supabase, no en el
JavaScript del cliente. El panel de administración (`admin.html`) es una página completamente
separada del sitio público, con su propia verificación de sesión. Ver el checklist completo
en `docs/guia-produccion.md` §4.
