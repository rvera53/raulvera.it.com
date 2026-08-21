# Integrar un formulario de contacto con el MS de mensajería

> **Para quién es este documento:** el agente/desarrollador que va a conectar un sitio
> **estático** (GitHub Pages, Netlify, S3, etc. — sin backend) al microservicio de mensajería
> de CaVera Code.
>
> **Qué NO cubre:** instalar, desplegar o modificar el microservicio. El MS ya está corriendo
> en producción y lo opera Eduardo. Tu trabajo es **solo consumirlo vía HTTP** desde el sitio.
> No necesitas su código, su repo ni sus variables de entorno.

---

## 0. Tu configuración — ya está dada de alta

El MS no es abierto: requiere que el proyecto esté registrado del lado del servidor. **Eso ya
se hizo.** Estos son tus valores:

| Dato | Valor | Para qué |
|------|-------|----------|
| **Base URL** | `https://microservices.ochenta.com.mx` | El MS |
| **Cliente** | `raul-vera` | Cómo aparece tu proyecto en los logs del MS |
| **Token (API Key)** | *te lo pasa Eduardo por canal privado* | Va en el header `x-api-key` de cada request |
| **`senderDomain`** | `ochenta` | El único que tu token tiene permitido |
| **Remitente** | `robot@ochenta.com.mx` (default) | En `/send-mail` omite `from` y sale solo. Cualquier `from` que mandes debe ser `@ochenta.com.mx` |
| **Origins autorizados** | `https://raulvera.it.com`<br>`https://www.raulvera.it.com`<br>`http://localhost:5500`<br>`http://localhost:5173`<br>`http://localhost:3000` | Ya en la lista blanca de CORS |
| **Correo destino** | **lo decides tú** — el que sea (Gmail, etc.) | Va hardcodeado en tu JS como el campo `to`. No se configura en el MS |

El token **no está en este documento a propósito**: este archivo vive en un repo, y un token
en git es un token quemado. Pídeselo a Eduardo por mensaje directo y pégalo solo en el JS del
sitio.

### Sobre el Origin (lee esto con cuidado — es el error #1 de integración)

El "origin" es **esquema + host + puerto, sin ruta**. Cosas que importan en tu caso:

- Tu dominio propio `https://raulvera.it.com` y su variante `www` ya están dados de alta.
  Son origins **distintos** entre sí; por eso van los dos.
- **`https://raulvera.it.com` y `http://raulvera.it.com` también son origins distintos.**
  Solo el `https` está permitido. Activa **"Enforce HTTPS"** en Settings → Pages de tu repo;
  sin eso, alguien que entre por `http://` va a ver el formulario fallar.
- El origin `https://<tuusuario>.github.io` **no** está en la lista. Con dominio propio
  configurado, GitHub Pages redirige de `github.io` al dominio propio, así que no hace falta.
  Si necesitas probar en la URL de `github.io` antes de que propague el DNS, pídele a Eduardo
  que la agregue (dale tu usuario de GitHub).
- Para desarrollo local ya están `5500` (Live Server de VS Code), `5173` (Vite) y `3000`.
  Si usas otro puerto — `python -m http.server` usa `8000` — pídelo o cámbiate a uno de esos.

---

## 1. Advertencia importante: el token va a quedar expuesto

Un sitio en GitHub Pages es 100% estático: no hay servidor donde esconder un secreto.
Cualquier token que pongas en el JS **es visible** para quien abra las DevTools o el
repositorio. Eso es una limitación del hosting, no algo que puedas resolver con
ofuscación, `.env` de Vite, ni build steps — todo eso termina en el bundle público.

**Esto es una decisión ya tomada y aceptada del lado de CaVera para sitios estáticos.**
El riesgo está acotado así:

- El token es **por cliente**: solo puede mandar correo desde el `senderDomain` que le
  asignaron. No sirve para suplantar a otros clientes ni otros dominios.
- El MS tiene **rate limiting** por cliente (50 req / 15 min) y por IP (20 req / 15 min).
- Se puede **rotar** en cualquier momento (nueva env var, sin tocar código del MS).

**Lo que sí tienes que hacer:**

1. Tu token ya es **dedicado a este sitio** y solo puede mandar email desde `ochenta`.
   Si algo falla, no pidas "el token que sí funciona" — reporta el error.
2. **No** dejes que el usuario controle el campo `to`. Hardcodea el destinatario.
   Si `to` viene del formulario, el token expuesto se vuelve un relay de spam abierto.
3. Pon un **honeypot** (campo oculto que los humanos no llenan y los bots sí) — sin backend
   es tu defensa antispam más barata. Viene implementado en el ejemplo de abajo.

**Si en algún momento quieren el token realmente oculto**, la ruta es meter un proxy mínimo
(Cloudflare Worker o una Vercel Function gratis) que guarde el token del lado servidor y
reenvíe al MS. Es ~20 líneas. **No lo hagas ahora si no te lo piden** — está fuera de alcance
de esta integración; menciónalo como opción y sigue.

---

## 2. El contrato

**Base URL:** `https://microservices.ochenta.com.mx`

### Headers

| Header | Valor | Notas |
|--------|-------|-------|
| `Content-Type` | `application/json` | Siempre |
| `x-api-key` | tu token | Usa **este**. `mg-api-key` existe pero es legado, no lo uses en integraciones nuevas |

### Endpoints que te interesan

| Método | Ruta | Para qué |
|--------|------|----------|
| `GET` | `/health` | Verificar que el MS responde. No requiere token |
| `POST` | `/send-mail` | **El correo al dueño del sitio** con los datos del formulario |
| `POST` | `/send-confirmation` | (Opcional) Acuse de recibo al visitante que llenó el formulario |
| `POST` | `/preview` | Devuelve el HTML renderizado **sin enviar correo**. Úsalo para iterar el diseño sin gastar cuota |

---

## 3. `POST /send-mail` — el correo al dueño del sitio

### Campos

| Campo | Tipo | Req | Default | Notas |
|-------|------|-----|---------|-------|
| `to` | string \| string[] | ✅ | — | **Hardcodéalo.** Nunca del input del usuario |
| `subject` | string | — | `"Han pedido informes desde página web."` | |
| `html` | string | — | `"Sin mensaje."` | El cuerpo. Se inyecta en un template con header/footer |
| `template` | string | — | — | Alternativa a `html`: usa un template con nombre (ver §5) |
| `vars` | object | — | — | Variables del template |
| `from` | string | — | `robot@ochenta.com.mx` | **Omítelo.** Si lo mandas, debe terminar en `@ochenta.com.mx` (ver abajo) |
| `replyTo` | string | — | — | **Aquí va el correo del visitante**, para que "Responder" le conteste a él |
| `cc` | string \| string[] | — | — | |
| `logo` | string | — | `null` | URL **HTTPS** obligatoria |
| `footerMsg` | string | — | texto genérico | |
| `senderDomain` | string | — | `"ochenta"` | El que te asignaron |
| `attachments` | Attachment[] | — | — | 10 MB por archivo, 25 MB total |

`Attachment` = `{ "filename": "cv.pdf", "data": "<base64>", "contentType": "application/pdf" }`

### El `from`: no lo mandes, usa el default

Los correos salen desde **`robot@ochenta.com.mx`**, el remitente por defecto del MS. Es a
propósito: deja claro que el correo viene del microservicio de mensajería y no de un buzón
personal, y evita que tengas que verificar un dominio propio en Mailgun.

`from` **no puede ser el correo del visitante.** El MS valida que el remitente pertenezca al
dominio del `senderDomain` (para `ochenta`: `@ochenta.com.mx`) y responde `400` si no. Es la
regla que impide que un token filtrado mande correo "desde" el dominio de otro cliente.

```jsonc
// ✅ recomendado en /send-mail: omite `from`, sale de robot@ochenta.com.mx
{ "to": "TU_CORREO_DESTINO", "replyTo": "juan@gmail.com" }

// ❌ 400 — "El remitente debe usar el dominio @ochenta.com.mx."
//    El correo del visitante va en replyTo, nunca en from.
{ "from": "juan@gmail.com", "to": "TU_CORREO_DESTINO" }

// ❌ 400 — tampoco tu dominio propio: no está verificado en Mailgun
{ "from": "hola@raulvera.it.com", "to": "TU_CORREO_DESTINO" }
```

**En `/send-confirmation` sí es obligatorio** (ahí no hay default que aplicar). Pasa el mismo
remitente, pero con nombre visible para que al visitante no le llegue un `robot@…` pelón:

```jsonc
{ "from": "Raúl Vera <robot@ochenta.com.mx>", "to": "juan@gmail.com" }
```

El nombre antes de los `<>` es libre; lo único que el MS valida es la parte del `@`.

### Respuestas

| Status | Body | Qué significa / qué hacer |
|--------|------|---------------------------|
| `200` | `{ status: 200, success: true, response: {...} }` | Enviado |
| `400` | `{ success: false, error: "..." }` | Payload inválido: falta `to`, email mal formado, `from` de dominio no permitido, logo sin HTTPS, template desconocido |
| `401` | `{ error: "API Key requerida." \| "API Key no válida." }` | Token ausente o mal. Revisa el header |
| `403` | `{ success: false, error: "Tu API Key no tiene permiso para usar el dominio \"x\"." }` | Mandaste un `senderDomain` que tu token no tiene. Usa el asignado |
| `429` | `{ success: false, error: "Demasiadas solicitudes..." }` | Rate limit (ver §7) |
| `500` | `{ success: false, error: "..." }` | Falló Mailgun |
| `503` | `{ success: false, error: "Dominio x no está configurado..." }` | Problema del lado del MS — avísale a Eduardo |

---

## 4. `POST /send-confirmation` — acuse de recibo al visitante (opcional)

Correo brandeado con headline, mensaje y botón CTA opcional.

| Campo | Tipo | Req | Notas |
|-------|------|-----|-------|
| `to` | string | ✅ | Aquí **sí** va el correo del visitante |
| `subject` | string | ✅ | |
| `from` | string | ✅ | **Requerido aquí** (a diferencia de `/send-mail`, donde se omite). Usa `Raúl Vera <robot@ochenta.com.mx>` |
| `headline` | string | ✅ | Título grande |
| `message` | string | ✅ | Párrafo |
| `primaryColor` | string | — | Hex, para header y botón. Usa el color de la marca del sitio |
| `logoUrl` | string | — | HTTPS obligatorio |
| `ctaText` / `ctaUrl` | string | — | Botón. Van juntos |
| `footerMsg`, `senderDomain`, `replyTo`, `cc`, `attachments` | — | — | Igual que `/send-mail` |

`template` + `vars` también aplican aquí (ver §5).

---

## 5. Templates con nombre (recomendado)

En lugar de armar el HTML a mano, puedes pedir un template registrado y mandar solo los datos.
Menos código de tu lado y el copy queda consistente.

| Template | Endpoint | `vars` |
|----------|----------|--------|
| `contacto-recibido` | `/send-mail` | `{ campos: { "Etiqueta": "valor", ... } }` — arma una tabla con lo que le pases, sin campos fijos |
| `gracias-por-contactar` | `/send-confirmation` | `{ nombre?: string }` — personaliza el saludo; si lo omites usa copy genérico |

Cualquier campo explícito del body (`subject`, `html`, `headline`, `message`) **gana** sobre el
default del template. Pedir un template inexistente o del endpoint equivocado → `400`.

```jsonc
// /send-mail con template
{
  "to": "TU_CORREO_DESTINO",
  "replyTo": "juan@gmail.com",
  "template": "contacto-recibido",
  "vars": { "campos": { "Nombre": "Juan Pérez", "Email": "juan@gmail.com", "Mensaje": "Hola" } }
}
```

> ⚠️ **Escapa el input del usuario.** Los valores de `vars.campos` (y cualquier `html` que
> mandes) se insertan en el HTML del correo **tal cual**. No es una vulnerabilidad del sitio,
> pero un visitante puede meter markup en el correo que recibe el dueño. Pasa cada valor por
> un `escapeHtml()` antes de mandarlo — está en el ejemplo de abajo.

---

## 6. Implementación de referencia (JS puro, sin dependencias)

Funciona tal cual en GitHub Pages. Si el sitio usa React/Vue, la función `enviarFormulario`
es idéntica; solo cambia cómo lees los campos y cómo pintas el estado.

### HTML

```html
<form id="contact-form" novalidate>
  <label>Nombre <input name="nombre" type="text" required autocomplete="name" /></label>
  <label>Email <input name="email" type="email" required autocomplete="email" /></label>
  <label>Teléfono <input name="telefono" type="tel" autocomplete="tel" /></label>
  <label>Mensaje <textarea name="mensaje" rows="5" required></textarea></label>

  <!-- Honeypot antispam: oculto para humanos, los bots lo llenan.
       Usa CSS (no `type="hidden"`, los bots lo detectan). -->
  <div class="hp" aria-hidden="true">
    <label>No llenes este campo <input name="website" tabindex="-1" autocomplete="off" /></label>
  </div>

  <button type="submit" id="submit-btn">Enviar</button>
  <p id="form-status" role="status" aria-live="polite"></p>
</form>

<style>
  .hp { position: absolute; left: -9999px; opacity: 0; height: 0; overflow: hidden; }
</style>
```

### JS

```js
// ── Config ────────────────────────────────────────────────────────────────
// El token es público por necesidad (sitio estático, sin backend). Ver §1.
const MS_URL = 'https://microservices.ochenta.com.mx';
const API_KEY = 'PEGA_AQUI_EL_TOKEN';        // te lo manda Eduardo por privado
const DESTINO = 'PON_AQUI_TU_CORREO';        // hardcodeado a propósito, nunca del form
const SENDER_DOMAIN = 'ochenta';             // el asignado a tu token
const NOMBRE_SITIO = 'Raúl Vera';
const ENVIAR_ACUSE = true;                   // correo de "gracias" al visitante

// Remitente. En /send-mail se OMITE (el MS pone robot@ochenta.com.mx solo);
// en /send-confirmation es obligatorio, por eso la constante.
const REMITENTE = `${NOMBRE_SITIO} <robot@ochenta.com.mx>`;

// ── Utilidades ────────────────────────────────────────────────────────────
const escapeHtml = (s) =>
  String(s ?? '')
    .replace(/&/g, '&amp;')
    .replace(/</g, '&lt;')
    .replace(/>/g, '&gt;')
    .replace(/"/g, '&quot;')
    .replace(/'/g, '&#39;');

async function postMS(ruta, payload, { timeoutMs = 15000 } = {}) {
  const ctrl = new AbortController();
  const t = setTimeout(() => ctrl.abort(), timeoutMs);
  try {
    const res = await fetch(`${MS_URL}${ruta}`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json', 'x-api-key': API_KEY },
      body: JSON.stringify(payload),
      signal: ctrl.signal,
    });
    // El MS siempre responde JSON, pero un 429/proxy puede devolver otra cosa.
    const data = await res.json().catch(() => ({}));
    if (!res.ok) throw new Error(data.error || `HTTP ${res.status}`);
    return data;
  } finally {
    clearTimeout(t);
  }
}

// ── Envío ─────────────────────────────────────────────────────────────────
const form = document.getElementById('contact-form');
const btn = document.getElementById('submit-btn');
const status = document.getElementById('form-status');

form.addEventListener('submit', async (e) => {
  e.preventDefault();
  if (btn.disabled) return;

  const fd = new FormData(form);
  const nombre = (fd.get('nombre') || '').toString().trim();
  const email = (fd.get('email') || '').toString().trim();
  const telefono = (fd.get('telefono') || '').toString().trim();
  const mensaje = (fd.get('mensaje') || '').toString().trim();
  const honeypot = (fd.get('website') || '').toString().trim();

  // Bot detectado: finge éxito y no gastes cuota del MS.
  if (honeypot) {
    status.textContent = '¡Gracias! Tu mensaje fue enviado.';
    form.reset();
    return;
  }

  if (!nombre || !email || !mensaje) {
    status.textContent = 'Completa nombre, email y mensaje.';
    return;
  }
  if (!/^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(email)) {
    status.textContent = 'Revisa tu correo electrónico.';
    return;
  }

  btn.disabled = true;
  status.textContent = 'Enviando…';

  try {
    // 1) Aviso al dueño del sitio. `to` hardcodeado, visitante en replyTo.
    await postMS('/send-mail', {
      to: DESTINO,
      replyTo: email,
      senderDomain: SENDER_DOMAIN,
      subject: `Nuevo mensaje de ${nombre} desde ${NOMBRE_SITIO}`,
      template: 'contacto-recibido',
      vars: {
        campos: {
          Nombre: escapeHtml(nombre),
          Email: escapeHtml(email),
          ...(telefono && { 'Teléfono': escapeHtml(telefono) }),
          Mensaje: escapeHtml(mensaje),
        },
      },
    });

    // 2) Acuse al visitante. Si falla, no rompe el flujo: el mensaje ya llegó.
    if (ENVIAR_ACUSE) {
      postMS('/send-confirmation', {
        to: email,
        from: REMITENTE,   // requerido en este endpoint (en /send-mail se omite)
        senderDomain: SENDER_DOMAIN,
        template: 'gracias-por-contactar',
        vars: { nombre },
        primaryColor: '#1a1a1a',
        footerMsg: `${NOMBRE_SITIO} — este es un correo automático, no lo respondas.`,
      }).catch((err) => console.warn('Acuse no enviado:', err));
    }

    form.reset();
    status.textContent = '¡Gracias! Tu mensaje fue enviado.';
  } catch (err) {
    console.error('[contacto]', err);   // detalle técnico solo en consola
    status.textContent = 'No pudimos enviar tu mensaje. Escríbenos a ' + DESTINO;
  } finally {
    btn.disabled = false;
  }
});
```

**Notas de la implementación:**

- El acuse (`/send-confirmation`) se dispara **sin `await`**: si falla, el visitante no ve un
  error — el mensaje al dueño ya se envió, que es lo que importa.
- El `catch` muestra un mensaje genérico + el correo de respaldo. Nunca pintes `err.message`
  en la UI: puede filtrar detalles del MS.
- `btn.disabled` evita dobles envíos (que además consumen cuota de rate limit).

---

## 7. Rate limiting — lo que tienes que saber

| Límite | Ventana | Se cuenta por |
|--------|---------|---------------|
| 20 requests | 15 min | **IP del visitante** |
| 50 requests | 15 min | Tu token (sumando todos los visitantes) |

Dos detalles que confunden al depurar:

1. **Cada envío cuenta como 2 requests contra el límite por IP.** El header `x-api-key` es
   custom, así que el navegador manda un preflight `OPTIONS` antes del `POST`. En la práctica
   son ~10 envíos por visitante cada 15 min — de sobra para un formulario, pero se agota
   rápido si estás probando a mano. Usa `/preview` para iterar (ver §8).

2. **Un `429` se ve en el navegador como un error de CORS.** El rate limiter corre antes del
   middleware de CORS, así que la respuesta 429 sale sin los headers `Access-Control-Allow-*`
   y el navegador reporta un error de CORS genérico en vez del 429 real.
   **Si `GET /health` responde bien pero tu POST da "CORS error", casi siempre es rate limit,
   no configuración.** Espera 15 minutos antes de asumir que el origin está mal dado de alta.

---

## 8. Probar sin gastar cuota: `POST /preview`

Mismo payload que `/send-mail` o `/send-confirmation`, más `type`. Corre **todas** las
validaciones (dominio, `from`, email, templates) pero **no manda correo**. Devuelve
`{ success: true, html: "<...>" }`.

```js
const { html } = await postMS('/preview', {
  type: 'send-mail',                // 'send-mail' (default) | 'send-confirmation'
  to: DESTINO,
  senderDomain: SENDER_DOMAIN,
  template: 'contacto-recibido',
  vars: { campos: { Nombre: 'Prueba', Mensaje: 'Hola' } },
});
document.write(html);               // o pégalo en un <iframe srcdoc>
```

Es la forma correcta de validar que el payload es válido y de ver cómo se ve el correo antes
de mandar el primero de verdad.

### Con curl

```bash
# ¿El MS está vivo? (sin token)
curl https://microservices.ochenta.com.mx/health

# ¿Mi token y mi payload sirven? (sin enviar correo)
curl -X POST https://microservices.ochenta.com.mx/preview \
  -H "Content-Type: application/json" \
  -H "x-api-key: TU_TOKEN" \
  -d '{"type":"send-mail","to":"TU_CORREO_DESTINO","template":"contacto-recibido","vars":{"campos":{"Nombre":"Prueba"}}}'
```

> `curl` **no manda header `Origin`**, así que no valida CORS. Que curl funcione y el navegador
> no = problema de origin no dado de alta. Es la mejor forma de aislar el problema.

---

## 9. Checklist de entrega

- [ ] Token recibido de Eduardo (por privado, no por el repo)
- [ ] Correo destino puesto en la constante `DESTINO` (lo eliges tú, no se pide a nadie)
- [ ] "Enforce HTTPS" activado en Settings → Pages
- [ ] `GET /health` responde `{ status: "ok", ... }`
- [ ] `POST /preview` con tu token responde `200` (valida token + payload)
- [ ] Envío real desde el sitio **publicado** (no solo local) llega a la bandeja
- [ ] "Responder" en ese correo contesta al visitante (`replyTo` bien puesto)
- [ ] `to` está hardcodeado, no viene del formulario
- [ ] Honeypot presente
- [ ] Botón se deshabilita durante el envío
- [ ] Errores muestran mensaje genérico + correo de respaldo, no `err.message`
- [ ] Valores del usuario pasan por `escapeHtml()`
- [ ] Si cambias de dominio o agregas un subdominio, avisar a Eduardo **antes** (cambia el origin)

---

## 10. Diagnóstico rápido

| Síntoma | Causa probable |
|---------|----------------|
| "CORS error" en consola, `/health` funciona | Origin no dado de alta, **o** un `429` disfrazado (§7). Prueba con curl para distinguir |
| `401 API Key requerida` | Falta el header `x-api-key`, o lo pusiste en el body |
| `401 API Key no válida` | Token con typo/espacio, o aún no configurado del lado del MS |
| `403 ... no tiene permiso para usar el dominio` | `senderDomain` distinto al asignado. Quítalo o usa el correcto |
| `400 El remitente debe usar el dominio @...` | Pusiste el correo del visitante en `from`. Va en `replyTo` (§3) |
| `400 Destinatario inválido` | `to` mal formado o vacío |
| `400 Template desconocido` | Nombre mal escrito, o template de `/send-confirmation` usado en `/send-mail` |
| `400 El logo debe ser una URL HTTPS` | URL con `http://` o relativa. Tiene que ser HTTPS absoluta |
| Manda `200` pero no llega el correo | Revisa spam. Si sigue sin aparecer, es del lado de Mailgun — avísale a Eduardo con la hora del envío |

Cualquier cosa que apunte al MS mismo (`503`, `500`, correos que salen `200` pero no llegan)
**no la arregles tú** — repórtala. El MS es un servicio compartido entre varios proyectos.
