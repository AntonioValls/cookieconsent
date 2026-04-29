# Implementación de Cookie Consent en Astro

Documentación completa de la implementación del sistema de consentimiento de cookies en este proyecto Astro, con soporte para **Google Consent Mode v2**, **Google Analytics**, **Google Maps** y **View Transitions**.

---

## Dependencias

```bash
# Paquete de cookieconsent personalizado (fork privado)
pnpm add github:AntonioValls/cookieconsent
```

El paquete provee:
- `@antoniovalls/cookieconsent` — lógica JS del banner
- `@antoniovalls/cookieconsent/dist/cookieconsent.css` — estilos base del banner

---

## Estructura de archivos

```
src/
  components/
    CookieConsent.astro          ← Componente que monta el banner
  scripts/
    cookieconsent.js             ← Lógica: configuración, GA, Maps, Consent Mode
  styles/
    cookieconsent-custom.css     ← Sobrescritura de variables CSS del banner
```

---

## 1. Componente `CookieConsent.astro`

```astro
---
import "@antoniovalls/cookieconsent/dist/cookieconsent.css";
import "../styles/cookieconsent-custom.css";
---

<!-- Nodo donde CookieConsent monta su UI. Persiste en View Transitions. -->
<div id="cc-root" transition:persist></div>

<script>
  import "../scripts/cookieconsent.js";
</script>
```

**Puntos clave:**
- El `<div id="cc-root">` necesita `transition:persist` para que el banner no se destruya y reconstruya en cada navegación con View Transitions (`<ClientRouter />`).
- El CSS base se importa aquí, antes del CSS personalizado, para que la cascada funcione correctamente.

---

## 2. Incluir el componente en el Layout

En `src/layouts/Layout.astro`, importar y colocar `<CookieConsent />` **dentro del `<body>`**, después del contenido principal:

```astro
---
import { ClientRouter } from "astro:transitions";
import CookieConsent from "../components/CookieConsent.astro";
// ... otros imports
---

<!doctype html>
<html lang="es">
  <head>
    <!-- Importante para recargar el inicio -->
    <ClientRouter fallback="none" />    <!-- ← al final de head -->
  </head>
  <body>
    <Header />
    <main><slot /></main>
    <Footer />
    <CookieConsent />   <!-- ← al final del body -->
  </body>
</html>
```

> **Importante:** `<ClientRouter fallback="none" />` activa las View Transitions de Astro. Sin `transition:persist` en `#cc-root`, el banner se reiniciaría en cada navegación y volvería a pedirle consentimiento al usuario.

---

## 3. Script de configuración `cookieconsent.js`

Archivo completo en `src/scripts/cookieconsent.js`:

```js
import * as CookieConsent from "@antoniovalls/cookieconsent";

// ─── Google Consent Mode v2 ───────────────────────────────────────────────────
// Estado inicial: todo denegado hasta consentimiento explícito.
window.dataLayer = window.dataLayer || [];
window.gtag =
  window.gtag ||
  function gtag() {
    window.dataLayer.push(arguments);
  };

window.gtag("consent", "default", {
  ad_storage: "denied",
  analytics_storage: "denied",
  ad_user_data: "denied",
  ad_personalization: "denied",
  functionality_storage: "granted",
  security_storage: "granted",
});

// ─── Actualiza Consent Mode según categorías aceptadas ───────────────────────
function updateGoogleConsent() {
  const analyticsAccepted = CookieConsent.acceptedCategory("analytics");
  const marketingAccepted = CookieConsent.acceptedCategory("marketing");

  window.gtag("consent", "update", {
    analytics_storage: analyticsAccepted ? "granted" : "denied",
    ad_storage: marketingAccepted ? "granted" : "denied",
    ad_user_data: marketingAccepted ? "granted" : "denied",
    ad_personalization: marketingAccepted ? "granted" : "denied",
  });
}

// ─── Carga Google Analytics (solo si acepta analítica) ───────────────────────
function loadGoogleAnalytics() {
  if (window.__gaLoaded) return;

  const measurementId = "G-XXXXXXXXXX"; // ← sustituir por el ID real

  const script = document.createElement("script");
  script.async = true;
  script.src = `https://www.googletagmanager.com/gtag/js?id=${measurementId}`;
  document.head.appendChild(script);

  window.gtag("js", new Date());
  window.gtag("config", measurementId, { anonymize_ip: true });

  window.__gaLoaded = true;
}

// ─── Google Maps: carga / descarga iframes condicionalmente ──────────────────
function loadGoogleMapsEmbeds() {
  document.querySelectorAll("[data-map-iframe]").forEach((iframe) => {
    if (iframe.dataset.loaded === "true") return;
    if (iframe.dataset.src) {
      iframe.src = iframe.dataset.src;
      iframe.dataset.loaded = "true";
    }
  });
  document.querySelectorAll("[data-map-placeholder]").forEach((el) => {
    el.classList.add("hidden");
  });
}

function unloadGoogleMapsEmbeds() {
  document.querySelectorAll("[data-map-iframe]").forEach((iframe) => {
    iframe.removeAttribute("src");
    iframe.dataset.loaded = "false";
  });
  document.querySelectorAll("[data-map-placeholder]").forEach((el) => {
    el.classList.remove("hidden");
  });
}

function updateGoogleMapsConsent() {
  if (CookieConsent.acceptedCategory("marketing")) {
    loadGoogleMapsEmbeds();
  } else {
    unloadGoogleMapsEmbeds();
  }
}

// Botones "Aceptar para ver el mapa" dentro de cada placeholder
function initMapConsentButtons() {
  document.querySelectorAll("[data-map-consent-btn]").forEach((button) => {
    button.addEventListener("click", () => {
      CookieConsent.acceptCategory("marketing");
      loadGoogleMapsEmbeds();
      updateGoogleConsent();
      loadMarketingScripts();
    });
  });
}

// ─── Scripts de marketing (Meta Pixel, Google Ads, etc.) ─────────────────────
function loadMarketingScripts() {
  if (window.__marketingLoaded) return;

  // Añadir aquí los pixels o tags de marketing necesarios
  // Ejemplo: console.log("Marketing scripts cargados");

  window.__marketingLoaded = true;
}

// ─── Configuración principal de CookieConsent ────────────────────────────────
CookieConsent.run({
  root: "#cc-root",

  guiOptions: {
    consentModal: {
      layout: "box",
      position: "bottom right",
      equalWeightButtons: true,
      flipButtons: false,
    },
    preferencesModal: {
      layout: "box",
      position: "right",
      equalWeightButtons: true,
      flipButtons: false,
    },
  },

  categories: {
    necessary: {
      enabled: true,
      readOnly: true,
    },
    analytics: {
      enabled: false,
      autoClear: {
        cookies: [
          { name: /^_ga/ },
          { name: "_gid" },
        ],
      },
    },
    marketing: {
      enabled: false,
      autoClear: {
        cookies: [
          { name: /^_fbp/ },
          { name: /^_gcl/ },
        ],
      },
    },
  },

  language: {
    default: "es",
    translations: {
      es: {
        consentModal: {
          title: "Usamos cookies",
          description:
            "Utilizamos cookies propias y de terceros para garantizar el funcionamiento de la web, analizar el tráfico y, si nos das permiso, mejorar nuestras campañas de marketing.",
          acceptAllBtn: "Aceptar todas",
          acceptNecessaryBtn: "Rechazar opcionales",
          showPreferencesBtn: "Configurar",
          footer: `
            <a href="/politica-de-cookies/">Política de cookies</a>
            <a href="/politica-de-privacidad/">Política de privacidad</a>
          `,
        },
        preferencesModal: {
          title: "Configurar cookies",
          acceptAllBtn: "Aceptar todas",
          acceptNecessaryBtn: "Rechazar opcionales",
          savePreferencesBtn: "Guardar configuración",
          closeIconLabel: "Cerrar",
          sections: [
            {
              title: "Uso de cookies",
              description:
                "Puedes elegir qué categorías de cookies permites. Las cookies técnicas son necesarias para que la web funcione correctamente.",
            },
            {
              title: "Cookies necesarias",
              description:
                "Son imprescindibles para el funcionamiento básico de la web y no se pueden desactivar.",
              linkedCategory: "necessary",
            },
            {
              title: "Cookies de analítica",
              description:
                "Nos ayudan a entender cómo se utiliza la web para mejorar contenidos, navegación y rendimiento.",
              linkedCategory: "analytics",
            },
            {
              title: "Cookies de marketing",
              description:
                "Permiten medir campañas publicitarias y mostrar contenido o anuncios personalizados.",
              linkedCategory: "marketing",
            },
            {
              title: "Más información",
              description:
                'Puedes consultar todos los detalles en nuestra <a href="/politica-de-cookies/">Política de cookies</a>.',
            },
          ],
        },
      },
    },
  },

  // Callbacks: se ejecutan en el momento del consentimiento
  onFirstConsent: () => {
    updateGoogleConsent();
    updateGoogleMapsConsent();
    if (CookieConsent.acceptedCategory("analytics")) loadGoogleAnalytics();
    if (CookieConsent.acceptedCategory("marketing")) loadMarketingScripts();
  },

  onConsent: () => {
    updateGoogleConsent();
    updateGoogleMapsConsent();
    if (CookieConsent.acceptedCategory("analytics")) loadGoogleAnalytics();
    if (CookieConsent.acceptedCategory("marketing")) loadMarketingScripts();
  },

  onChange: () => {
    updateGoogleConsent();
    updateGoogleMapsConsent();
    if (CookieConsent.acceptedCategory("analytics")) loadGoogleAnalytics();
    if (CookieConsent.acceptedCategory("marketing")) loadMarketingScripts();
  },
});

// Reinicializar botones del mapa en cada navegación (View Transitions)
document.addEventListener("astro:page-load", () => {
  initMapConsentButtons();
  updateGoogleMapsConsent();
});
```

---

## 4. Personalización de estilos `cookieconsent-custom.css`

Sobrescribe las variables CSS del banner para que coincida con la identidad visual del proyecto:

```css
#cc-main {
  --cc-bg: #ffffff;
  --cc-primary-color: #111827;
  --cc-secondary-color: #4b5563;

  --cc-btn-primary-bg: #111827;
  --cc-btn-primary-color: #ffffff;
  --cc-btn-primary-border-color: #111827;
  --cc-btn-primary-hover-bg: #000000;
  --cc-btn-primary-hover-color: #ffffff;
  --cc-btn-primary-hover-border-color: #000000;

  --cc-btn-secondary-bg: #f3f4f6;
  --cc-btn-secondary-color: #111827;
  --cc-btn-secondary-border-color: #f3f4f6;
  --cc-btn-secondary-hover-bg: #e5e7eb;
  --cc-btn-secondary-hover-color: #111827;
  --cc-btn-secondary-hover-border-color: #e5e7eb;

  --cc-toggle-on-bg: #111827;
  --cc-toggle-off-bg: #d1d5db;
  --cc-toggle-on-knob-bg: #ffffff;
  --cc-toggle-off-knob-bg: #ffffff;
}

#cc-main .cm {
  box-shadow: 0 20px 60px rgba(15, 23, 42, 0.18);
}

#cc-main .cm__title,
#cc-main .pm__title {
  font-weight: 700;
}
```

---

## 5. Integrar Google Maps con consentimiento

Para cualquier iframe de Google Maps, usar `data-src` en lugar de `src` y añadir un placeholder de bloqueo:

```html
<!-- Placeholder visible mientras no hay consentimiento de marketing -->
<div
    data-map-placeholder
    class="absolute inset-0 z-10 flex flex-col items-center justify-center gap-3 bg-slate-100"
  >
    <svg
      width="32"
      height="32"
      fill="none"
      stroke="#15297f"
      stroke-linecap="round"
      stroke-linejoin="round"
      viewBox="0 0 24 24"
      xmlns="http://www.w3.org/2000/svg"
    >
      <path d="M12 12m-3 0a3 3 0 1 0 6 0 3 3 0 1 0-6 0"></path>
      <path
        d="M12 2a7 7 0 0 1 7 7c0 5.25-7 13-7 13S5 14.25 5 9a7 7 0 0 1 7-7"
      ></path>
    </svg>

    <p class="px-4 text-center text-sm text-slate-500">
      Acepta las cookies de terceros para ver el mapa.
    </p>

    <button
      type="button"
      data-map-consent-btn
      class="rounded-xl bg-autoliver-blue px-4 py-2 text-sm font-semibold text-white transition hover:bg-autoliver-dark-blue"
    >
      Aceptar y ver mapa
    </button>

    <button
      type="button"
      data-cc="show-preferencesModal"
      class="text-xs font-medium text-slate-500 underline underline-offset-4 hover:text-slate-700"
    >
      Configurar cookies
    </button>
  </div>

<!-- Iframe bloqueado hasta consentimiento -->
<iframe
  title="Ubicación xxxx"
  data-map-iframe
  data-src="https://www.google.com/maps/embed?pb=TU_EMBED_URL"
  data-loaded="false"
  width="600"
  height="450"
  style="border:0;"
  allowfullscreen=""
  loading="lazy"
  referrerpolicy="no-referrer-when-downgrade"
></iframe>
```

El script `loadGoogleMapsEmbeds()` transfiere `data-src` a `src` solo cuando el usuario acepta la categoría `marketing`, y `unloadGoogleMapsEmbeds()` elimina `src` si retira el consentimiento.

---

## 6. Configurar Google Analytics real

En `cookieconsent.js`, sustituir el placeholder:

```js
const measurementId = "G-XXXXXXXXXX"; // ← reemplazar por el ID real de GA4
```

---

## 7. Callbacks disponibles

| Callback | Cuándo se ejecuta |
|---|---|
| `onFirstConsent` | Primera vez que el usuario acepta o rechaza |
| `onConsent` | En cada carga de página cuando ya existe preferencia guardada |
| `onChange` | Cuando el usuario modifica sus preferencias en el modal |

Los tres deben actualizar `updateGoogleConsent()` y `updateGoogleMapsConsent()` para que el estado de los scripts refleje las preferencias actuales.

---

## 8. Categorías de cookies

| Categoría | `enabled` por defecto | `readOnly` | Limpia cookies al rechazar |
|---|---|---|---|
| `necessary` | `true` | `true` | No |
| `analytics` | `false` | `false` | `_ga*`, `_gid` |
| `marketing` | `false` | `false` | `_fbp*`, `_gcl*` |

---

## 9. Botón "Configurar cookies" en el Footer

CookieConsent expone el atributo `data-cc` para disparar acciones sin JS adicional. Úsalo en cualquier elemento del DOM, como el footer, para que el usuario pueda revisar sus preferencias en cualquier momento:

```astro
<button
  type="button"
  data-cc="show-preferencesModal"
  class="text-sm underline underline-offset-4 cursor-pointer"
>
  Configurar cookies
</button>
```

**Valores disponibles para `data-cc`:**

| Valor | Acción |
|---|---|
| `show-preferencesModal` | Abre el modal de configuración de categorías |
| `show-consentModal` | Vuelve a mostrar el banner inicial |
| `accept-all` | Acepta todas las categorías |
| `accept-necessary` | Rechaza todas las categorías opcionales |

No requiere ningún `addEventListener` extra; el paquete detecta estos atributos automáticamente al inicializarse.

---

## Notas importantes

- **`transition:persist`** en `#cc-root` es obligatorio cuando se usa `<ClientRouter />`. Sin él, el banner reaparecería en cada navegación.
- El evento **`astro:page-load`** sustituye a `DOMContentLoaded` en proyectos con View Transitions. Usarlo para reinicializar elementos del DOM que cambian entre páginas (como botones de mapa).
- **Google Consent Mode v2** se inicializa en modo `denied` antes de que cargue cualquier script de Google. Los scripts de GA y Maps solo se inyectan tras el consentimiento explícito.
- El flag `window.__gaLoaded` evita cargar el script de Analytics más de una vez por sesión.
