# mojica API Port Design

## 1. Purpose

This document defines the Port contract used by the image generation use case to access an external image generation function.

A Port represents the input, success result, and failure result required by its caller. External service communication formats and HTTP formats are not included in the Port contract.

## 2. ImageGenerationPort

`ImageGenerationPort` receives a validated `ImageGenerationRequest` and returns either an image generation result or a Port error.

### Contract

```text
generate(
    request: ImageGenerationRequest
) -> Result<GeneratedImage, ImageGenerationPortError>
```

### Input

The input must be an `ImageGenerationRequest` that satisfies the Model invariants.

The Port does not accept unvalidated values or HTTP DTOs.

### Success Result

On success, the Port returns a `GeneratedImage`. The image data included in the Port result is:

- `content`: Binary image data
- `mediaType`: Image media type

## 3. ImageGenerationPortError

`ImageGenerationPortError` represents an image generation result as a failure in the Port contract.

### Attributes

| Attribute | Description |
| --- | --- |
| `code` | Language-independent error code |
| `retryAfter` | Number of seconds after which retrying may be possible; unset when unavailable |
| `details` | Supplementary information limited to what may be exposed externally |

### Error Codes

| `code` | Meaning |
| --- | --- |
| `RATE_LIMITED` | Generation is unavailable because a usage limit was reached |
| `TIMEOUT` | A generation result was not obtained within the time limit |
| `UNAVAILABLE` | The image generation function is unavailable |
| `INVALID_RESPONSE` | A response could not be interpreted as a successful result |
| `FAILED` | Image generation failed |

The public attributes of `ImageGenerationPortError` must not contain credentials, internal URLs, stack traces, or communication details that do not need to be exposed.

Set `retryAfter` only when the retryable period can be determined safely.

## 4. Port Test Contract

Port tests verify the input, success, and failure contracts observable from the Port.

At minimum, verify the following behaviors:

- Return a `GeneratedImage` for a valid `ImageGenerationRequest`
- Include image data and media type in the success result
- Return a rate-limit failure as `RATE_LIMITED`
- Return a time-limit failure as `TIMEOUT`
- Return an unavailable-function failure as `UNAVAILABLE`
- Return an uninterpretable response as `INVALID_RESPONSE`
- Return a generation failure as `FAILED`
- Exclude communication details that do not need to be exposed from error results

## 5. Decisions

- Use `ImageGenerationPort` as the outbound image generation contract
- Use `GeneratedImage` for success results and `ImageGenerationPortError` for failure results
- Use `RATE_LIMITED`, `TIMEOUT`, `UNAVAILABLE`, `INVALID_RESPONSE`, and `FAILED` as Port error codes
