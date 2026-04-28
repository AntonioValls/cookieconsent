# Manual de uso — @antoniovalls/cookieconsent

Manual completo y autosuficiente del paquete (basado en `vanilla-cookieconsent` v3.1.0). Pensado para no depender de la documentación online del proyecto original.

---

## Índice

1. [Instalación](#1-instalación)
2. [Configuración mínima](#2-configuración-mínima)
3. [Estructura del objeto `run()`](#3-estructura-del-objeto-run)
4. [Categorías](#4-categorías)
5. [Servicios](#5-servicios-dentro-de-una-categoría)
6. [Idiomas y traducciones](#6-idiomas-y-traducciones)
7. [Bloquear scripts hasta el consentimiento](#7-bloquear-scripts-hasta-el-consentimiento)
8. [Auto-borrado de cookies](#8-auto-borrado-de-cookies)
9. [Callbacks (eventos)](#9-callbacks-eventos)
10. [Personalización de la UI](#10-personalización-de-la-ui)
11. [Opciones de cookie / localStorage](#11-opciones-de-cookie--localstorage)
12. [Modos: opt-in vs opt-out](#12-modos-opt-in-vs-opt-out)
13. [Revisiones de consentimiento](#13-revisiones-de-consentimiento)
14. [Abrir el modal desde un botón](#14-abrir-el-modal-desde-un-botón)
15. [API completa](#15-api-completa)
16. [Referencia de configuración](#16-referencia-de-configuración)
17. [Recetas frecuentes](#17-recetas-frecuentes)
18. [Integración por framework](#18-integración-por-framework)

---

## 1. Instalación

Desde GitHub directamente (sin npm):

```bash
pnpm add github:AntonioValls/cookieconsent
# o
npm i github:AntonioValls/cookieconsent
```

El paquete viene **ya compilado** (`dist/` se sube al repo), así que no necesitas hacer build al instalarlo.

Importación habitual:

```js
import '@antoniovalls/cookieconsent/dist/cookieconsent.css';
import * as CookieConsent from '@antoniovalls/cookieconsent';
```

> Nota: si tu bundler no resuelve `dist/cookieconsent.css`, importa el CSS por ruta absoluta del paquete o cópialo a tu carpeta de assets.

---

## 2. Configuración mínima

Las dos claves obligatorias son `categories` y `language`:

```js
CookieConsent.run({
    categories: {
        necessary: { enabled: true, readOnly: true },
        analytics: {}
    },
    language: {
        default: 'es',
        translations: {
            es: {
                consentModal: {
                    title: 'Usamos cookies',
                    description: 'Este sitio usa cookies para mejorar tu experiencia.',
                    acceptAllBtn: 'Aceptar todas',
                    acceptNecessaryBtn: 'Rechazar todas',
                    showPreferencesBtn: 'Preferencias'
                },
                preferencesModal: {
                    title: 'Preferencias de cookies',
                    acceptAllBtn: 'Aceptar todas',
                    acceptNecessaryBtn: 'Rechazar todas',
                    savePreferencesBtn: 'Guardar selección',
                    closeIconLabel: 'Cerrar',
                    sections: [
                        { title: 'Necesarias', linkedCategory: 'necessary' },
                        { title: 'Analíticas', linkedCategory: 'analytics' }
                    ]
                }
            }
        }
    }
});
```

---

## 3. Estructura del objeto `run()`

Todas las opciones de primer nivel:

```js
CookieConsent.run({
    root,                  // string | Element — dónde montar el modal (default: document.body)
    mode,                  // 'opt-in' | 'opt-out' (default: 'opt-in')
    autoShow,              // boolean — mostrar modal si no hay consentimiento (default: true)
    revision,              // number — versión del consentimiento (default: 0)
    manageScriptTags,      // boolean — interceptar <script type="text/plain"> (default: true)
    autoClearCookies,      // boolean — borrar cookies al rechazar categoría (default: true)
    disablePageInteraction,// boolean — overlay oscuro y bloquear scroll (default: false)
    hideFromBots,          // boolean — no renderizar para crawlers (default: true)
    lazyHtmlGeneration,    // boolean — generar HTML solo cuando se necesita (default: true)

    guiOptions: { /* ver §10 */ },
    cookie:     { /* ver §11 */ },
    categories: { /* ver §4 */ },
    language:   { /* ver §6 */ },

    onFirstConsent,        // callback — primera aceptación
    onConsent,             // callback — primera aceptación + cada carga de página
    onChange,              // callback — cambios posteriores
    onModalShow,           // callback — modal visible
    onModalHide,           // callback — modal oculto
    onModalReady           // callback — modal añadido al DOM
});
```

---

## 4. Categorías

Cada categoría se define como una clave dentro de `categories`:

```js
categories: {
    necessary: {
        enabled: true,   // habilitada por defecto
        readOnly: true   // el usuario no la puede deshabilitar
    },
    analytics: {
        autoClear: {
            cookies: [
                { name: /^_ga/ },     // regex: borra todas las _ga*
                { name: '_gid' }      // string: nombre exacto
            ],
            reloadPage: false         // recargar tras borrar (default: false)
        }
    },
    marketing: {}
}
```

Propiedades de una categoría:

| Propiedad   | Tipo                | Default | Descripción                                                       |
|-------------|---------------------|---------|-------------------------------------------------------------------|
| `enabled`   | boolean             | `false` | Habilitada por defecto (relevante en modo `opt-out`)              |
| `readOnly`  | boolean             | `false` | El usuario no puede desactivarla                                  |
| `services`  | object              | —       | Servicios individuales con su propio toggle (ver §5)              |
| `autoClear` | object              | —       | Cookies a borrar cuando se rechaza la categoría                   |

`autoClear.cookies` acepta:
- `name`: `string` o `RegExp`
- `path`: ruta esperada (opcional)
- `domain`: dominio esperado (opcional, útil para borrar cookies del dominio padre desde subdominios)

---

## 5. Servicios (dentro de una categoría)

Un *servicio* es un script o grupo de scripts con su propio toggle independiente dentro de una categoría:

```js
categories: {
    analytics: {
        services: {
            ga: {
                label: 'Google Analytics',
                onAccept: () => { /* cargar GA */ },
                onReject: () => { /* limpiar GA */ },
                cookies: [{ name: /^_ga/ }]
            },
            youtube: {
                label: 'YouTube embeds',
                onAccept: () => {},
                onReject: () => {}
            }
        }
    }
}
```

Cuando se define un servicio aparece automáticamente un toggle dentro del toggle de la categoría.

---

## 6. Idiomas y traducciones

```js
language: {
    default: 'es',
    autoDetect: 'browser',  // 'document' (lee <html lang>) | 'browser' (navigator.language)
    rtl: ['ar', 'he'],      // idiomas RTL (opcional)
    translations: {
        es: { consentModal: {...}, preferencesModal: {...} },
        en: { consentModal: {...}, preferencesModal: {...} },

        // También se puede cargar desde un archivo externo:
        fr: '/locales/cc-fr.json',

        // O dinámicamente:
        de: async () => (await fetch('/locales/cc-de.json')).json()
    }
}
```

### Estructura de una traducción

```js
{
    consentModal: {
        label,             // accesibilidad (si no hay título)
        title,
        description,
        acceptAllBtn,
        acceptNecessaryBtn,
        showPreferencesBtn,
        closeIconLabel,    // muestra una "X" en layout 'box'
        revisionMessage,   // visible cuando cambia el número de revisión
        footer             // HTML libre (enlaces a privacidad, etc.)
    },
    preferencesModal: {
        title,
        acceptAllBtn,
        acceptNecessaryBtn,
        savePreferencesBtn,
        closeIconLabel,
        serviceCounterLabel,  // ej: 'Servicio|Servicios' (singular|plural)
        sections: [
            {
                title,
                description,
                linkedCategory,  // si existe, esa sección será un toggle
                cookieTable: {
                    caption,
                    headers: { name: 'Cookie', domain: 'Dominio', desc: 'Descripción' },
                    body: [{ name: '_ga', domain: '...', desc: '...' }]
                }
            }
        ]
    }
}
```

---

## 7. Bloquear scripts hasta el consentimiento

### Vía atributos en `<script>`

Por defecto el plugin intercepta cualquier `<script type="text/plain">` con atributo `data-category`:

```html
<!-- Antes -->
<script src="https://www.googletagmanager.com/gtag/js?id=GA_ID"></script>
<script>
  window.dataLayer = window.dataLayer || [];
  function gtag(){dataLayer.push(arguments);}
  gtag('js', new Date());
  gtag('config', 'GA_ID');
</script>

<!-- Después -->
<script
    type="text/plain"
    data-category="analytics"
    data-src="https://www.googletagmanager.com/gtag/js?id=GA_ID"
></script>

<script
    type="text/plain"
    data-category="analytics"
    data-service="Google Analytics"
>
    window.dataLayer = window.dataLayer || [];
    function gtag(){dataLayer.push(arguments);}
    gtag('js', new Date());
    gtag('config', 'GA_ID');
</script>
```

Atributos disponibles:

| Atributo        | Obligatorio | Descripción                                                |
|-----------------|-------------|------------------------------------------------------------|
| `type`          | sí          | Debe ser `"text/plain"`                                    |
| `data-category` | sí          | Nombre de la categoría que activa este script              |
| `data-service`  | no          | Toggle individual dentro de la categoría                   |
| `data-type`     | no          | Tipo real al ejecutarse (ej. `"module"`)                   |
| `data-src`      | no          | Alternativa a `src` (evita validaciones del navegador)     |

### Ejecutar al **deshabilitar** una categoría/servicio

Anteponiendo `!`:

```html
<script type="text/plain" data-category="!analytics">
    // se ejecuta cuando 'analytics' se desactiva (si antes estaba activa)
</script>
```

### Vía callbacks (sin atributos)

```js
CookieConsent.run({
    onConsent: () => {
        if (CookieConsent.acceptedCategory('analytics')) {
            // cargar GA aquí
        }
    },
    onChange: ({ changedCategories, changedServices }) => {
        if (changedCategories.includes('analytics')) {
            if (CookieConsent.acceptedCategory('analytics')) {
                // se acaba de activar
            } else {
                // se acaba de desactivar
            }
        }
    }
});
```

---

## 8. Auto-borrado de cookies

Si `autoClearCookies: true` (default), al rechazar una categoría se borran automáticamente las cookies declaradas en `autoClear.cookies`:

```js
analytics: {
    autoClear: {
        cookies: [
            { name: /^_ga/ },
            { name: '_gid' },
            { name: 'session', path: '/', domain: '.midominio.com' }
        ],
        reloadPage: true   // recargar la página tras borrar
    }
}
```

Para borrar cookies manualmente:

```js
CookieConsent.eraseCookies(['_ga', '_gid'], '/', 'midominio.com');
CookieConsent.eraseCookies(/^_ga/);
```

---

## 9. Callbacks (eventos)

```js
CookieConsent.run({
    onFirstConsent: ({ cookie }) => {
        // primera vez que el usuario acepta/rechaza
    },
    onConsent: ({ cookie }) => {
        // primera vez + cada carga de página con consentimiento válido
    },
    onChange: ({ cookie, changedCategories, changedServices }) => {
        // cambio posterior de preferencias
    },
    onModalShow:  ({ modalName }) => { /* 'consentModal' | 'preferencesModal' */ },
    onModalHide:  ({ modalName }) => {},
    onModalReady: ({ modalName, modal }) => { /* modal: HTMLElement */ }
});
```

### Equivalente con `addEventListener`

```js
window.addEventListener('cc:onConsent',     ({ detail }) => { /* detail.cookie */ });
window.addEventListener('cc:onFirstConsent',({ detail }) => {});
window.addEventListener('cc:onChange',      ({ detail }) => {});
window.addEventListener('cc:onModalShow',   ({ detail }) => {});
window.addEventListener('cc:onModalHide',   ({ detail }) => {});
window.addEventListener('cc:onModalReady',  ({ detail }) => {});
```

---

## 10. Personalización de la UI

```js
guiOptions: {
    consentModal: {
        layout: 'box',                  // 'box' | 'box wide' | 'box inline' | 'cloud' | 'cloud inline' | 'bar' | 'bar inline'
        position: 'bottom right',       // 'top' | 'middle' | 'bottom' [+ 'left'|'center'|'right']
        equalWeightButtons: true,       // mismo peso visual aceptar/rechazar (recomendado por GDPR)
        flipButtons: false              // intercambiar orden de botones
    },
    preferencesModal: {
        layout: 'box',                  // 'box' | 'bar' | 'bar wide'
        position: 'right',              // solo válido con layout 'bar': 'left' | 'right'
        equalWeightButtons: true,
        flipButtons: false
    }
}
```

### Modo oscuro

Añade la clase `cc--darkmode` al `<html>` o `<body>`:

```html
<html class="cc--darkmode">
```

O dinámicamente:

```js
document.documentElement.classList.toggle('cc--darkmode', userPrefersDark);
```

### CSS variables principales

Puedes sobreescribirlas en tu propio CSS:

```css
:root {
    --cc-bg: #fff;
    --cc-primary-color: #2c2f31;
    --cc-secondary-color: #5e6266;
    --cc-btn-primary-bg: #2c2f31;
    --cc-btn-primary-color: #fff;
    --cc-btn-primary-hover-bg: #23272a;
    --cc-btn-secondary-bg: #eaecf0;
    --cc-btn-secondary-color: #2c2f31;
    --cc-toggle-bg-on: #2c2f31;
    --cc-toggle-bg-off: #8b9197;
    --cc-toggle-knob-bg: #fff;
    --cc-overlay-bg: rgba(0, 0, 0, 0.65);
    --cc-modal-border-radius: 0.5rem;
    --cc-btn-border-radius: 0.4rem;
}
```

> Para la lista exacta y actualizada de variables, mira `src/scss/abstracts/_light-color-scheme.scss` y `_dark-color-scheme.scss` en este repo.

---

## 11. Opciones de cookie / localStorage

```js
cookie: {
    name: 'cc_cookie',           // nombre (default: 'cc_cookie')
    domain: location.hostname,   // dominio
    path: '/',                   // path
    sameSite: 'Lax',             // 'Lax' | 'Strict' | 'None'
    secure: true,                // flag Secure
    expiresAfterDays: 182,       // duración o (acceptType) => number
    useLocalStorage: false       // si true, usa localStorage en vez de cookie
}
```

`expiresAfterDays` puede ser una función para variar la duración según la decisión:

```js
expiresAfterDays: (acceptType) => acceptType === 'all' ? 365 : 90
```

---

## 12. Modos: opt-in vs opt-out

| Modo       | Comportamiento                                                                       |
|------------|--------------------------------------------------------------------------------------|
| `opt-in`   | (default) Hasta que el usuario acepte, **nada** se ejecuta excepto `necessary`.      |
| `opt-out`  | Las categorías con `enabled: true` se ejecutan **antes** del consentimiento. El usuario puede después desactivarlas. |

Para la UE/GDPR siempre debes usar `opt-in`.

---

## 13. Revisiones de consentimiento

Cuando cambias tu política de cookies, incrementa `revision` y todos los usuarios verán el modal de nuevo:

```js
CookieConsent.run({
    revision: 1,   // incrementa cada vez que cambies tu política
    language: {
        default: 'es',
        translations: {
            es: {
                consentModal: {
                    revisionMessage: '<br><br>Hemos actualizado nuestra política de cookies.',
                    // ...
                }
            }
        }
    }
});
```

---

## 14. Abrir el modal desde un botón

Vía atributo `data-cc` (forma recomendada):

```html
<button type="button" data-cc="show-preferencesModal">Gestionar cookies</button>
<button type="button" data-cc="show-consentModal">Mostrar banner</button>
<button type="button" data-cc="accept-all">Aceptar todas</button>
<button type="button" data-cc="accept-necessary">Solo necesarias</button>
<button type="button" data-cc="accept-custom">Aceptar selección actual</button>
```

Vía API:

```js
CookieConsent.showPreferences();
CookieConsent.show();          // muestra el banner; pasar `true` lo regenera si no existe
```

---

## 15. API completa

Todas las funciones disponibles en el namespace `CookieConsent`:

```ts
// Configurar y arrancar
run(config): Promise<void>

// Modales
show(createModal?: boolean): void
hide(): void
showPreferences(): void
hidePreferences(): void

// Aceptar/rechazar programáticamente
acceptCategory(categories: string | string[], excludedCategories?: string[]): void
acceptService(services: string | string[], category: string): void

// Consultar estado
acceptedCategory(name: string): boolean
acceptedService(serviceName: string, categoryName: string): boolean
validConsent(): boolean
validCookie(cookieName: string): boolean
getUserPreferences(): {
    acceptType: 'all' | 'custom' | 'necessary',
    acceptedCategories: string[],
    rejectedCategories: string[],
    acceptedServices: { [cat: string]: string[] },
    rejectedServices: { [cat: string]: string[] }
}

// Cookie del propio plugin
getCookie(): CookieValue
getCookie(field: keyof CookieValue): any
setCookieData({ value, mode: 'overwrite' | 'update' }): boolean
eraseCookies(cookies: string | RegExp | (string|RegExp)[], path?, domain?): void

// Configuración
getConfig(): Config
getConfig(field: keyof Config): any

// Cargar scripts
loadScript(src: string, attributes?: { [k: string]: string }): Promise<boolean>

// Idiomas
setLanguage(code: string, forceSet?: boolean): Promise<boolean>

// Reset total
reset(eraseCookie?: boolean): void
```

### Forma del cookie del plugin

```ts
{
    categories: string[],            // categorías aceptadas
    revision: number,
    data: any,                       // datos custom (vía setCookieData)
    consentId: string,               // UUID del usuario
    consentTimestamp: string,        // primera aceptación (ISO)
    lastConsentTimestamp: string,    // última actualización (ISO)
    languageCode: string,
    services: { [category: string]: string[] },
    expirationTime: number
}
```

---

## 16. Referencia de configuración

Tabla resumen de todas las opciones de primer nivel:

| Opción                    | Tipo                          | Default            |
|---------------------------|-------------------------------|--------------------|
| `root`                    | `string \| Element \| null`   | `document.body`    |
| `mode`                    | `'opt-in' \| 'opt-out'`       | `'opt-in'`         |
| `autoShow`                | `boolean`                     | `true`             |
| `revision`                | `number`                      | `0`                |
| `manageScriptTags`        | `boolean`                     | `true`             |
| `autoClearCookies`        | `boolean`                     | `true`             |
| `disablePageInteraction`  | `boolean`                     | `false`            |
| `hideFromBots`            | `boolean`                     | `true`             |
| `lazyHtmlGeneration`      | `boolean`                     | `true`             |
| `guiOptions`              | `GuiOptions`                  | —                  |
| `cookie`                  | `CookieOptions`               | ver §11            |
| `categories`              | `{[k]: Category}` (requerido) | —                  |
| `language`                | `Language` (requerido)        | —                  |
| `onFirstConsent`          | `(p) => void`                 | —                  |
| `onConsent`               | `(p) => void`                 | —                  |
| `onChange`                | `(p) => void`                 | —                  |
| `onModalShow`             | `(p) => void`                 | —                  |
| `onModalHide`             | `(p) => void`                 | —                  |
| `onModalReady`            | `(p) => void`                 | —                  |

---

## 17. Recetas frecuentes

### Google Analytics 4

```js
CookieConsent.run({
    categories: {
        necessary: { enabled: true, readOnly: true },
        analytics: {
            autoClear: { cookies: [{ name: /^_ga/ }, { name: '_gid' }] },
            services: {
                ga: {
                    label: 'Google Analytics',
                    onAccept: async () => {
                        await CookieConsent.loadScript(
                            'https://www.googletagmanager.com/gtag/js?id=G-XXXXXXX'
                        );
                        window.dataLayer = window.dataLayer || [];
                        function gtag(){ dataLayer.push(arguments); }
                        gtag('js', new Date());
                        gtag('config', 'G-XXXXXXX');
                    }
                }
            }
        }
    },
    language: { /* ... */ }
});
```

### Reabrir el banner cuando el usuario haga clic en un enlace de footer

```html
<a href="#" data-cc="show-preferencesModal">Configurar cookies</a>
```

### Detectar idioma del navegador

```js
language: {
    default: 'es',
    autoDetect: 'browser',
    translations: {
        es: { /* ... */ },
        en: { /* ... */ },
        ca: { /* ... */ }
    }
}
```

### Cambiar idioma en runtime

```js
await CookieConsent.setLanguage('en');
```

### Logging del consentimiento al servidor

```js
onFirstConsent: ({ cookie }) => {
    fetch('/api/consent-log', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({
            consentId: cookie.consentId,
            timestamp: cookie.consentTimestamp,
            categories: cookie.categories,
            services: cookie.services
        })
    });
}
```

### Almacenar datos custom dentro del cookie

```js
CookieConsent.setCookieData({
    value: { plan: 'pro', signupDate: '2026-04-28' },
    mode: 'update'   // 'overwrite' reemplaza, 'update' fusiona
});
```

### Resetear todo (debug)

```js
CookieConsent.reset(true);   // borra el cookie también
location.reload();
```

---

## 18. Integración por framework

### Astro

`src/components/CookieConsent.astro`:

```astro
---
// componente sin props
---
<script>
    import '@antoniovalls/cookieconsent/dist/cookieconsent.css';
    import * as CookieConsent from '@antoniovalls/cookieconsent';

    CookieConsent.run({
        categories: { necessary: { enabled: true, readOnly: true }, analytics: {} },
        language: { default: 'es', translations: { es: { /* ... */ } } }
    });
</script>
```

Inclúyelo en tu `Layout.astro` antes de `</body>`.

### Next.js (App Router)

`app/components/CookieConsent.tsx`:

```tsx
'use client';
import { useEffect } from 'react';
import '@antoniovalls/cookieconsent/dist/cookieconsent.css';
import * as CookieConsent from '@antoniovalls/cookieconsent';

export default function CookieConsentBanner() {
    useEffect(() => {
        CookieConsent.run({
            categories: { necessary: { enabled: true, readOnly: true }, analytics: {} },
            language: { default: 'es', translations: { es: { /* ... */ } } }
        });
    }, []);
    return null;
}
```

Y en tu `app/layout.tsx`:

```tsx
import CookieConsentBanner from './components/CookieConsent';

export default function RootLayout({ children }) {
    return (
        <html lang="es">
            <body>
                {children}
                <CookieConsentBanner />
            </body>
        </html>
    );
}
```

### Vue 3

`plugins/CookieConsentVue.js`:

```js
import '@antoniovalls/cookieconsent/dist/cookieconsent.css';
import * as CookieConsent from '@antoniovalls/cookieconsent';

export default {
    install: (app, pluginConfig) => {
        app.config.globalProperties.$CookieConsent = CookieConsent;
        CookieConsent.run(pluginConfig);
    }
};
```

`main.js`:

```js
import { createApp } from 'vue';
import App from './App.vue';
import CookieConsentVue from './plugins/CookieConsentVue';

createApp(App)
    .use(CookieConsentVue, {
        categories: { necessary: { enabled: true, readOnly: true }, analytics: {} },
        language: { default: 'es', translations: { es: { /* ... */ } } }
    })
    .mount('#app');
```

### HTML estático

```html
<!DOCTYPE html>
<html lang="es">
<head>
    <link rel="stylesheet" href="/path/to/cookieconsent.css">
</head>
<body>
    <!-- contenido -->

    <script src="/path/to/cookieconsent.umd.js"></script>
    <script>
        CookieConsent.run({
            categories: { necessary: { enabled: true, readOnly: true }, analytics: {} },
            language: { default: 'es', translations: { es: { /* ... */ } } }
        });
    </script>
</body>
</html>
```

---

## Notas finales

- Si actualizas tu política de cookies, **incrementa `revision`** para forzar reconsentimiento.
- En la UE usa siempre `mode: 'opt-in'` y nunca cargues scripts de tracking antes de `acceptedCategory()`.
- Para auditoría, guarda el `consentId` y `consentTimestamp` del cookie tras `onFirstConsent`.
- Si quieres cambiar estilos profundamente, edita los SCSS en `src/scss/` y vuelve a hacer `pnpm build`.

Crédito original: [vanilla-cookieconsent](https://github.com/orestbida/cookieconsent) por Orest Bida (MIT).
