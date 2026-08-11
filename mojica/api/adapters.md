# mojica API Adapter Design

## 1. Purpose

This document defines the responsibilities and boundaries of the Adapter that converts the `ImageGenerationPort` contract into HTTP communication with the Glyph Forge API.

The Adapter is placed in the Infrastructure layer and contains dependencies on external technologies such as the Glyph Forge API, HTTP client, JSON library, and ASP.NET Core.

## 2. Scope

This document defines:

- Responsibilities of `GlyphForgeImageGenerationAdapter`
- The conversion boundary from `ImageGenerationPort` to the external API
- Selection of an external endpoint according to `ImageType`
- Conversion from `HexColor` to RGB values
- Validation of external responses and conversion to Domain results
- Conversion of external API errors to `ImageGenerationPortError`
- The Adapter test contract

## 3. Dependency Direction

```text
ImageGenerationPort
    ▲
    │ implements
    │
GlyphForgeImageGenerationAdapter
    │
    ├── HTTP Client
    ├── Glyph Forge Request/Response DTO
    └── Glyph Forge API
```

`GlyphForgeImageGenerationAdapter` implements `ImageGenerationPort` and keeps dependencies on Glyph Forge API URLs, the HTTP client, and external DTOs inside the Adapter.

## 4. Adapter Responsibilities

### Responsibilities

- Receive a validated `ImageGenerationRequest`
- Convert `ImageType` to a Glyph Forge API endpoint
- Convert the value of `HexColor` to RGB values
- Create a Glyph Forge API-specific request DTO
- Send an HTTP request
- Handle timeouts and cancellation
- Validate the HTTP status, headers, Content-Type, and image data
- Convert a Glyph Forge API-specific response DTO or binary data into `GeneratedImage`
- Convert external API-specific failures into `ImageGenerationPortError`

## 5. Port Implementation Contract

The Adapter implements the following Port contract:

```text
generate(
    request: ImageGenerationRequest
) -> Result<GeneratedImage, ImageGenerationPortError>
```

Input is limited to an `ImageGenerationRequest` that satisfies the Port contract. The Adapter does not receive unvalidated strings or HTTP requests.

On success, return a `GeneratedImage` containing image binary data and media type.

On failure, return `ImageGenerationPortError` instead of HTTP client exceptions, JSON deserialization exceptions, or external API-specific error types.

## 6. Request Conversion

The Adapter converts Domain Models to external API DTOs at the following boundary:

```text
ImageGenerationRequest
        │
        ├── ImageType ───────────▶ Glyph Forge endpoint
        ├── RenderText ──────────▶ Glyph Forge text field
        ├── PatternCharacter ────▶ Glyph Forge character field
        ├── HexColor ────────────▶ RgbColor ──▶ Glyph Forge color DTO
        └── PatternCharacter ────▶ Glyph Forge character field
```

The conversion follows the Glyph Forge API contract defined in Section 15. External API-specific DTOs are used only inside the Adapter.

## 7. Endpoint Selection

Use the following mapping between `ImageType` and Glyph Forge API endpoints:

| `ImageType` | Method | Path |
| --- | --- | --- |
| `standard` | `POST` | `/images` |
| `x-background` | `POST` | `/images/background` |
| `x-icon` | `POST` | `/images/x-icon` |

Keep this mapping inside the Adapter's Infrastructure implementation.

An undefined `ImageType` must not reach endpoint selection. If an undefined value is detected defensively, handle it as `FAILED` and do not send a request to the external API.

## 8. HEX-to-RGB Conversion

Before sending a request to the external API, the Adapter obtains RGB values from `HexColor`.

```text
HexColor("#FF69B4")
        │
        ▼
RgbColor(red: 255, green: 105, blue: 180)
        │
        ▼
Glyph Forge API-specific color DTO
```

`HexColor` and `RgbColor` are responsible for validating the `#RRGGBB` format, interpreting hexadecimal values, and validating component ranges. The Adapter only places the obtained RGB values into the Glyph Forge API-specific DTO.

## 9. HTTP Request

The Adapter sends HTTP requests that satisfy the following requirements:

- Use the `POST` method
- Use the path corresponding to `ImageType`
- Use the Content-Type defined by the Glyph Forge API contract
- Obtain required credentials and configuration values from a secure configuration boundary
- Propagate the cancellation token to the HTTP request
- Apply the configured timeout
- Do not pass HTTP input DTOs or unvalidated values directly into the request body

The specific Glyph Forge API request DTO follows the contract in Section 15. Do not record secrets in source code or logs.

## 10. HTTP Response

The Adapter processes an external API response in the following order:

1. Check the HTTP status
2. Convert `429 Too Many Requests` to `RATE_LIMITED`
3. Set `retryAfter` when `Retry-After` can be interpreted safely
4. Convert a timeout to `TIMEOUT`
5. Convert other communication failures or unavailable states to `UNAVAILABLE`
6. Check the Content-Type of a successful response
7. Verify that image binary data exists and can be read
8. Convert the image data and media type into `GeneratedImage`
9. Convert an uninterpretable image response to `INVALID_RESPONSE`

When the Glyph Forge API indicates an image generation failure, convert it to `FAILED`. Do not pass the external API error body, stack trace, internal URL, or credentials to an upper layer.

## 11. Error Conversion

The Adapter converts external API failures into `ImageGenerationPortError`.

| External event | Port error |
| --- | --- |
| Rate limit | `RATE_LIMITED` |
| Timeout | `TIMEOUT` |
| DNS, connection, TLS, or HTTP client communication failure | `UNAVAILABLE` |
| Response that cannot be interpreted as an image | `INVALID_RESPONSE` |
| Glyph Forge API generation failure | `FAILED` |

## 12. Timeout and Cancellation

The Adapter must not wait for the external API beyond the configured timeout.

When the caller supplies a cancellation token, propagate it to the HTTP client. Convert internal exceptions to `TIMEOUT` or an appropriate Port error so the caller can distinguish cancellation and timeout outcomes.

Do not automatically retry after a timeout. The image generation API has no idempotency-key contract, so this avoids duplicate generation caused by retries.

## 13. Configuration and Secrets

Inject environment-dependent values used by the Adapter through the configuration boundary.

- Glyph Forge API base URL
- Connection and response timeouts
- Required service-specific headers

Do not hard-code the base URL or secrets in the Adapter. Do not include secrets in exceptions, logs, Port errors, or public API responses.

## 14. Test Contract

### Small Tests

Replace the HTTP client and verify conversion results observable from the Adapter.

- Convert `standard` to `/images`
- Convert `x-background` to `/images/background`
- Convert `x-icon` to `/images/x-icon`
- Convert `#FF69B4` to RGB values 255, 105, and 180
- Convert a successful response to `GeneratedImage`
- Convert 429 to `RATE_LIMITED`
- Convert a timeout to `TIMEOUT`
- Convert a communication failure to `UNAVAILABLE`
- Convert an invalid image response to `INVALID_RESPONSE`
- Exclude external API internal details from error results

### Medium Tests

Use an available test Glyph Forge API or HTTP stub to verify the contract from the Adapter to the external API boundary.

- Process the actual Content-Type and image binary data
- Interpret `Retry-After` safely
- Propagate timeout and cancellation to external communication
- Convert each external API error response into a Port error

For tests using real Glyph Forge API communication, do not store secrets in the repository or share authentication state or ports between tests.

## 15. Glyph Forge API Contract

Based on the Glyph Forge API implementation, the Adapter's external communication contract is finalized as follows.

### Request

| mojica value | Glyph Forge API field | Conversion |
| --- | --- | --- |
| `text` | `frame_text` | Pass the string unchanged |
| `foregroundCharacter` | `inner_text` | Pass the string unchanged |
| `backgroundCharacter` | `outer_text` | Pass the string unchanged |
| `foregroundColor` | `inner_color` | Convert `RgbColor` to `[R, G, B]` |
| `backgroundColor` | `outer_color` | Convert `RgbColor` to `[R, G, B]` |

Use the following JSON format:

```json
{
  "frame_text": "KA",
  "inner_text": "🌻",
  "outer_text": "☀",
  "inner_color": [255, 212, 0],
  "outer_color": [255, 105, 180]
}
```

Do not specify `frame_font_size` or `output_font_size` in the mojica API. Use the Glyph Forge API default value of `20`.

Use `application/json; charset=utf-8` as the request Content-Type. Under the current Glyph Forge API contract, do not add an authentication header.

### Endpoints

| `ImageType` | Method | Path |
| --- | --- | --- |
| `standard` | `POST` | `/images` |
| `x-background` | `POST` | `/images/background` |
| `x-icon` | `POST` | `/images/x-icon` |

### Success Response

- HTTP status is `200 OK`
- Content-Type is `image/png`
- Response body is PNG image binary data
- The Adapter creates `GeneratedImage` from the image binary data and Content-Type

### Error Response

| Glyph Forge API response | Adapter handling |
| --- | --- |
| `422 Unprocessable Entity` | `FAILED` |
| `429 Too Many Requests` | `RATE_LIMITED` |
| `503 Service Unavailable` | `UNAVAILABLE` |
| Other 5xx responses | `FAILED` |
| A 2xx response that cannot be interpreted as an image | `INVALID_RESPONSE` |

For `429` and `503`, set `retryAfter` to the integer number of seconds specified by `Retry-After`. The Glyph Forge API allows a burst of three requests per client and replenishes ten requests per minute when rate-limited; when capacity is unavailable, it returns `503` with `Retry-After: 1`.

### Timeout and Retry

The Glyph Forge API image generation limit is 30 seconds. Set the Adapter's HTTP client timeout to 35 seconds so the response can be received.

Convert an HTTP client timeout to `TIMEOUT`. The Adapter does not automatically retry on timeout, communication failure, or `503`. The image generation API has no idempotency-key contract, so this avoids duplicate generation caused by retries.

## 16. Decisions

- Place the Adapter in the Infrastructure layer
- Have `GlyphForgeImageGenerationAdapter` implement `ImageGenerationPort`
- Keep the external API URL, HTTP client, and DTOs from leaking outside the Adapter
- Do not duplicate Domain Model validation in the Adapter
- Convert external API failures to `ImageGenerationPortError`
- Follow the request, response, and error contracts for the Glyph Forge API defined in Section 15
