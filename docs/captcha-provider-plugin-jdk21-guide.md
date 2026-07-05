# Implementing the Generic CAPTCHA Provider Plug-In (SAP Commerce 2211-jdk21)

> Re-integrating a security CAPTCHA (e.g. Google reCAPTCHA) on the **Customer
> Registration** flow after upgrading to `2211-jdk21`, where the legacy
> `captchaaddon` / built-in Google reCAPTCHA integration has been **removed**.

**Target version:** `2211-jdk21.13` (n-1 pattern)
**Storefront:** Composable Storefront (Spartacus) — the OCC contract also applies to any custom OCC client.

---

## 1. Background — what changed and why

| Aspect | Before (≤ JDK17 line) | After (`2211-jdk21`, plug-in model since **2211.28**) |
|---|---|---|
| Google reCAPTCHA | Built-in via `captchaaddon` extension | **Removed.** No built-in provider |
| Integration model | Provider hardcoded into the addon | **Generic CAPTCHA Provider Plug-In point** — you supply the provider |
| Provider choice | Google reCAPTCHA only | **Any** provider (reCAPTCHA v2/v3, hCaptcha, Turnstile, …) |
| Validation | Addon strategy | **Custom validation strategy bean** you register at the plug-in point |

**Key clarification (per SAP):** Google reCAPTCHA is **not** discontinued as a
*provider*. Only SAP's *hardcoded integration* (`captchaaddon`) was deprecated
and removed. You may keep using Google reCAPTCHA — you just implement it
yourself against the new plug-in point.

> This task belongs in the **"Follow-up Manual Activities"** bucket of the
> jdk21 framework upgrade. OpenRewrite recipes do **not** migrate this for you.

---

## 2. Architecture / request flow

```
[Storefront registration form]
        │  user solves CAPTCHA widget  →  provider returns a token
        ▼
[OCC call: POST .../users  (registration)]
        │  HTTP header: sap-commerce-cloud-captcha-token: <token>
        ▼
[commercewebservicescommons: CaptchaValidationInterceptor]
        │  triggers on controller methods annotated @CaptchaAware
        │  (only if CAPTCHA is enabled for the current BaseStore)
        ▼
[CAPTCHA validation strategy bean]  ◄── ★ THE PLUG-IN POINT you implement
        │  calls provider verify endpoint, e.g.
        │  POST https://www.google.com/recaptcha/api/siteverify
        ▼
   valid → request proceeds   |   invalid → 4xx, registration rejected
```

Two OCC surfaces are involved:
1. **Config endpoint** — `GET /{baseSiteId}/captcha/config` → returns whether
   CAPTCHA is enabled + the public/site key. Spartacus reads this to decide
   whether to render the widget and send the token header.
2. **Validation** — the `@CaptchaAware` interceptor validates the token header
   on the protected endpoint (registration).

---

## 3. Prerequisites

- Commerce upgraded to `2211-jdk21` (JDK 21 / SapMachine 21, Spring 6, Jakarta).
- `commercewebservicescommons` / OCC extensions present (they carry
  `@CaptchaAware` + `CaptchaValidationInterceptor`).
- `captchaaddon` **removed** from `localextensions.xml` and from any storefront
  addon installation.
- A provider account. For Google reCAPTCHA: a **site key** (public) and a
  **secret key** (private) from https://www.google.com/recaptcha/admin.

---

## 4. Backend implementation (custom extension)

Create/extend a custom extension, e.g. `cf400captcha` (or reuse an existing
`*core` extension).

### 4.1 The validation strategy — the plug-in point

Implement the platform CAPTCHA validation strategy interface. The strategy
receives the token and must return whether it is valid.

> ⚠️ **Confirm the exact interface + method name against your version's
> javadoc** (`GET /doc/.../commercewebservicescommons/...`), because names have
> shifted across releases. In the `2211-jdk21` line the plug-in point is the
> CAPTCHA validation strategy bean invoked by `CaptchaValidationInterceptor`.
> The pattern below is stable even if the FQN differs slightly.

```java
package com.cf400.captcha.strategy;

import de.hybris.platform.servicelayer.config.ConfigurationService;
// import de.hybris.platform.commercewebservicescommons.strategies.CaptchaValidationStrategy; // verify FQN

import java.net.URI;
import java.net.http.HttpClient;
import java.net.http.HttpRequest;
import java.net.http.HttpResponse;
import java.net.URLEncoder;
import java.nio.charset.StandardCharsets;

import com.fasterxml.jackson.databind.ObjectMapper;
import org.apache.commons.lang3.StringUtils;

/**
 * Google reCAPTCHA v2/v3 implementation of the CAPTCHA plug-in point.
 * Registered as the strategy bean the interceptor delegates to.
 */
public class GoogleReCaptchaValidationStrategy /* implements CaptchaValidationStrategy */ {

    private static final String VERIFY_URL = "https://www.google.com/recaptcha/api/siteverify";
    private ConfigurationService configurationService;
    private final HttpClient httpClient = HttpClient.newHttpClient(); // Java 21 java.net.http
    private final ObjectMapper objectMapper = new ObjectMapper();

    // @Override
    public boolean validate(final String captchaToken) {
        if (StringUtils.isBlank(captchaToken)) {
            return false;
        }
        final String secret = configurationService.getConfiguration()
                .getString("captcha.recaptcha.secretKey", StringUtils.EMPTY);
        if (StringUtils.isBlank(secret)) {
            return false; // fail closed if misconfigured
        }
        try {
            final String body = "secret=" + enc(secret) + "&response=" + enc(captchaToken);
            final HttpRequest request = HttpRequest.newBuilder(URI.create(VERIFY_URL))
                    .header("Content-Type", "application/x-www-form-urlencoded")
                    .POST(HttpRequest.BodyPublishers.ofString(body))
                    .build();
            final HttpResponse<String> resp =
                    httpClient.send(request, HttpResponse.BodyHandlers.ofString());
            final var node = objectMapper.readTree(resp.body());
            final boolean success = node.path("success").asBoolean(false);
            // For reCAPTCHA v3, also enforce a score threshold:
            // return success && node.path("score").asDouble(0d) >= 0.5d;
            return success;
        } catch (final Exception e) {
            // log + fail closed
            return false;
        }
    }

    private static String enc(final String v) {
        return URLEncoder.encode(v, StandardCharsets.UTF_8);
    }

    public void setConfigurationService(final ConfigurationService cs) {
        this.configurationService = cs;
    }
}
```

### 4.2 Register the strategy at the plug-in point (Spring)

In `resources/<ext>-spring.xml` (web/OCC application context that carries the
interceptor), define your bean and **alias it to the plug-in point bean id**
that the interceptor looks up. This is what "plugging in" means — you override
the default (empty) strategy with yours.

```xml
<bean id="googleReCaptchaValidationStrategy"
      class="com.cf400.captcha.strategy.GoogleReCaptchaValidationStrategy">
    <property name="configurationService" ref="configurationService"/>
</bean>

<!-- Point the platform's plug-in point at your implementation.
     ⚠️ Confirm the exact alias/bean id the interceptor resolves in your
     version (search commercewebservicescommons spring xml for the captcha
     strategy bean id). Common id: 'captchaValidationStrategy'. -->
<alias name="googleReCaptchaValidationStrategy" alias="captchaValidationStrategy"/>
```

### 4.3 Annotate / confirm the protected endpoint

The OOTB registration endpoint (`UsersController#createUser`, i.e.
`POST /{baseSiteId}/users`) is already annotated `@CaptchaAware` in recent
releases. If you use a **custom** registration endpoint, annotate it:

```java
@CaptchaAware
@PostMapping(value = "/users")
public UserWsDTO createUser(...) { ... }
```

The interceptor extracts the token from the request header and calls your
strategy. No manual interceptor wiring needed.

### 4.4 Configuration properties

```properties
# Enable/disable is primarily driven by the BaseStore flag (see §5),
# but keep provider secrets in properties (or, better, a secret store).
captcha.recaptcha.secretKey=<your-google-secret-key>
# The site (public) key is exposed to the storefront via /captcha/config
captcha.recaptcha.publicKey=<your-google-site-key>
```

> Keep the **secret key** server-side only. Never ship it to the storefront.

---

## 5. Enable CAPTCHA for the store (Backoffice)

CAPTCHA activation is per-BaseStore, not just a property:

**Backoffice → Base Commerce → Base Store → (your store) → Properties tab →
`Captcha Widget Enabled` = true**, and set the public/site key field.

The OCC `GET /{baseSiteId}/captcha/config` endpoint returns this state, so the
storefront only renders the widget and sends the token header when enabled.

---

## 6. Storefront (Composable / Spartacus)

1. **Enable the CAPTCHA feature** and provide the reCAPTCHA `grecaptcha` site
   key in the CMS Captcha component / config.
2. Add **`CaptchaGuard`** to the registration route's guards so the widget is
   required before submit.
3. `OccUserProfileAdapter.appendCaptchaToken()` automatically appends the
   `sap-commerce-cloud-captcha-token` header on registration **when CAPTCHA is
   enabled** (per `/captcha/config`).

> Known pitfall (SAP/spartacus #19655): in some 2211.28-era builds the header
> was sent even when disabled. Verify your Spartacus lib version honors the
> `isCaptchaEnabled` flag; upgrade the storefront lib if affected.

---

## 7. Verification checklist

- [ ] `captchaaddon` fully removed from `localextensions.xml` and storefront addons.
- [ ] Custom strategy bean deployed and aliased to the plug-in point id.
- [ ] `Captcha Widget Enabled = true` + site key set on the BaseStore.
- [ ] `GET /{baseSiteId}/captcha/config` returns `enabled: true` + the site key.
- [ ] Registration widget renders; submit sends `sap-commerce-cloud-captcha-token`.
- [ ] Valid token → registration succeeds; missing/invalid token → 4xx.
- [ ] Secret key is server-side only; strategy **fails closed** on error/misconfig.
- [ ] (reCAPTCHA v3) score threshold enforced.

---

## 8. How this slots into your jdk21 upgrade plan

Your stated sequence, with CAPTCHA folded in:

1. Current: `2211.43` on JDK17.
2. Framework upgrade via **OpenRewrite** recipes.
3. Local system → **JDK21** (SapMachine 21).
4. Move functionality to `2211-jdk21.1`.
5. **Follow-up Manual Activities** ← *this CAPTCHA re-integration lives here*
   (remove `captchaaddon`; implement + register the strategy; enable per store).
6. Fix custom-development server-startup issues.
7. Stabilize on `2211-jdk21.1`.
8. Upgrade to target n-1 (`2211-jdk21.13`).

---

## 9. Items to confirm against *your* exact version

Because SAP Help Portal / javadoc are gated, verify these three identifiers in
your installed `2211-jdk21.13` sources/javadoc before coding:

1. The **FQN + method signature** of the CAPTCHA validation strategy interface
   in `commercewebservicescommons`.
2. The **bean id** the `CaptchaValidationInterceptor` resolves (the alias
   target in §4.2).
3. The exact **header constant** name (currently `sap-commerce-cloud-captcha-token`).

---

## References

- SAP Help — Implementing CAPTCHA Plug-In Point:
  https://help.sap.com/docs/SAP_COMMERCE_CLOUD_PUBLIC_CLOUD/e1391e5265574bfbb56ca4c0573ba1dc/f41a260f8d334538ace92004b8e57288.html
- SAP Help — CAPTCHA OCC APIs:
  https://help.sap.com/docs/SAP_COMMERCE_CLOUD_PUBLIC_CLOUD/e1391e5265574bfbb56ca4c0573ba1dc/8a42681cd28549b6a81d6faffc7d6b92.html
- SAP Help — Framework Update (Java JDK 21 & Spring 6.2):
  https://help.sap.com/docs/SAP_COMMERCE_CLOUD_PUBLIC_CLOUD/75d4c3895cb346008545900bffe851ce/9efd1f6212134dec8236a146cac4c98a.html
- KBA 3653999 — Information about JDK 21 Upgrade
- SAP/spartacus #19655 — CAPTCHA enable/disable header behavior:
  https://github.com/SAP/spartacus/issues/19655
- Google reCAPTCHA verify API: https://developers.google.com/recaptcha/docs/verify
