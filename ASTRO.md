# Implementación en Astro 5 / 6

Guía completa para integrar `@antoniovalls/cookieconsent` en un proyecto Astro. Cubre el caso estándar (MPA), View Transitions con `<ClientRouter>` y carga condicional de scripts de terceros.

---

## Índice

1. [Instalación](#1-instalación)
2. [Componente `CookieBanner.astro`](#2-componente-cookiebannerastro)
3. [Incluirlo en el Layout](#3-incluirlo-en-el-layout)
4. [Compatibilidad con View Transitions](#4-compatibilidad-con-view-transitions)
5. [Configuración con i18n del sitio](#5-configuración-con-i18n-del-sitio)
6. [Tema claro/oscuro siguiendo al sitio](#6-tema-clarooscuro-siguiendo-al-sitio)
7. [Carga condicional de Google Analytics 4](#7-carga-condicional-de-google-analytics-4)
8. [Botón para reabrir preferencias](#8-botón-para-reabrir-preferencias)
9. [TypeScript](#9-typescript)
10. [Variables CSS personalizadas](#10-variables-css-personalizadas)
11. [SSR vs estático](#11-ssr-vs-estático)
12. [Troubleshooting](#12-troubleshooting)

---

## 1. Instalación

```bash
pnpm add github:AntonioValls/cookieconsent
```

Para fijar versión (recomendado) cuando crees tags en tu repo:

```bash
pnpm add github:AntonioValls/cookieconsent#v3.1.0-av.1
```

Verifica que se ha añadido a `package.json`:

```jsonc
{
    "dependencies": {
        "@antoniovalls/cookieconsent": "github:AntonioValls/cookieconsent"
    }
}
```

---

## 2. Componente `CookieBanner.astro`

Crea `src/components/CookieBanner.astro`:

```astro
---
// CookieBanner.astro — sin props
---

<!-- Nodo donde se montará el modal. Importante para View Transitions. -->
<div id="cc-root" data-cookieconsent-root></div>

<script>
    import '@antoniovalls/cookieconsent/dist/cookieconsent.css';
    import * as CookieConsent from '@antoniovalls/cookieconsent';

    // Inicializa solo una vez por sesión, aunque el script se re-ejecute
    if (!window.__ccInitialized) {
        window.__ccInitialized = true;

        CookieConsent.run({
            root: '#cc-root',
            guiOptions: {
                consentModal: {
                    layout: 'box',
                    position: 'bottom right',
                    equalWeightButtons: true
                },
                preferencesModal: {
                    layout: 'box',
                    equalWeightButtons: true
                }
            },
            categories: {
                necessary: { enabled: true, readOnly: true },
                analytics: {
                    autoClear: { cookies: [{ name: /^_ga/ }, { name: '_gid' }] }
                }
            },
            language: {
                default: 'es',
                autoDetect: 'browser',
                translations: {
                    es: {
                        consentModal: {
                            title: 'Usamos cookies',
                            description: 'Este sitio usa cookies para mejorar tu experiencia. Puedes aceptar todas o gestionar tus preferencias.',
                            acceptAllBtn: 'Aceptar todas',
                            acceptNecessaryBtn: 'Solo necesarias',
                            showPreferencesBtn: 'Preferencias',
                            footer: '<a href="/privacidad">Política de privacidad</a>'
                        },
                        preferencesModal: {
                            title: 'Preferencias de cookies',
                            acceptAllBtn: 'Aceptar todas',
                            acceptNecessaryBtn: 'Solo necesarias',
                            savePreferencesBtn: 'Guardar selección',
                            closeIconLabel: 'Cerrar',
                            sections: [
                                {
                                    title: 'Uso de cookies',
                                    description: 'Las cookies necesarias son imprescindibles para que la web funcione. Las analíticas nos ayudan a entender cómo se usa.'
                                },
                                {
                                    title: 'Estrictamente necesarias',
                                    description: 'Imprescindibles para el funcionamiento del sitio.',
                                    linkedCategory: 'necessary'
                                },
                                {
                                    title: 'Analíticas',
                                    description: 'Estadísticas anónimas de uso del sitio.',
                                    linkedCategory: 'analytics'
                                }
                            ]
                        }
                    }
                }
            }
        });
    }
</script>
```

> El `<div id="cc-root">` y la opción `root: '#cc-root'` son críticos para View Transitions (sección 4). En MPA puro funciona igual sin ellos.

---

## 3. Incluirlo en el Layout

`src/layouts/BaseLayout.astro`:

```astro
---
import CookieBanner from '../components/CookieBanner.astro';
const { title } = Astro.props;
---

<!doctype html>
<html lang="es">
    <head>
        <meta charset="utf-8" />
        <meta name="viewport" content="width=device-width" />
        <title>{title}</title>
    </head>
    <body>
        <slot />
        <CookieBanner />
    </body>
</html>
```

Con esto el banner aparece en cualquier página que use `BaseLayout`.

---

## 4. Compatibilidad con View Transitions

Si usas `<ClientRouter />` (Astro 5/6) las navegaciones son client-side y el `<body>` se reemplaza, por lo que el modal del banner desaparecería en cada navegación.

**Solución**: marcar el contenedor del modal como persistente con `transition:persist`.

`CookieBanner.astro`:

```astro
---
---

<div id="cc-root" transition:persist></div>

<script>
    import '@antoniovalls/cookieconsent/dist/cookieconsent.css';
    import * as CookieConsent from '@antoniovalls/cookieconsent';

    const init = () => {
        if (window.__ccInitialized) return;
        window.__ccInitialized = true;

        CookieConsent.run({
            root: '#cc-root',
            // ... resto de la configuración
        });
    };

    // Primera carga
    init();

    // Si Astro re-ejecuta el script al volver con bfcache / SSR
    document.addEventListener('astro:page-load', init);
</script>
```

> El `transition:persist` evita que Astro reemplace el `<div id="cc-root">` durante la navegación, así el modal y sus listeners sobreviven entre páginas.

### Layout con `<ClientRouter />`

`BaseLayout.astro`:

```astro
---
import { ClientRouter } from 'astro:transitions';
import CookieBanner from '../components/CookieBanner.astro';
---

<!doctype html>
<html lang="es">
    <head>
        <ClientRouter />
        <!-- ... -->
    </head>
    <body>
        <slot />
        <CookieBanner />
    </body>
</html>
```

> Astro 5+ usa `<ClientRouter />` (en versiones anteriores se llamaba `<ViewTransitions />`).

---

## 5. Configuración con i18n del sitio

Si tu Astro tiene rutas multidioma (`/es/...`, `/en/...`), detecta el idioma actual y úsalo como `default`:

`CookieBanner.astro`:

```astro
---
const lang = Astro.currentLocale ?? 'es';
---

<div id="cc-root" transition:persist data-lang={lang}></div>

<script>
    import '@antoniovalls/cookieconsent/dist/cookieconsent.css';
    import * as CookieConsent from '@antoniovalls/cookieconsent';

    const root = document.getElementById('cc-root')!;
    const lang = root.dataset.lang ?? 'es';

    if (!window.__ccInitialized) {
        window.__ccInitialized = true;

        CookieConsent.run({
            root: '#cc-root',
            language: {
                default: lang,
                translations: {
                    es: { /* ... */ },
                    en: { /* ... */ },
                    ca: { /* ... */ }
                }
            },
            categories: {
                necessary: { enabled: true, readOnly: true },
                analytics: {}
            }
        });
    }
</script>
```

Para cambiar idioma cuando el usuario navega entre `/es` y `/en`:

```js
document.addEventListener('astro:after-swap', () => {
    const newLang = document.documentElement.lang;
    CookieConsent.setLanguage(newLang);
});
```

---

## 6. Tema claro/oscuro siguiendo al sitio

Si tu sitio alterna `class="dark"` en el `<html>`, sincroniza con el plugin (que espera `cc--darkmode`):

```js
const syncTheme = () => {
    const isDark = document.documentElement.classList.contains('dark');
    document.documentElement.classList.toggle('cc--darkmode', isDark);
};

syncTheme();

// Si tu app cambia el tema en runtime
new MutationObserver(syncTheme).observe(document.documentElement, {
    attributes: true,
    attributeFilter: ['class']
});
```

Pégalo dentro del mismo `<script>` del componente, debajo del `CookieConsent.run()`.

---

## 7. Carga condicional de Google Analytics 4

Patrón recomendado: cargar GA4 **solo** si el usuario acepta `analytics`. No uses `<script>` directamente en el `<head>`.

`CookieBanner.astro` (extendiendo el ejemplo anterior):

```astro
---
const GA_ID = 'G-XXXXXXXXXX';
---

<div id="cc-root" transition:persist data-ga={GA_ID}></div>

<script>
    import '@antoniovalls/cookieconsent/dist/cookieconsent.css';
    import * as CookieConsent from '@antoniovalls/cookieconsent';

    const gaId = document.getElementById('cc-root')!.dataset.ga!;

    const loadGA = async () => {
        await CookieConsent.loadScript(`https://www.googletagmanager.com/gtag/js?id=${gaId}`);
        window.dataLayer = window.dataLayer || [];
        function gtag(){ dataLayer.push(arguments); }
        gtag('js', new Date());
        gtag('config', gaId, { anonymize_ip: true });
    };

    if (!window.__ccInitialized) {
        window.__ccInitialized = true;

        CookieConsent.run({
            root: '#cc-root',
            categories: {
                necessary: { enabled: true, readOnly: true },
                analytics: {
                    autoClear: { cookies: [{ name: /^_ga/ }, { name: '_gid' }] },
                    services: {
                        ga: {
                            label: 'Google Analytics',
                            onAccept: loadGA
                        }
                    }
                }
            },
            language: { /* ... */ }
        });
    }
</script>
```

### Tracking de page views con View Transitions

Con `<ClientRouter />` no hay recargas, así que GA4 no contará los page views automáticamente. Hay que dispararlos manualmente:

```js
document.addEventListener('astro:page-load', () => {
    if (CookieConsent.acceptedService('Google Analytics', 'analytics') && window.gtag) {
        window.gtag('event', 'page_view', {
            page_path: location.pathname,
            page_title: document.title
        });
    }
});
```

---

## 8. Botón para reabrir preferencias

En tu `Footer.astro`:

```astro
<footer>
    <nav>
        <a href="/privacidad">Privacidad</a>
        <a href="/aviso-legal">Aviso legal</a>
        <button type="button" data-cc="show-preferencesModal">
            Configurar cookies
        </button>
    </nav>
</footer>
```

El atributo `data-cc="show-preferencesModal"` lo gestiona el plugin automáticamente — no necesitas JS adicional.

Otros valores útiles:
- `data-cc="show-consentModal"` — reabrir el banner
- `data-cc="accept-all"` — aceptar todas
- `data-cc="accept-necessary"` — solo necesarias
- `data-cc="accept-custom"` — aceptar la selección actual

---

## 9. TypeScript

El paquete incluye `types/index.d.ts`. Astro lo detecta automáticamente.

Si quieres declarar la propiedad global `__ccInitialized` que usamos:

`src/env.d.ts`:

```ts
/// <reference types="astro/client" />

declare global {
    interface Window {
        __ccInitialized?: boolean;
        dataLayer?: any[];
        gtag?: (...args: any[]) => void;
    }
}

export {};
```

Para tipar la configuración:

```ts
import type * as CookieConsent from '@antoniovalls/cookieconsent';

const config: Parameters<typeof CookieConsent.run>[0] = {
    categories: { necessary: { enabled: true, readOnly: true } },
    language: { default: 'es', translations: { es: { /* ... */ } } }
};
```

---

## 10. Variables CSS personalizadas

Sobreescribe las variables del plugin en tu CSS global (`src/styles/global.css`):

```css
:root {
    --cc-bg: #ffffff;
    --cc-primary-color: #18181b;
    --cc-secondary-color: #52525b;
    --cc-btn-primary-bg: #18181b;
    --cc-btn-primary-color: #fff;
    --cc-btn-primary-hover-bg: #27272a;
    --cc-btn-secondary-bg: #f4f4f5;
    --cc-btn-secondary-color: #18181b;
    --cc-btn-secondary-hover-bg: #e4e4e7;
    --cc-toggle-bg-on: #18181b;
    --cc-toggle-bg-off: #a1a1aa;
    --cc-toggle-knob-bg: #fff;
    --cc-overlay-bg: rgba(0, 0, 0, 0.65);
    --cc-modal-border-radius: 0.75rem;
    --cc-btn-border-radius: 0.5rem;
}

.cc--darkmode {
    --cc-bg: #09090b;
    --cc-primary-color: #fafafa;
    --cc-secondary-color: #a1a1aa;
    --cc-btn-primary-bg: #fafafa;
    --cc-btn-primary-color: #18181b;
    --cc-btn-secondary-bg: #27272a;
    --cc-btn-secondary-color: #fafafa;
}
```

Importa el CSS global en tu Layout:

```astro
---
import '../styles/global.css';
---
```

> El CSS del plugin se carga desde el `<script>` del componente (`import '@antoniovalls/cookieconsent/dist/cookieconsent.css'`). Tus overrides en `global.css` deben cargarse **después** para que ganen.

Si tienes problemas de orden, importa el CSS del plugin también en `global.css` al principio:

```css
@import '@antoniovalls/cookieconsent/dist/cookieconsent.css';

/* tus overrides aquí */
:root { /* ... */ }
```

Y quita el `import` del CSS del componente.

---

## 11. SSR vs estático

CookieConsent es **100% cliente**. No se ejecuta en el servidor — los `<script>` de Astro siempre corren en el navegador. No tienes que hacer nada especial:

- **`output: 'static'`**: funciona ✓
- **`output: 'server'` (SSR)**: funciona ✓
- **`output: 'hybrid'`**: funciona ✓

El plugin usa `localStorage` o cookies según `cookie.useLocalStorage`. En SSR no toques esto desde el servidor — déjalo en el `<script>` del componente.

---

## 12. Troubleshooting

### El modal aparece pero el CSS no se carga

Asegúrate de importar el CSS dentro del `<script>` del componente, **no** desde el frontmatter:

```astro
<!-- ❌ MAL -->
---
import '@antoniovalls/cookieconsent/dist/cookieconsent.css';
---

<!-- ✓ BIEN -->
<script>
    import '@antoniovalls/cookieconsent/dist/cookieconsent.css';
</script>
```

### Con View Transitions el banner desaparece al navegar

Falta `transition:persist` en el contenedor:

```astro
<div id="cc-root" transition:persist></div>
```

### Se inicializa varias veces / aparece dos veces

Falta el guard `window.__ccInitialized`. Sin View Transitions, Astro re-ejecuta el script en cada carga; con bfcache (volver atrás) puede dispararse otra vez.

### El idioma no cambia al navegar entre `/es` y `/en`

Hay que llamar a `setLanguage` en el evento `astro:after-swap`:

```js
document.addEventListener('astro:after-swap', () => {
    CookieConsent.setLanguage(document.documentElement.lang);
});
```

### TypeScript se queja de `Cannot find module '@antoniovalls/cookieconsent'`

Reinicia el servidor TypeScript del editor. En VS Code: `Cmd/Ctrl+Shift+P` → "TypeScript: Restart TS Server".

Si persiste, comprueba que `types/index.d.ts` está en el paquete instalado:

```bash
ls node_modules/@antoniovalls/cookieconsent/types/
```

### El analytics no se dispara en navegaciones internas

Con `<ClientRouter />` no hay recargas. Tienes que disparar `page_view` manualmente en `astro:page-load` (ver sección 7).

### El banner aparece detrás de otros elementos

Aumenta el `z-index` en tu CSS:

```css
.cc-overlay,
#cc-main {
    z-index: 9999 !important;
}
```

---

## Resumen rápido

Lo mínimo viable en Astro:

1. `pnpm add github:AntonioValls/cookieconsent`
2. Crear `src/components/CookieBanner.astro` con el snippet de la sección 2 (con `transition:persist` si usas `<ClientRouter />`).
3. Incluir `<CookieBanner />` antes de `</body>` en tu Layout.
4. Si usas analytics, configurar `services` con `onAccept` que llama a `CookieConsent.loadScript(...)`.
5. Si usas View Transitions, escuchar `astro:page-load` para mantener el plugin sincronizado con la nueva página.

Para el resto de opciones (categorías, traducciones, callbacks, API completa) consulta [USAGE.md](./USAGE.md).
