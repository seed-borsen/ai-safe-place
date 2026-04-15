---
name: mjml-email-templates
description: Working with MJML email templates in service-postoffice. Activate when building or editing newsletter templates, email layouts, personalization tokens, or the RequestDataMail/Office base classes.
metadata:
  tags: mjml, email, newsletters, postoffice, templates
---

## Overview

`service-postoffice` renders emails using MJML templates. MJML is compiled by an **external HTTP service** (not inline) — never attempt to render MJML locally. Templates use PHP interpolation for dynamic data and proprietary personalization tokens for subscriber-level data.

---

## Class Architecture

```
Office (base)
└── RequestDataMail extends Office
    ├── buildMjml()           — entry point, receives POST/GET JSON payload
    └── generateMjmlTemplate($dataResult): string
            — buffers output with ob_start(), includes driver template
```

Each newsletter type is a "driver" with its own `template.mjml` file:
```
drivers/
  borsen-tag/template.mjml
  borsen-breaking/template.mjml
  ...
```

---

## Template Variable Pattern

Templates use PHP short echo tags, not Blade:

```php
<?= $data['preheader'] ?>
<?= $data['newsletter_name'] ?>

<?php foreach($data['latest'] ?? [] as $index => $article) { ?>
    <mj-image src="<?= $article['image'] ?>"></mj-image>
    <mj-text>
        <a href="<?= $article['link'] ?>?sso={{account.token}}&<?= $article['utm_link'] ?>">
            <?= $article['title'] ?>
        </a>
        <?= $article['excerpt'] ?>
    </mj-text>
<?php } ?>
```

Always use `?? []` on arrays to avoid null iteration errors.

---

## Personalization Tokens

These are **not PHP** — they are processed by the email delivery platform after rendering:

| Token | Purpose |
|-------|---------|
| `{{account.token}}` | SSO/subscriber token for paywalled links |
| `{{EMAIL=md5}}` | MD5 hash of subscriber email (tracking) |
| `{{resourcereference}}` | Unsubscribe resource reference |
| `{{newsletter_id}}` | Newsletter ID for unsubscribe links |

These must remain as literal strings in the output — never escape or PHP-encode them.

---

## MJML Rendering

Rendering is done via an external HTTP endpoint:

```php
$endpoint = getenv('MJML_ENDPOINT');

$response = $this->guzzle->post($endpoint, [
    'json' => ['mjml' => $mjmlString]
]);

$html = json_decode($response->getBody(), true)['html'];
```

The MJML endpoint is environment-configured. Never hardcode it.

---

## Adding a New Newsletter Type

1. Create `drivers/{newsletter-slug}/template.mjml`
2. Extend `RequestDataMail` or add a new driver class
3. Define the data shape expected in `buildMjml()`
4. Add `{{account.token}}` to all subscriber-facing links
5. Add `{{resourcereference}}` and `{{newsletter_id}}` to the unsubscribe footer

---

## Common MJML Components Used

```xml
<mjml>
  <mj-head>
    <mj-preview><?= $data['preheader'] ?></mj-preview>
    <mj-attributes>
      <mj-all font-family="Arial, sans-serif" />
    </mj-attributes>
  </mj-head>
  <mj-body>
    <mj-section>
      <mj-column>
        <mj-text><?= $data['title'] ?></mj-text>
        <mj-image src="<?= $data['image'] ?>" />
        <mj-button href="<?= $data['link'] ?>?sso={{account.token}}">
          Read more
        </mj-button>
      </mj-column>
    </mj-section>
  </mj-body>
</mjml>
```

---

## Key Rules

- MJML is rendered by an external service — do not install local MJML compiler
- Personalization tokens (`{{...}}`) are delivery-platform syntax — preserve them as literal strings
- Use `ob_start()` / `ob_get_clean()` for template buffering, not string concatenation
- Always null-coalesce array access in templates: `$data['key'] ?? ''`
- Data arrives as decoded JSON — validate shape in `buildMjml()` before passing to template
