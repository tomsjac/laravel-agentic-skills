# Vue.js + Inertia Stack

Configuration for a Laravel monolith with Vue 3, Inertia.js, and TypeScript.

## Dependencies

```bash
composer require inertiajs/inertia-laravel
npm install @inertiajs/vue3 vue @vitejs/plugin-vue
npm install -D typescript vue-tsc tailwindcss @tailwindcss/vite prettier eslint
```

## `resources/` structure

```
resources/
├── css/
│   └── app.css                    # @tailwind base/components/utilities
├── js/
│   ├── app.ts                     # Entry point (createInertiaApp)
│   ├── bootstrap.ts               # Axios config
│   ├── Pages/                     # Inertia pages (mirror of the routes)
│   │   ├── Dashboard.vue
│   │   ├── Profile/
│   │   │   ├── Edit.vue
│   │   │   └── Show.vue
│   │   └── Auth/
│   │       ├── Login.vue
│   │       └── Register.vue
│   ├── Components/                # Reusable components
│   │   ├── Common/                # Generic (Button, Input, Modal, etc.)
│   │   │   ├── Button.vue
│   │   │   ├── Input.vue
│   │   │   └── Modal.vue
│   │   └── [Feature]/             # Specific to a business domain
│   ├── Layouts/                   # Shared layouts
│   │   ├── AppLayout.vue          # Authenticated layout
│   │   └── GuestLayout.vue        # Guest layout
│   ├── composables/               # Vue composables (equivalent of React hooks)
│   │   ├── useAuth.ts
│   │   ├── useForm.ts
│   │   └── useFetch.ts
│   ├── types/                     # TypeScript types
│   │   ├── global.d.ts            # Global declarations (Inertia, Vite)
│   │   ├── index.ts               # Central export of the types
│   │   └── models/                # Types mirroring the Eloquent models
│   │       ├── User.ts
│   │       └── index.ts
│   ├── lib/                       # Utilities and helpers
│   │   ├── constants.ts           # Application constants
│   │   ├── helpers.ts             # Utility functions
│   │   └── api.ts                 # API call helpers
│   └── stores/                    # Pinia stores (if state management is needed)
│       └── useAppStore.ts
└── views/
    └── app.blade.php              # Blade root template
```

### Vue naming conventions

- **PascalCase** for component folders: `Pages/`, `Components/`, `Layouts/`
- **PascalCase** for `.vue` files: `Dashboard.vue`, `AppLayout.vue`
- **camelCase** for composables: `composables/useForm.ts`
- **camelCase** for utilities: `lib/helpers.ts`
- Pages mirror the route structure: `/dashboard` → `Pages/Dashboard.vue`

## Configuration files

### `vite.config.ts`

```typescript
import { defineConfig } from 'vite';
import laravel from 'laravel-vite-plugin';
import vue from '@vitejs/plugin-vue';
import tailwindcss from '@tailwindcss/vite';
import path from 'path';

export default defineConfig({
    plugins: [
        laravel(['resources/js/app.ts']),
        vue({
            template: {
                transformAssetUrls: {
                    base: null,
                    includeAbsolute: false,
                },
            },
        }),
        tailwindcss(),
    ],
    resolve: {
        alias: {
            '@': path.resolve(__dirname, './resources/js'),
        },
    },
});
```

### `tsconfig.json`

```json
{
    "compilerOptions": {
        "target": "ES2020",
        "module": "ESNext",
        "moduleResolution": "bundler",
        "lib": ["ES2020", "DOM", "DOM.Iterable"],
        "jsx": "preserve",
        "strict": true,
        "isolatedModules": true,
        "noEmit": true,
        "esModuleInterop": true,
        "skipLibCheck": true,
        "forceConsistentCasingInFileNames": true,
        "allowJs": true,
        "resolveJsonModule": true,
        "types": ["vite/client", "node"],
        "baseUrl": ".",
        "paths": {
            "@/*": ["resources/js/*"]
        }
    },
    "include": [
        "resources/js/**/*.ts",
        "resources/js/**/*.d.ts",
        "resources/js/**/*.vue"
    ],
    "exclude": ["node_modules"]
}
```

### `tailwind.config.js`

```javascript
import defaultTheme from 'tailwindcss/defaultTheme';

export default {
    content: [
        './vendor/laravel/framework/src/Illuminate/Pagination/resources/views/*.blade.php',
        './storage/framework/views/*.php',
        './resources/views/**/*.blade.php',
        './resources/js/**/*.vue',
    ],
    theme: {
        extend: {
            fontFamily: {
                sans: ['Figtree', ...defaultTheme.fontFamily.sans],
            },
        },
    },
    plugins: [],
};
```

## Entry Point (`resources/js/app.ts`)

```typescript
import '../css/app.css';
import './bootstrap';

import { createApp, h } from 'vue';
import { createInertiaApp } from '@inertiajs/vue3';
import { resolvePageComponent } from 'laravel-vite-plugin/inertia-helpers';

createInertiaApp({
    title: (title) => `${title} - ${import.meta.env.VITE_APP_NAME || 'Laravel'}`,
    resolve: (name) =>
        resolvePageComponent(
            `./Pages/${name}.vue`,
            import.meta.glob<any>('./Pages/**/*.vue'),
        ),
    setup({ el, App, props, plugin }) {
        createApp({ render: () => h(App, props) })
            .use(plugin)
            .mount(el);
    },
    progress: {
        color: '#4B5563',
    },
});
```

## Root Template (`resources/views/app.blade.php`)

```blade
<!DOCTYPE html>
<html lang="{{ str_replace('_', '-', app()->getLocale()) }}">
<head>
    <meta charset="utf-8">
    <meta name="viewport" content="width=device-width, initial-scale=1">
    <meta name="csrf-token" content="{{ csrf_token() }}">
    <title>{{ config('app.name') }}</title>
    @vite('resources/js/app.ts')
    @inertiaHead
</head>
<body>
    @inertia
</body>
</html>
```

## npm scripts

```json
{
    "scripts": {
        "dev": "vite",
        "build": "vite build",
        "preview": "vite preview",
        "type-check": "vue-tsc --noEmit",
        "lint": "eslint . --ext .vue,.ts --fix",
        "format": "prettier --write resources/js/"
    }
}
```

## Useful types (`resources/js/types/global.d.ts`)

```typescript
import type { Page } from '@inertiajs/core';

declare module 'vue' {
    interface ComponentCustomProperties {
        $page: Page;
    }
}

declare module '@inertiajs/vue3' {
    interface PageProps {
        auth: {
            user: {
                id: number;
                name: string;
                email: string;
            };
        };
        flash: {
            success?: string;
            error?: string;
        };
    }
}
```
