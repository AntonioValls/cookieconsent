# @antoniovalls/cookieconsent

Fork personal de [vanilla-cookieconsent](https://github.com/orestbida/cookieconsent) (v3.1.0) con personalizaciones para mis proyectos.

Plugin ligero y compatible con GDPR para gestionar el consentimiento de cookies, escrito en JavaScript vanilla.

## Instalación

Desde GitHub directamente:

```bash
pnpm add github:AntonioValls/cookieconsent
```

## Uso básico

```js
import 'vanilla-cookieconsent/dist/cookieconsent.css';
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

## Documentación

La documentación completa de la API base está en el proyecto original: [cookieconsent.orestbida.com](https://cookieconsent.orestbida.com).

## Desarrollo

```bash
pnpm install
pnpm build      # genera dist/
pnpm dev        # rebuild en watch
pnpm test       # jest
```

## Créditos

Basado en [vanilla-cookieconsent](https://github.com/orestbida/cookieconsent) por Orest Bida (MIT).

## Licencia

MIT — ver [LICENSE](./LICENSE).
