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

## 1a. Migrating from an existing `captchaaddon` setup

> **Applies to you if:** you are already on `2211-jdk21.13` with the legacy
> `captchaaddon` still installed and working, using `recaptcha.publickey` /
> `recaptcha.privatekey` in `local.properties` and per-environment
> `ccv2-config` properties for DEV / QAS / PROD.

### Deprecated ≠ deleted (why it still works)

`captchaaddon` still **ships and functions** on `2211-jdk21.13` — that is why
your current setup works. But it is **deprecated and unsupported**: it can be
removed in a future update patch, and SAP will not fix issues raised against
it. So you are not forced to migrate in the same cutover as the JDK21 move —
you can keep it as a **bridge** and migrate on your own schedule.

| Option | What it means | When to pick it |
|---|---|---|
| **A. Keep `captchaaddon` (bridge)** | No code change; accept unsupported-status risk | De-risk the JDK21 cutover; migrate later |
| **B. Migrate to plug-in point** | Remove addon, implement strategy, re-map keys | Supported end state (recommended target) |

This guide assumes **Google reCAPTCHA v2** (checkbox) — the `publickey` /
`privatekey` naming is the classic v2 convention. For v3 (invisible/score),
see the score-threshold note in §4.1.

### Property mapping — old → new

The two old properties do **not** map 1:1; the new model splits them by *where
they live*:

| Old (`captchaaddon`) | New (plug-in point) | Where it lives |
|---|---|---|
| `recaptcha.privatekey` (secret) | secret read by your **validation strategy**, server-side | property `captcha.recaptcha.secretKey` — **never** sent to storefront |
| `recaptcha.publickey` (site key) | site key returned by `GET /{baseSiteId}/captcha/config` | BaseStore field (Backoffice) + property `captcha.recaptcha.publicKey` |
| addon auto-enabled | **`Captcha Widget Enabled`** flag | per-BaseStore, Backoffice |

### Your DEV / QAS / PROD `ccv2-config`

This carries over almost unchanged — only the **key names** change. Keep the
**distinct reCAPTCHA key pair per environment** you already maintain (each env's
domain is registered separately in the Google reCAPTCHA admin console). The
secret key stays server-side; only the public/site key reaches the browser.

```properties
# BEFORE (captchaaddon) — per env in ccv2-config (DEV / QAS / PROD)
recaptcha.publickey=<env site key>
recaptcha.privatekey=<env secret key>

# AFTER (plug-in point) — same per-env split, renamed
captcha.recaptcha.publicKey=<env site key>
captcha.recaptcha.secretKey=<env secret key>
```

> In CCv2, these remain environment-specific properties (Cloud Portal →
> environment → Properties, or your `ccv2-config` / manifest `properties` per
> aspect). No change to *how* you scope them per environment — just the names.

### Step-by-step migration (A → B)

1. **Deploy the new strategy alongside the addon.** Add the `cf400captcha`
   extension (§4) with the validation strategy + Spring alias. Do **not** remove
   `captchaaddon` yet — both can coexist while you validate.
2. **Add the renamed properties** (`captcha.recaptcha.secretKey` /
   `.publicKey`) to `local.properties` and to each env's `ccv2-config`,
   alongside the old ones for now.
3. **Enable the plug-in path per BaseStore:** set `Captcha Widget Enabled = true`
   and the site key on the BaseStore (§5).
4. **Confirm the OCC contract:** `GET /{baseSiteId}/captcha/config` returns
   `enabled: true` + the new site key; registration sends the
   `sap-commerce-cloud-captcha-token` header (§6, §7).
5. **Switch the storefront** to the Composable Captcha component + `CaptchaGuard`
   (if you are on Accelerator JSP, the addon's JSP widget is what you are
   replacing — see note below).
6. **Remove `captchaaddon`:** delete it from `localextensions.xml` and from the
   storefront addon install, then delete the old `recaptcha.publickey` /
   `recaptcha.privatekey` properties from all envs.
7. **Regression test** registration on every environment (DEV → QAS → PROD).

> **Accelerator (JSP) storefront note:** `captchaaddon` renders a server-side
> JSP widget. The plug-in point is OCC/header-based, designed for Composable
> Storefront. If your B2C storefront is **Accelerator JSP** (not Spartacus),
> confirm with SAP how they expect the widget rendered on JSP once the addon is
> gone — the plug-in point covers *validation*, but the JSP-side *widget
> rendering* was the addon's job. This is the one gap to clarify before fully
> removing the addon on a JSP storefront.

### Rollback

Because you migrate additively (step 1–4 leave `captchaaddon` in place), rollback
is: set `Captcha Widget Enabled = false` (or revert to the addon's flag) and
redeploy — the old addon path resumes with the old properties still present.

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

**The interface (confirmed OOTB, `2211-jdk21`):**
`de.hybris.platform.commercewebservicescommons.strategies.CaptchaValidationStrategy`
has **four** abstract methods — all must be implemented, which is exactly why a
partial implementation fails with *"is not abstract and does not override
abstract method preCheckToken(CaptchaValidationContext)"*:

| Method | Returns | Purpose |
|---|---|---|
| `preCheckToken(CaptchaValidationContext)` | `boolean` | cheap format/regex pre-check before the remote call |
| `validate(CaptchaValidationContext)` | `CaptchaValidationResult` | **real validation** — call your provider here |
| `getProviderType()` | `CaptchaProviderWsDtoType` | which provider (`DEFAULT`, …) |
| `getVersion()` | `CaptchaVersionWsDtoType` | provider version (`V1`, …) |

DTOs live in `de.hybris.platform.commercewebservicescommons.dto.captcha`.
`CaptchaValidationContext.getCaptchaToken()` returns the token;
`CaptchaValidationResult` has `setSuccess(boolean)` (+ a reason for failures).

> **Why this matters:** the OOTB `DefaultCaptchaValidationStrategy.validate()`
> **always returns `success = true`** (a no-op stub). So on `2211-jdk21.13` the
> plug-in exists but *validates nothing* until you supply real logic. This is
> the migration.

```java
package com.alfanar.hybris.ceramic.core.strategies.impl;

import de.hybris.platform.commercewebservicescommons.dto.captcha.CaptchaProviderWsDtoType;
import de.hybris.platform.commercewebservicescommons.dto.captcha.CaptchaValidationContext;
import de.hybris.platform.commercewebservicescommons.dto.captcha.CaptchaValidationResult;
import de.hybris.platform.commercewebservicescommons.dto.captcha.CaptchaVersionWsDtoType;
import de.hybris.platform.commercewebservicescommons.strategies.CaptchaValidationStrategy;
import de.hybris.platform.servicelayer.config.ConfigurationService;

import java.net.URI;
import java.net.URLEncoder;
import java.net.http.HttpClient;
import java.net.http.HttpRequest;
import java.net.http.HttpResponse;
import java.nio.charset.StandardCharsets;
import java.util.Objects;
import java.util.regex.Pattern;

import com.fasterxml.jackson.databind.JsonNode;
import com.fasterxml.jackson.databind.ObjectMapper;
import org.apache.commons.lang3.StringUtils;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

/**
 * Google reCAPTCHA v2 (checkbox) implementation of the CAPTCHA plug-in point.
 * Replaces the no-op DefaultCaptchaValidationStrategy with real verification.
 */
public class GoogleReCaptchaValidationStrategy implements CaptchaValidationStrategy
{
    private static final Logger LOG = LoggerFactory.getLogger(GoogleReCaptchaValidationStrategy.class);
    private static final String VERIFY_URL = "https://www.google.com/recaptcha/api/siteverify";
    private static final Pattern TOKEN_PATTERN = Pattern.compile("^[0-9a-zA-Z-_]+$");

    private ConfigurationService configurationService;
    private final HttpClient httpClient = HttpClient.newHttpClient(); // Java 21 java.net.http
    private final ObjectMapper objectMapper = new ObjectMapper();

    /** Cheap format pre-check (mirrors the OOTB regex). Interceptor calls this first. */
    @Override
    public boolean preCheckToken(final CaptchaValidationContext context)
    {
        final String token = context.getCaptchaToken();
        return Objects.nonNull(token) && TOKEN_PATTERN.matcher(token).matches();
    }

    /** Real validation: verify the token with Google's siteverify endpoint. */
    @Override
    public CaptchaValidationResult validate(final CaptchaValidationContext context)
    {
        final CaptchaValidationResult result = new CaptchaValidationResult();
        final String token = context.getCaptchaToken();
        final String secret = getConfigurationService().getConfiguration()
                .getString("captcha.recaptcha.secretKey", StringUtils.EMPTY);

        if (StringUtils.isBlank(token) || StringUtils.isBlank(secret))
        {
            result.setSuccess(false); // fail closed on missing token/misconfig
            return result;
        }
        try
        {
            final String body = "secret=" + enc(secret) + "&response=" + enc(token);
            final HttpRequest request = HttpRequest.newBuilder(URI.create(VERIFY_URL))
                    .header("Content-Type", "application/x-www-form-urlencoded")
                    .POST(HttpRequest.BodyPublishers.ofString(body))
                    .build();
            final HttpResponse<String> resp =
                    httpClient.send(request, HttpResponse.BodyHandlers.ofString());
            final JsonNode json = objectMapper.readTree(resp.body());
            boolean success = json.path("success").asBoolean(false);
            // reCAPTCHA v3 only — also enforce a score threshold:
            // success = success && json.path("score").asDouble(0d) >= 0.5d;
            result.setSuccess(success);
        }
        catch (final InterruptedException e)
        {
            Thread.currentThread().interrupt();
            LOG.error("reCAPTCHA verification interrupted", e);
            result.setSuccess(false);
        }
        catch (final Exception e)
        {
            LOG.error("reCAPTCHA verification failed", e);
            result.setSuccess(false); // fail closed
        }
        return result;
    }

    @Override
    public CaptchaProviderWsDtoType getProviderType()
    {
        // Keep DEFAULT to override the OOTB strategy in place (single provider).
        // If your CaptchaProviderWsDtoType enum has a reCAPTCHA-specific value
        // and you run multiple providers, return that instead.
        return CaptchaProviderWsDtoType.DEFAULT;
    }

    @Override
    public CaptchaVersionWsDtoType getVersion()
    {
        return CaptchaVersionWsDtoType.V1; // set to the enum value matching your reCAPTCHA version
    }

    private static String enc(final String v)
    {
        return URLEncoder.encode(v, StandardCharsets.UTF_8);
    }

    public ConfigurationService getConfigurationService() { return configurationService; }
    public void setConfigurationService(final ConfigurationService cs) { this.configurationService = cs; }
}
```

### 4.2 Register the strategy at the plug-in point (Spring)

The simplest, most reliable migration is to **override the OOTB default strategy
bean** so your implementation is used with no provider-selection plumbing. In
`resources/<ext>-spring.xml`, define your bean with the **same id** the platform
uses for the default strategy, so your extension's definition wins (ensure your
extension loads after `commercewebservicescommons` in `localextensions.xml`).

```xml
<!-- Override the OOTB no-op strategy. Confirm the default bean id by searching
     commercewebservicescommons-spring.xml (typically 'defaultCaptchaValidationStrategy'). -->
<bean id="defaultCaptchaValidationStrategy"
      class="com.alfanar.hybris.ceramic.core.strategies.impl.GoogleReCaptchaValidationStrategy">
    <property name="configurationService" ref="configurationService"/>
</bean>
```

> **Multi-provider alternative:** if the platform keeps a *registry of strategies
> keyed by `getProviderType()`/`getVersion()`*, don't reuse the default id —
> register your bean under a new id, return a distinct `CaptchaProviderWsDtoType`
> from `getProviderType()`, and configure that provider type as active for the
> store. For a single reCAPTCHA provider, overriding the default bean (above) is
> simpler and sufficient.

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

## 9. Confirmed against `2211-jdk21` OOTB (`DefaultCaptchaValidationStrategy`)

These were verified from the OOTB source and are baked into §4 above:

1. **Interface:** `de.hybris.platform.commercewebservicescommons.strategies.CaptchaValidationStrategy`
   — four abstract methods: `preCheckToken(CaptchaValidationContext):boolean`,
   `validate(CaptchaValidationContext):CaptchaValidationResult`,
   `getProviderType():CaptchaProviderWsDtoType`, `getVersion():CaptchaVersionWsDtoType`.
2. **DTOs:** `de.hybris.platform.commercewebservicescommons.dto.captcha.*`
   (`CaptchaValidationContext.getCaptchaToken()`, `CaptchaValidationResult.setSuccess(boolean)`).
3. **OOTB default** = `DefaultCaptchaValidationStrategy`, whose `validate()` is a
   **no-op returning `success=true`** — must be overridden (§4.2).

Still worth confirming in *your* build:

- The exact **default bean id** to override in Spring (search
  `commercewebservicescommons-spring.xml`; typically `defaultCaptchaValidationStrategy`).
- Whether strategies are selected via a **registry keyed by provider type**
  (multi-provider) vs. a single default bean (see §4.2 alternative).
- The available values of the **`CaptchaProviderWsDtoType` / `CaptchaVersionWsDtoType`**
  enums (for `getProviderType()` / `getVersion()`).
- The **header constant** the interceptor reads (currently `sap-commerce-cloud-captcha-token`).

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
