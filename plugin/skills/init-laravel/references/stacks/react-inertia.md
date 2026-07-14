# React + Inertia Stack

Configuration for a Laravel monolith with React, Inertia.js, and TypeScript.

## Dependencies

```bash
composer require inertiajs/inertia-laravel
npm install @inertiajs/react react react-dom @vitejs/plugin-react
npm install -D typescript @types/react @types/react-dom tailwindcss @tailwindcss/vite prettier eslint
```

## `resources/` structure

```
resources/
├── css/
│   └── app.css                    # @tailwind base/components/utilities
├── js/
│   ├── app.tsx                    # Entry point (createInertiaApp + createRoot)
│   ├── bootstrap.ts               # Axios config
│   ├── pages/                     # Inertia pages (mirror of the routes)
│   │   ├── Dashboard.tsx
│   │   ├── Profile/
│   │   │   ├── Edit.tsx
│   │   │   └── Show.tsx
│   │   └── Auth/
│   │       ├── Login.tsx
│   │       └── Register.tsx
│   ├── components/                # Reusable components
│   │   ├── common/                # Generic (Button, Input, Modal, etc.)
│   │   │   ├── Button.tsx
│   │   │   ├── Input.tsx
│   │   │   └── Modal.tsx
│   │   └── [feature]/             # Specific to a business domain
│   ├── layouts/                   # Shared layouts
│   │   ├── AppLayout.tsx          # Authenticated layout
│   │   └── GuestLayout.tsx        # Guest layout
│   ├── hooks/                     # Custom React hooks
│   │   ├── useAuth.ts
│   │   ├── useForm.ts
│   │   └── useFetch.ts
│   ├── types/                     # TypeScript types
│   │   ├── index.ts               # Central export of the types
│   │   └── models.ts             # Types mirroring the Eloquent models
│   ├── lib/                       # Utilities and helpers
│   │   ├── constants.ts           # Application constants
│   │   ├── helpers.ts             # Utility functions
│   │   └── api.ts                 # API call helpers
│   └── contexts/                  # React contexts (shared state)
│       └── AppContext.tsx
└── views/
    └── app.blade.php              # Blade root template
```

### React naming conventions

- **camelCase** for folders: `pages/`, `components/`, `layouts/`, `hooks/`
- **PascalCase** for component files: `Dashboard.tsx`, `AppLayout.tsx`
- **camelCase** for hooks: `hooks/useForm.ts`
- **camelCase** for utilities: `lib/helpers.ts`
- Pages mirror the route structure: `/dashboard` → `pages/Dashboard.tsx`
- One component per file, named like the default export

## Configuration files

### `vite.config.ts`

```typescript
import { defineConfig } from 'vite';
import laravel from 'laravel-vite-plugin';
import react from '@vitejs/plugin-react';
import tailwindcss from '@tailwindcss/vite';
import path from 'path';

export default defineConfig({
    plugins: [
        laravel(['resources/js/app.tsx']),
        react(),
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
        "jsx": "react-jsx",
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
        "resources/js/**/*.tsx",
        "resources/js/**/*.d.ts"
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
        './resources/js/**/*.tsx',
        './resources/js/**/*.jsx',
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

## Entry Point (`resources/js/app.tsx`)

```typescript
import '../css/app.css';
import './bootstrap';

import { createRoot } from 'react-dom/client';
import { createInertiaApp } from '@inertiajs/react';
import { resolvePageComponent } from 'laravel-vite-plugin/inertia-helpers';

createInertiaApp({
    title: (title) => `${title} - ${import.meta.env.VITE_APP_NAME || 'Laravel'}`,
    resolve: (name) =>
        resolvePageComponent(
            `./pages/${name}.tsx`,
            import.meta.glob<any>('./pages/**/*.tsx'),
        ),
    setup({ el, App, props }) {
        const root = createRoot(el);
        root.render(<App {...props} />);
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
    @vite('resources/js/app.tsx')
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
        "type-check": "tsc --noEmit",
        "lint": "eslint . --ext .tsx,.ts --fix",
        "format": "prettier --write resources/js/"
    }
}
```

## Useful types (`resources/js/types/index.ts`)

```typescript
import type { Page } from '@inertiajs/core';

export interface User {
    id: number;
    name: string;
    email: string;
}

export interface PageProps {
    auth: {
        user: User;
    };
    flash: {
        success?: string;
        error?: string;
    };
}

export type InertiaPage<T = {}> = Page<PageProps & T>;
```

## Example layout (`resources/js/layouts/AppLayout.tsx`)

```tsx
import { PropsWithChildren } from 'react';
import { Head } from '@inertiajs/react';

export default function AppLayout({ children, title }: PropsWithChildren<{ title?: string }>) {
    return (
        <>
            <Head title={title} />
            <div className="min-h-screen bg-gray-100">
                <nav>{/* Navigation */}</nav>
                <main>{children}</main>
            </div>
        </>
    );
}
```
