# @antoniovalls/cookieconsent

Fork personal de [vanilla-cookieconsent](https://github.com/orestbida/cookieconsent) (v3.1.0) con personalizaciones para mis proyectos.

Plugin ligero y compatible con GDPR para gestionar el consentimiento de cookies, escrito en JavaScript vanilla.

## Documentación

- 📘 [**USAGE.md**](./USAGE.md) — Manual completo: API, configuración, callbacks, traducciones, recetas frecuentes.
- 🚀 [**ASTRO.md**](./ASTRO.md) — Guía específica para integrarlo en proyectos Astro 5/6 (View Transitions, i18n, GA4, etc.).

## Instalación

```bash
pnpm add github:AntonioValls/cookieconsent
```

## Uso básico

```js
import '@antoniovalls/cookieconsent/dist/cookieconsent.css';
import * as CookieConsent from '@antoniovalls/cookieconsent';

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
                    description: 'Este sitio utiliza cookies para mejorar la experiencia.',
                    acceptAllBtn: 'Aceptar todas',
                    acceptNecessaryBtn: 'Rechazar todas',
                    showPreferencesBtn: 'Preferencias'
                },
                preferencesModal: {
                    title: 'Preferencias de cookies',
                    acceptAllBtn: 'Aceptar todas',
                    acceptNecessaryBtn: 'Rechazar todas',
                    savePreferencesBtn: 'Guardar preferencias',
                    closeIconLabel: 'Cerrar',
                    sections: [
                        { title: 'Cookies estrictamente necesarias', linkedCategory: 'necessary' },
                        { title: 'Cookies analíticas', linkedCategory: 'analytics' }
                    ]
                }
            }
        }
    }
});
```

Para integración en Astro, ver [ASTRO.md](./ASTRO.md).

## Desarrollo

```bash
pnpm install
pnpm build      # genera dist/ (UMD + ESM + CSS)
pnpm dev        # rebuild en watch
```

## Créditos

Basado en [vanilla-cookieconsent](https://github.com/orestbida/cookieconsent) por Orest Bida (MIT).

## Licencia

MIT — ver [LICENSE](./LICENSE).
