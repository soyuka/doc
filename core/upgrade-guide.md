# Upgrade Guide

## API Platform 4.3

### Hydra Documentation Exposes All ApiResource Declarations

Previously, when a class had multiple `#[ApiResource]` attributes, only the first one was exposed
in the Hydra documentation and entrypoint. Now **all** `ApiResource` declarations are iterated
and exposed in both the Hydra documentation and the entrypoint.

If you have multiple `#[ApiResource]` on the same entity class, clients consuming the Hydra
documentation may see additional classes and entrypoint properties that were previously hidden.

### Hydra Class Identifiers Use Short Name

Hydra documentation classes now consistently use `#ShortName` as their `@id` instead of
schema.org type URIs (e.g., `schema:Product`). Semantic types configured via `types` are now
exposed through `rdfs:subClassOf`.

If your clients parse Hydra documentation and rely on class `@id` values or property ranges
matching schema.org URIs, update them to expect `#ShortName` identifiers.

### JSON-LD `@type` with `itemUriTemplate` and Output DTOs

When using `output` with `itemUriTemplate` on a collection operation, the JSON-LD `@type` now
uses the resource class name instead of the output DTO class name. This ensures semantic
consistency with `itemUriTemplate` behavior.

If you rely on `@type` matching the output DTO class name, update your client code accordingly.

### `enable_link_security` Deprecated and Enabled by Default

The `enable_link_security` configuration option is deprecated and will be removed in API Platform 5.0.
The default value has changed from `false` to `true`, meaning security on Links (sub-resources) is
now always active.

If you explicitly set `enable_link_security: false`, remove it and review your security
configuration for sub-resources.

### `ObjectMapperProcessor` Deprecated

`ObjectMapperProcessor` is deprecated in API Platform 4.3. Use the dedicated `ObjectMapperInputProcessor`
and `ObjectMapperOutputProcessor` instead.

### Doctrine Read-Only Entities Automatically Exclude PUT and PATCH

If a Doctrine entity is marked as `#[ORM\Entity(readOnly: true)]`, API Platform automatically
removes `Put` and `Patch` operations. Read-only entities can still have `Get`, `GetCollection`,
`Post`, and `Delete` operations.

If you have read-only entities with explicit PUT or PATCH operations, they will be silently removed.

### Security `isGranted` Evaluated Before Provider

When `security` is configured on an operation and the expression does not reference the `object`
variable, the access check is now evaluated **before** calling the state provider. This means
unauthorized requests are rejected without triggering database queries.

When the expression uses `object`, the provider is still called first (as before).

### Doctrine Filters Throw on Missing `property`

`ExactFilter`, `IriFilter`, `PartialSearchFilter`, and `UuidFilter` now throw
`InvalidArgumentException` if the parameter's `property` is null. Previously this would silently
produce errors or unexpected behavior. Make sure your filter parameters always specify a `property`.

### JSON:API Spec-Compliant Resource Identifiers

API Platform 4.3 introduces a new `use_iri_as_id` option under `api_platform.jsonapi`.
When set to `false`, JSON:API responses use the entity identifier (e.g., `"10"`) as the
`id` field instead of the IRI (e.g., `"/api/dummies/10"`), which conforms to the JSON:API
specification's convention of using `type` + scalar `id` for resource identification.
The IRI is then moved to `data.links.self`.

The default is `true`, preserving existing behaviour. To opt in to spec-compliant identifiers:

```yaml
# config/packages/api_platform.yaml
api_platform:
    jsonapi:
        use_iri_as_id: false
```

This option is deprecated as of API Platform 4.4 (the `true` default will emit a
deprecation notice) and will be removed in API Platform 5.0, where entity identifiers
become the only supported mode.

If you use JSON:API and rely on the IRI as `data.id`, update your clients before upgrading
to 5.0 to read `data.links.self` for the IRI and `data.id` for the scalar identifier.

## API Platform 3.4

Remove the `keep_legacy_inflector`, the `event_listeners_backward_compatibility_layer` and the `rfc_7807_compliant_errors` flag:

```diff
api_platform:
-        event_listeners_backward_compatibility_layer: false
-        keep_legacy_inflector: false
        extra_properties:
-            standard_put: true
-            rfc_7807_compliant_errors: true
```

If you use a custom normalizer for validation exception use:

```yaml
api_platform:
  validator:
    legacy_validation_exception: true
```

Indeed, we will throw another validation class in API Platform 4 we will throw `ApiPlatform\Validator\Exception\ValidationException` instead of `ApiPlatform\Symfony\Validator\Exception\ValidationException`

It's really important to add the `use_symfony_listeners` flag, set to `true` if you use Symfony listeners or controllers:

```yaml
api_platform:
  use_symfony_listeners: false
```

The `keep_legacy_inflector` flag will be removed from API Platform 4, you need to fix your issues first. In API Platform 3.4, the Inflector is available as a service that you can configure through:

```yaml
api_platform:
  inflector: api_platform.metadata.inflector
```

Implement the `ApiPlatform\Metadata\InflectorInterface` if you need to tweak its behavior.

We added an `hydra_prefix` configuration as the `hydra:` prefix will be removed by default in API Platform 4:

```yaml
api_platform:
  serializer:
    hydra_prefix: false
```

Standard PUT is now `true` by default, you can change its value using:

```yaml
api_platform:
  defaults:
    extra_properties:
      standard_put: true
```

We recommend using the standalone API Platform packages instead of the Core monolithic repository.

Update your `composer.json` like that:

```patch
 {
     "require": {
-        "api-platform/core": "^3",
+        "api-platform/symfony": "^3 || ^4"
+        // also add the extra packages you need, like "api-platform/doctrine-orm"
     }
 }
```

## API Platform 3.1/3.2

This is the recommended configuration for API Platform 3.2. We review each of these changes in this document.

```yaml
api_platform:
  title: Hello API Platform
  version: 1.0.0
  formats:
    jsonld: ['application/ld+json']
  docs_formats:
    jsonld: ['application/ld+json']
    jsonopenapi: ['application/vnd.openapi+json']
    html: ['text/html']
  defaults:
    stateless: true
    cache_headers:
      vary: ['Content-Type', 'Authorization', 'Origin']
    extra_properties:
      standard_put: true
      rfc_7807_compliant_errors: true
  event_listeners_backward_compatibility_layer: false
  keep_legacy_inflector: false
```

### Formats

We noticed that API Platform was enabling `json` by default because of our OpenAPI support. We introduced the new `application/vnd.openapi+json`. Therefore if you want `json` you need to explicitly handle it:

```yaml
formats:
  json: ['application/json']
```

You can also remove documentations you're not using via the new `docs_formats`.

A new option `error_formats` is also used for content negotiation.

### Event listeners

For new users we recommend to use

```yaml
event_listeners_backward_compatibility_layer: false
```

This allows API Platform to not use http kernel event listeners. It also allows you to force options like `read: true` or `validate: true`. This simplifies use cases like [validating a delete operation](https://api-platform.com/docs/v3.2/guides/delete-operation-with-validation/)
Event listeners will not get removed and are not deprecated, they'll use our providers and processors in a future version.

### Inflector

We're switching to `symfony/string` [inflector](https://symfony.com/doc/current/components/string.html#inflector), to keep using `doctrine/inflector` use:

```yaml
keep_legacy_inflector: true
```

We strongly recommend that you use your own inflector anyways with a [PathSegmentNameGenerator](https://github.com/api-platform/core/blob/f776f11fd23e5397a65c1355a9ebcbb20afac9c2/src/Metadata/Operation/UnderscorePathSegmentNameGenerator.php).

### Errors

```yaml
defaults:
  extra_properties:
    rfc_7807_compliant_errors: true
```

As this is an `extraProperties` it's configurable per resource/operation. This is improving the compatibility of Hydra errors with JSON problem. It also enables new extension points on [Errors](https://api-platform.com/docs/v3.2/core/errors/) such as [Error provider](https://api-platform.com/docs/v3.2/guides/error-provider/) and [Error Resource](https://api-platform.com/docs/v3.2/guides/error-resource/).

### OpenApi context

You may want to convert your openApiContext to openapi, doing so is quite fastidious, @lyrixx created a rector script to help if needed:

[https://github.com/lyrixx/rector-apip-openapi](https://github.com/lyrixx/rector-apip-openapi)
