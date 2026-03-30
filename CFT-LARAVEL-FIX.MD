# Cloudflare Tunnel + Laragon Guide

## Fixing Mixed Content Errors

When working with applications behind a Cloudflare tunnel and Laragon, you may encounter mixed content errors. These errors occur when secure HTTPS content is loaded alongside unsecured HTTP content, leading to security warnings in browsers. Below are the various fixes you can implement to resolve these issues:

### Fix 1: Trust Proxies + Force HTTPS

To ensure that your application recognizes the originating protocol (HTTP or HTTPS), you need to trust the Cloudflare proxy. In your `AppServiceProvider.php`, add the following code:

```php
use Illuminate\Foundation\Http\Middleware\TrustProxies;

public function boot()
{
    TrustProxies::setTrustedProxies([$this->app['request']->getClientIp()], TrustProxies::HEADER_X_FORWARDED_ALL);
    
    \\\Request::setTrustedProxies([$this->app['request']->getClientIp()], Request::HEADER_X_FORWARDED_ALL);
}
```

Additionally, force HTTPS in your application by adding this middleware in `app/Http/Kernel.php`:

```php
protected $middleware = [
    \\App\Http\Middleware\ForceHttps::class,
];
```

### Fix 2: Automatic HTTPS Rewrites

In your Cloudflare dashboard, you can enable Automatic HTTPS Rewrites under the SSL/TLS app. This feature helps in converting URLs from HTTP to HTTPS automatically, which will significantly reduce mixed content errors.

### Fix 3: Protocol-Relative URLs

Ensure that all asset URLs are protocol-relative. Instead of using `http://` or `https://`, use `//` for URLs in scripts, images, or stylesheets. For example:

```html
<script src="//example.com/script.js"></script>
<img src="//example.com/image.png" />
```

### Non-Laravel Apps Section

For non-Laravel applications, ensure that your server configuration is set to redirect all HTTP requests to HTTPS, and that all resource links are updated to use HTTPS.

### Comparison Table

| Issue                  | Description                                 | Fix                                 |
|------------------------|---------------------------------------------|-------------------------------------|
| Mixed Content Errors    | Inconsistent content protocols               | Implement trust proxies and force HTTPS |
| Browser Warnings        | Security warnings due to unsecured content  | Enable Automatic HTTPS Rewrites     |
| Loading Speed           | Slower page loads due to mixed content      | Use protocol-relative URLs          |
| General Application Load | Ensures smooth user experience              | Implement fixes outlined above      |

### Quick Checklist

1. Ensure all URLs are HTTPS
2. Enable Automatic HTTPS Rewrites in Cloudflare
3. Implement trust proxies in your application
4. Verify no protocol-relative URLs exist
5. Test your application in multiple browsers to ensure no mixed content errors remain.

By following the above steps, you should be able to fix mixed content errors in your Cloudflare Tunnel + Laragon setup efficiently.