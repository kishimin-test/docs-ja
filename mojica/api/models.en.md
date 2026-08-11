# mojica API Model Design

## 1. Purpose

This document defines the business rules of the image generation API independently of external technologies such as HTTP, ASP.NET Core, databases, and the Glyph Forge API.

In this document, a Model is a Domain Model that represents the values and invariants required for image generation. HTTP request/response DTOs and communication models for the Glyph Forge API are not included in the Model.

## 2. Model Responsibilities

The Model is responsible for:

- Representing values required for image generation
- Validating value formats and ranges
- Representing image types
- Representing and converting HEX and RGB color values
- Maintaining invariants for the complete image generation request
- Representing validation results produced by the Domain

The Model is not responsible for:

- Parsing HTTP requests
- Interpreting `Accept-Language`
- Determining HTTP status codes
- Serializing or deserializing JSON
- Storing or retrieving data from a database
- Communicating with the Glyph Forge API
- Depending on ASP.NET Core or an ORM

## 3. Dependency Direction

```text
Controller / Infrastructure
            │
            ▼
          Model
            ▲
            │
          Service
```

The Model must not reference the Controller, Service, Repository, Infrastructure, HTTP, database, or external APIs.

Even when a Repository Interface is placed on the Model side, the Model must not reference its implementation or database types. Repository details are defined in a separate Repository design document.

## 4. Domain Model List

| Model | Kind | Role |
| --- | --- | --- |
| `ImageGenerationRequest` | Aggregate / Domain Model | Groups the values required for image generation and maintains their invariants |
| `ImageType` | Enum / Value | Represents the type of image to generate |
| `RenderText` | Value Object | Represents the text to render |
| `PatternCharacter` | Value Object | Represents the string used for the foreground or background pattern |
| `HexColor` | Value Object | Represents a color in `#RRGGBB` format |
| `RgbColor` | Value Object | Represents a color in RGB format |
| `GeneratedImage` | Domain Result | Represents generated image data |
| `ModelValidationError` | Domain Error | Represents a Model validation failure |

## 5. ImageType

The type of image to generate is represented by a predefined value rather than an arbitrary string.

| Value | Description | Glyph Forge API destination |
| --- | --- | --- |
| `standard` | Standard image | `POST /images` |
| `x-background` | X background image | `POST /images/background` |
| `x-icon` | X icon image | `POST /images/x-icon` |

`ImageType` does not contain a Glyph Forge API path. The Infrastructure adapter is responsible for converting it to an external API endpoint.

Undefined values cannot be created as an `ImageType`. When an external input string cannot be converted to an `ImageType`, the Service or input boundary converts it into a Domain validation error.

## 6. RenderText

`RenderText` represents the text to be rendered as character art.

### Constraints

- Required
- At least 1 character
- At most 64 characters
- Must not consist only of whitespace
- Must not contain control characters

### Creation Rules

Values that do not satisfy the constraints are rejected at creation time. Because a created value always satisfies the constraints, the Service and Infrastructure must not duplicate the same validation.

Character counts use Unicode grapheme clusters. Emoji and combining characters are treated as one character as perceived by the user. A character represented by a surrogate pair must not be counted as two characters.

## 7. PatternCharacter

`PatternCharacter` represents the string that forms the foreground or background pattern.

Character counting follows the same Unicode grapheme-cluster rule as `RenderText`.

The two uses are distinguished by Domain meaning:

| Use | Domain meaning |
| --- | --- |
| `foregroundCharacter` | Character used for rendering |
| `backgroundCharacter` | Character tiled around the rendered characters |

### Common Constraints

- Required
- At least 1 character
- At most 128 characters
- Must not contain control characters
- A value consisting only of whitespace is allowed by itself

### Cross-Field Invariant

Both `foregroundCharacter` and `backgroundCharacter` must not consist only of whitespace.

At least one of them must contain at least one visible character. This cross-field constraint is validated by `ImageGenerationRequest`, not by an individual `PatternCharacter`.

## 8. HexColor

`HexColor` is a Value Object that represents an API-boundary HEX color in normalized form.

### Constraints

- Required
- Must use the `#RRGGBB` format
- The six digits after `#` must be hexadecimal digits
- Each RGB component must be in the range `0` through `255`

### Normalization

The internal value is case-insensitive. When converted to an external string, it uses one normalized representation.

```text
#ff69b4
↓
#FF69B4
```

### Conversion to RGB

`HexColor` can calculate RGB values, but it does not know the Glyph Forge API request format.

```text
#FF69B4
↓
R: 255
G: 105
B: 180
```

Conversion to a Glyph Forge API-specific color DTO is performed by Infrastructure.

## 9. RgbColor

`RgbColor` stores red, green, and blue components as values.

| Value | Type | Constraint |
| --- | --- | --- |
| `red` | Integer | 0 through 255 |
| `green` | Integer | 0 through 255 |
| `blue` | Integer | 0 through 255 |

`RgbColor` validates the range of each component at creation time. Negative values, values greater than 255, fractional values, and unset values cannot be created.

## 10. ImageGenerationRequest

`ImageGenerationRequest` is a validated Domain Model passed to the image generation use case.

### Attributes

| Attribute | Type | Required |
| --- | --- | :---: |
| `type` | `ImageType` | Yes |
| `text` | `RenderText` | Yes |
| `foregroundCharacter` | `PatternCharacter` | Yes |
| `foregroundColor` | `HexColor` | Yes |
| `backgroundCharacter` | `PatternCharacter` | Yes |
| `backgroundColor` | `HexColor` | Yes |

### Invariants

- All attributes are present
- All Value Object constraints are satisfied
- `foregroundCharacter` and `backgroundCharacter` do not both consist only of whitespace
- No external API endpoint or HTTP information is stored

### Creation

External input must not be handled directly as an `ImageGenerationRequest`. The Controller or input Mapper receives an HTTP DTO, creates the Value Objects, and then creates the `ImageGenerationRequest`.

If creation fails, a partially invalid Model must not be passed to the Service.

## 11. GeneratedImage

`GeneratedImage` is a Domain Result representing a successful image generation result.

### Attributes

| Attribute | Description |
| --- | --- |
| `content` | Binary image data |
| `mediaType` | Image media type |
| `fileName` | Unique filename used for downloading |

The MVP does not persist generated images, so `GeneratedImage` does not contain a storage location or database ID.

The Service generates `fileName` for each request. The format is:

```text
mojica-{imageType}-{UUID}.png
```

Example:

```text
mojica-x-icon-550e8400-e29b-41d4-a716-446655440000.png
```

`imageType` uses the normalized value of `ImageType`. User-provided input values must not be included in the filename.

Conversion from the Glyph Forge API response DTO to `GeneratedImage` is performed at the Infrastructure boundary. The Controller converts `GeneratedImage` into an HTTP response.

## 12. ModelValidationError

When Model creation or invariant validation fails, a Domain error is returned so that callers can classify the failure.

### Attributes

| Attribute | Description |
| --- | --- |
| `code` | Language-independent error code |
| `target` | Domain attribute or combination of attributes with the problem |
| `reason` | Machine-detectable failure reason represented as `ModelValidationReason` |
| `details` | Safe supplementary information, when necessary |

`ModelValidationError` does not contain Japanese or English display messages or HTTP status codes. `reason` is represented by the closed type `ModelValidationReason`.

Expected validation failures are handled as a `Result<T, ModelValidationError>`-equivalent return value, not as exceptions. Handling unexpected runtime failures is outside the Model's responsibility.

The Service or Controller converts `ModelValidationError` into the public API error contract. Display messages are resolved outside the Model based on `Accept-Language`.

## 13. Domain Error Examples

| `code` | `target` | Condition |
| --- | --- | --- |
| `REQUIRED` | Attribute name | A required value is missing |
| `LENGTH_OUT_OF_RANGE` | Attribute name | Character count is outside the allowed range |
| `CONTROL_CHARACTER` | Attribute name | Contains a control character |
| `INVALID_HEX_COLOR` | Color attribute name | Is not in `#RRGGBB` format |
| `UNSUPPORTED_IMAGE_TYPE` | `type` | Image type is not defined |
| `VISIBLE_CHARACTER_REQUIRED` | Combination of character attributes | Both values consist only of whitespace |

Conversion to the public API's `VALIDATION_ERROR` or HTTP `422 Unprocessable Entity` is the responsibility of the Controller boundary.

## 14. API DTO Mapping

API JSON DTOs and Domain Models must not be treated as the same types.

| API DTO | Domain Model |
| --- | --- |
| `type: string` | `ImageType` |
| `text: string` | `RenderText` |
| `foregroundCharacter: string` | `PatternCharacter` |
| `foregroundColor: string` | `HexColor` |
| `backgroundCharacter: string` | `PatternCharacter` |
| `backgroundColor: string` | `HexColor` |

`Accept-Language` is not passed into the Domain Model. It is handled outside the Model as execution context required to localize error messages.

## 15. Test Contract

The Model must be testable without external services or a database.

At minimum, tests must cover the following behaviors:

- A valid `ImageType` can be created
- An undefined `ImageType` is rejected
- `RenderText` required, length, whitespace, and control-character constraints are validated
- `PatternCharacter` required, length, and control-character constraints are validated
- A state where both pattern characters consist only of whitespace is rejected
- A valid `#RRGGBB` value can be created as a `HexColor`
- An invalid HEX format is rejected
- HEX-to-RGB conversion produces the correct result
- The range of each RGB component is validated
- An `ImageGenerationRequest` can be created from valid values
- An `ImageGenerationRequest` containing invalid values cannot be created

Tests verify values and validation results observable from the Model, not HTTP status codes or Glyph Forge API communication. HTTP contract and external API contract tests are performed at the Controller and Infrastructure boundaries respectively.

## 16. Decisions

There are no unresolved items. Character counting, generated image filenames, and validation failure representation follow the definitions in this document.
