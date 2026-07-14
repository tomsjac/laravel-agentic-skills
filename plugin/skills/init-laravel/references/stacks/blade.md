# Blade Stack

Configuration for a Laravel monolith with Blade templates.

## Dependencies

```bash
npm install -D tailwindcss @tailwindcss/vite
```

Offer the optional addition of:
- Livewire: `composer require livewire/livewire`
- Alpine.js (included by default with Livewire)

## `resources/` structure

```
resources/
├── css/
│   └── app.css                    # @tailwind base/components/utilities
├── js/
│   ├── app.js                     # Entry point
│   └── bootstrap.js               # Axios config
└── views/
    ├── layouts/
    │   ├── app.blade.php          # Main authenticated layout
    │   └── guest.blade.php        # Guest layout
    ├── components/                # Blade components
    │   ├── button.blade.php
    │   ├── input.blade.php
    │   └── modal.blade.php
    ├── pages/                     # Application pages
    │   ├── dashboard.blade.php
    │   └── profile/
    │       ├── edit.blade.php
    │       └── show.blade.php
    └── auth/                      # Authentication pages
        ├── login.blade.php
        └── register.blade.php
```

## Configuration files

### `vite.config.js`

```javascript
import { defineConfig } from 'vite';
import laravel from 'laravel-vite-plugin';
import tailwindcss from '@tailwindcss/vite';

export default defineConfig({
    plugins: [
        laravel(['resources/css/app.css', 'resources/js/app.js']),
        tailwindcss(),
    ],
});
```

### `tailwind.config.js`

```javascript
import defaultTheme from 'tailwindcss/defaultTheme';

export default {
    content: [
        './vendor/laravel/framework/src/Illuminate/Pagination/resources/views/*.blade.php',
        './storage/framework/views/*.php',
        './resources/views/**/*.blade.php',
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

## npm scripts

```json
{
    "scripts": {
        "dev": "vite",
        "build": "vite build"
    }
}
```

## Main layout (`resources/views/layouts/app.blade.php`)

```blade
<!DOCTYPE html>
<html lang="{{ str_replace('_', '-', app()->getLocale()) }}">
<head>
    <meta charset="utf-8">
    <meta name="viewport" content="width=device-width, initial-scale=1">
    <meta name="csrf-token" content="{{ csrf_token() }}">
    <title>{{ config('app.name') }} - @yield('title')</title>
    @vite(['resources/css/app.css', 'resources/js/app.js'])
</head>
<body>
    <main>
        @yield('content')
    </main>
</body>
</html>
```
