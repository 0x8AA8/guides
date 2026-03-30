# Fixing Mixed Content Errors: Laragon + Cloudflare Tunnel

> **Why this happens:** Cloudflare Tunnel terminates HTTPS at the edge and forwards traffic to your Laragon app over plain HTTP. Your app never sees an HTTPS request directly, so it generates all URLs with `http://`. The browser then blocks HTTP resources loaded inside an HTTPS page as mixed content.

---

## Table of Contents

- [Fix 1 — Laravel: Trust Proxies + Force HTTPS](#fix-1--laravel-trust-proxies--force-https)
- [Fix 2 — Cloudflare: Automatic HTTPS Rewrites](#fix-2--cloudflare-automatic-https-rewrites)
- [Fix 3 — Protocol-Relative URLs](#fix-3--protocol-relative-urls)
- [Non-Laravel Apps](#non-laravel-apps)
- [Which Fixes to Apply Together](#which-fixes-to-apply-together)
- [Quick Checklist](#quick-checklist)

---

## Fix 1 — Laravel: Trust Proxies + Force HTTPS

This is the **permanent, root-cause fix**. Without it, Laravel never knows the request arrived via HTTPS because Cloudflare decrypts HTTPS and forwards plain HTTP to your Laragon server.

### Step 1A — Configure the Cloudflare Tunnel Origin

Before touching code, make sure your tunnel points correctly to Laragon. In your **Cloudflare Zero Trust Dashboard → Networks → Tunnels → Configure → Public Hostnames**:

| Field        | Value                              |
|--------------|------------------------------------|
| Domain       | `yourdomain.com`                   |
| Service Type | `HTTP`                             |
| URL          | `http://localhost:80`              |

> Do **NOT** set Service Type to HTTPS — Laragon has no local cert. Cloudflare handles HTTPS at the edge only.

---

### Step 1B — Update `.env`

```env
APP_URL=https://your-tunnel-domain.com
APP_ENV=production
```

> If you keep `APP_ENV=local`, adjust the `boot()` condition in Step 1D accordingly (e.g. remove the environment check entirely).

---

### Step 1C — Configure TrustProxies

#### Laravel 11+ (`bootstrap/app.php`)

Laravel 11 removed the `TrustProxies.php` middleware file. Configure it directly inside the `withMiddleware` block:

```php
use Illuminate\Http\Request;

->withMiddleware(function (Middleware $middleware) {
    $middleware->trustProxies(
        at: '*',
        headers: Request::HEADER_X_FORWARDED_FOR
               | Request::HEADER_X_FORWARDED_HOST
               | Request::HEADER_X_FORWARDED_PORT
               | Request::HEADER_X_FORWARDED_PROTO
    );
})
```

#### Laravel 10 and below (`app/Http/Middleware/TrustProxies.php`)

Open the existing file and update these two properties:

```php
<?php

namespace App\Http\Middleware;

use Illuminate\Http\Middleware\TrustProxies as Middleware;
use Illuminate\Http\Request;

class TrustProxies extends Middleware
{
    /**
     * '*' trusts all IPs including dynamic Cloudflare edge IPs.
     * You can also list specific Cloudflare IP ranges for tighter security.
     */
    protected $proxies = '*';

    protected $headers =
        Request::HEADER_X_FORWARDED_FOR    |
        Request::HEADER_X_FORWARDED_HOST   |
        Request::HEADER_X_FORWARDED_PORT   |
        Request::HEADER_X_FORWARDED_PROTO  |
        Request::HEADER_X_FORWARDED_AWS_ELB;
}
```

> `HEADER_X_FORWARDED_PROTO` is the critical header — it tells Laravel the original connection used `https`, making `request()->isSecure()` return `true`.

Make sure `TrustProxies` is registered in `app/Http/Kernel.php` under `$middleware` (it should be by default in fresh installs):

```php
protected $middleware = [
    \App\Http\Middleware\TrustProxies::class,
    // ...
];
```

---

### Step 1D — Force HTTPS in AppServiceProvider

Open `app/Providers/AppServiceProvider.php` and add URL forcing in the `boot()` method:

```php
<?php

namespace App\Providers;

use Illuminate\Support\Facades\URL;
use Illuminate\Support\ServiceProvider;

class AppServiceProvider extends ServiceProvider
{
    public function boot(): void
    {
        // Adjust condition to match your APP_ENV
        if (app()->environment('production')) {
            URL::forceScheme('https');
        }

        // OR force unconditionally during local dev with tunnel:
        // URL::forceScheme('https');
    }
}
```

This forces **all** URL helpers — `url()`, `route()`, `asset()`, `action()` — to generate `https://` links.

---

### Step 1E — Clear All Caches

Stale cached config or routes will still serve old `http://` URLs until cleared:

```bash
php artisan config:clear
php artisan route:clear
php artisan view:clear
php artisan cache:clear
```

---

### Step 1F — Verify the Fix

```bash
php artisan tinker
>>> url('/any-path')
# Expected: https://your-tunnel-domain.com/any-path ✅

>>> request()->isSecure()
# Expected: true ✅ (run inside an actual request context)
```

---

## Fix 2 — Cloudflare: Automatic HTTPS Rewrites

This is an **edge-level fix**. Cloudflare scans your page's HTML before delivering it to the browser and rewrites any `http://` resource references to `https://` on the fly. It acts as a safety net even after Fix 1 is applied.

### Step 2A — Set SSL Mode to "Full"

Go to **Cloudflare Dashboard → your domain → SSL/TLS → Overview**:

| SSL Mode       | Description                                      | Use?                         |
|----------------|--------------------------------------------------|------------------------------|
| Off            | No HTTPS at all                                  | ❌ Never                     |
| Flexible       | HTTPS to browser, HTTP to origin                 | ⚠️ Causes redirect loops    |
| **Full**       | HTTPS to browser, HTTP to origin (no cert check) | ✅ Correct for Laragon        |
| Full (Strict)  | HTTPS both ways, valid origin cert required      | ❌ Laragon has no local cert  |

Select **Full** — Laragon serves plain HTTP locally and has no SSL certificate.

---

### Step 2B — Enable Automatic HTTPS Rewrites

Go to **Cloudflare Dashboard → your domain → SSL/TLS → Edge Certificates**:

1. Scroll down to **Automatic HTTPS Rewrites**
2. Toggle it **ON**
3. Changes apply immediately — no save button needed

This rewrites `http://` to `https://` for resource URLs in your HTML responses, including:

- `<script src="...">`
- `<link href="...">`
- `<img src="...">`
- `<form action="...">`
- `<iframe src="...">`
- `<audio src="...">` and `<video src="...">`

---

### Step 2C — Enable "Always Use HTTPS"

Still on the **Edge Certificates** page:

1. Find **Always Use HTTPS**
2. Toggle it **ON**

This ensures any visitor hitting your domain via `http://` gets a `301` redirect to `https://` before it ever reaches your app.

---

### Step 2D — (Optional) Add a Cloudflare Page Rule

For more granular control, you can create a Page Rule to enforce HTTPS on specific paths. Go to **Rules → Page Rules → Create Page Rule**:

```
URL: http://yourdomain.com/*
Setting: Always Use HTTPS
```

---

### Step 2E — Verify via DevTools

1. Open your app URL in Chrome
2. Press `F12` → **Console** tab
3. There should be **zero** mixed content warnings
4. In the **Network** tab, confirm all resources load from `https://`

> **Limitation:** Automatic HTTPS Rewrites only works on resources Cloudflare can detect in the HTML response. JavaScript that dynamically constructs `http://` URLs at runtime **will not** be caught by this fix — use Fix 1 or Fix 3 for those cases.

---

## Fix 3 — Protocol-Relative URLs

This is a **code-level fix** for any hardcoded `http://` references in your templates, JS files, or config. Protocol-relative URLs have no scheme prefix, so the browser automatically uses whatever protocol the current page uses (`https://` when on an HTTPS page).

### Step 3A — In HTML Templates (Blade / PHP)

```html
<!-- ❌ Hardcoded HTTP — will be blocked by the browser -->
<script src="http://yourdomain.com/js/app.js"></script>
<link href="http://yourdomain.com/css/app.css" rel="stylesheet">
<img src="http://yourdomain.com/images/logo.png">

<!-- ✅ Protocol-relative — inherits current page scheme -->
<script src="//yourdomain.com/js/app.js"></script>
<link href="//yourdomain.com/css/app.css" rel="stylesheet">
<img src="//yourdomain.com/images/logo.png">

<!-- ✅ Root-relative (best for same-domain assets) -->
<script src="/js/app.js"></script>
<link href="/css/app.css" rel="stylesheet">
<img src="/images/logo.png">
```

> Root-relative URLs (`/path/to/asset`) are the safest choice for same-domain assets — they don't depend on scheme detection at all.

---

### Step 3B — In Laravel Blade Templates

Never hardcode `http://` — always use Laravel's URL helpers which respect `forceScheme()` from Fix 1:

```blade
{{-- ❌ Hardcoded — breaks under HTTPS --}}
<script src="http://{{ $appDomain }}/js/app.js"></script>

{{-- ✅ asset() helper — always uses the correct scheme --}}
<script src="{{ asset('js/app.js') }}"></script>

{{-- ✅ Named route --}}
<a href="{{ route('dashboard') }}">Dashboard</a>

{{-- ✅ Form action --}}
<form action="{{ route('upload.store') }}" method="POST" enctype="multipart/form-data">
    @csrf
    <input type="file" name="file">
    <button type="submit">Upload</button>
</form>

{{-- ✅ For API base URL in meta tag (used by JS) --}}
<meta name="base-url" content="{{ url('/') }}">
```

---

### Step 3C — In JavaScript / Axios / Fetch

```js
// ❌ Hardcoded HTTP — blocked on HTTPS pages
const response = await fetch('http://yourdomain.com/api/upload', {
    method: 'POST',
    body: formData,
});

// ✅ Option 1: Protocol-relative
const response = await fetch('//yourdomain.com/api/upload', {
    method: 'POST',
    body: formData,
});

// ✅ Option 2: Root-relative (best when JS runs on the same domain)
const response = await fetch('/api/upload', {
    method: 'POST',
    body: formData,
});

// ✅ Option 3: Read base URL from Blade meta tag
const baseUrl = document.querySelector('meta[name="base-url"]').content;
const response = await fetch(`${baseUrl}/api/upload`, {
    method: 'POST',
    body: formData,
});
```

---

### Step 3D — In Axios Config (if using Axios)

If you set a global `baseURL` in your Axios config:

```js
// ❌ Hardcoded HTTP
axios.defaults.baseURL = 'http://yourdomain.com';

// ✅ Read from meta tag
axios.defaults.baseURL = document.querySelector('meta[name="base-url"]').content;

// ✅ Or root-relative
axios.defaults.baseURL = '/';
```

---

### Step 3E — Find All Remaining Hardcoded HTTP References

Use grep to scan your entire project from your Laragon project root (run in Git Bash or WSL):

```bash
# Scan Blade templates
grep -rn "http://" resources/views/ --include="*.blade.php"

# Scan public JS files
grep -rn "http://" public/js/ --include="*.js"

# Scan source JS files (if using Vite/Mix)
grep -rn "http://" resources/js/ --include="*.js"

# Scan config files
grep -rn "http://" config/ --include="*.php"

# Scan all PHP files (broad scan)
grep -rn "http://" app/ --include="*.php"
```

Replace every hit with a root-relative (`/path`), protocol-relative (`//domain/path`), or Laravel helper equivalent.

---

## Non-Laravel Apps

### Plain PHP

Add this at the top of your entry file (e.g. `index.php`) to tell PHP the connection is HTTPS:

```php
if (isset($_SERVER['HTTP_X_FORWARDED_PROTO']) && $_SERVER['HTTP_X_FORWARDED_PROTO'] === 'https') {
    $_SERVER['HTTPS'] = 'on';
    $_SERVER['SERVER_PORT'] = 443;
}
```

Then use `$_SERVER['HTTPS']` safely in your URL building logic.

---

### Node.js / Express

Enable proxy trust so Express reads forwarded headers from Cloudflare:

```js
const express = require('express');
const app = express();

// Trust the first hop (Cloudflare)
app.set('trust proxy', 1);

// Now req.secure and req.protocol will reflect HTTPS correctly
app.use((req, res, next) => {
    if (!req.secure) {
        return res.redirect(301, 'https://' + req.headers.host + req.url);
    }
    next();
});
```

---

### Nginx Virtual Host in Laragon

Edit your Laragon virtual host config at `C:\laragon\etc\nginx\sites-enabled\yourapp.conf`:

```nginx
server {
    listen 80;
    server_name yourapp.test;
    root "C:/laragon/www/yourapp/public";

    location / {
        try_files $uri $uri/ /index.php?$query_string;

        # Pass forwarded headers to the app
        proxy_set_header X-Forwarded-Proto $http_x_forwarded_proto;
        proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
        proxy_set_header X-Real-IP         $remote_addr;
        proxy_set_header Host              $host;
    }

    location ~ \.php$ {
        fastcgi_pass 127.0.0.1:9000;
        fastcgi_index index.php;
        include fastcgi_params;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        fastcgi_param HTTPS on;
    }
}
```

After editing, restart Nginx from the Laragon tray icon.

---

## Which Fixes to Apply Together

| Scenario                               | Fix 1        | Fix 2             | Fix 3                           |
|----------------------------------------|--------------|-------------------|---------------------------------|
| Laravel, URLs via helpers only         | ✅ Required  | ✅ Safety net     | ⚠️ Only if hardcoded URLs exist |
| Laravel, with hardcoded URLs in JS     | ✅ Required  | ✅ Required       | ✅ Required                     |
| Non-Laravel PHP app                    | —            | ✅ Required       | ✅ Required                     |
| Node.js / Express                      | Use trust proxy | ✅ Required    | ✅ Required                     |
| WordPress                              | Use plugin   | ✅ Required       | ✅ For hardcoded links           |

> **Recommendation:** Apply all three together for the most bulletproof result.

---

## Quick Checklist

- [ ] Cloudflare Tunnel origin service set to `http://localhost:80` (Type: HTTP)
- [ ] `APP_URL` starts with `https://` in `.env`
- [ ] `TrustProxies` configured with `$proxies = '*'` and `HEADER_X_FORWARDED_PROTO`
- [ ] `URL::forceScheme('https')` added in `AppServiceProvider::boot()`
- [ ] Cloudflare SSL mode set to **Full** (not Flexible, not Full Strict)
- [ ] **Automatic HTTPS Rewrites** enabled in Cloudflare Edge Certificates
- [ ] **Always Use HTTPS** enabled in Cloudflare Edge Certificates
- [ ] All artisan caches cleared (`config`, `route`, `view`, `cache`)
- [ ] No hardcoded `http://` URLs remaining in Blade templates
- [ ] No hardcoded `http://` URLs in JavaScript / Axios config
- [ ] Verified with DevTools Console — zero mixed content warnings
