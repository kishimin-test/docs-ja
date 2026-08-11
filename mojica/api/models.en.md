# mojica API Model Design

## 1. Purpose

This document defines the values and invariants required for image generation as Domain Models.

HTTP request/response DTOs and communication models for external services are not included in the Model.

## 2. Model Responsibilities

The Model is responsible for:

- Representing values required for image generation
- Validating value formats and ranges
- Representing image types
- Representing and converting HEX and RGB color values
- Maintaining invariants for the complete image generation request
- Representing validation results produced by the Domain

## 3. Domain Model List

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

## 4. ImageType

The type of image to generate is represented by a predefined value rather than an arbitrary string.

| Value | Description |
| --- | --- |
| `standard` | Standard image |
| `x-background` | X background image |
| `x-icon` | X icon image |

`ImageType` represents only an image-type value and does not contain information about an external system.

Undefined values cannot be created as an `ImageType`.

## 5. RenderText

`RenderText` represents the text to be rendered as character art.

### Constraints

- Required
- At least 1 character
- At most 64 characters
- Must not consist only of whitespace
- Must not contain control characters

### Creation Rules

Values that do not satisfy the constraints are rejected at creation time. A created value always satisfies the constraints.

Character counts use Unicode grapheme clusters. Emoji and combining characters are treated as one character as perceived by the user. A character represented by a surrogate pair must not be counted as two characters.

## 6. PatternCharacter

`PatternCharacter` represents the string that forms the foreground or background pattern.

Character counting follows the same Unicode grapheme-cluster rule as `RenderText`.

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

At least one of them must contain at least one visible character. This cross-field constraint is validated by `ImageGenerationRequest`.

## 7. HexColor

`HexColor` is a Value Object that represents a normalized HEX color.

### Constraints

- Required
- Must use the `#RRGGBB` format
- The six digits after `#` must be hexadecimal digits
- Each RGB component must be in the range `0` through `255`

### Normalization

The internal value is case-insensitive. String representation uses one normalized form.

```text
#ff69b4
↓
#FF69B4
```

### Conversion to RGB

`HexColor` can calculate RGB values.

```text
#FF69B4
↓
R: 255
G: 105
B: 180
```

## 8. RgbColor

`RgbColor` stores red, green, and blue components as values.

| Value | Type | Constraint |
| --- | --- | --- |
| `red` | Integer | 0 through 255 |
| `green` | Integer | 0 through 255 |
| `blue` | Integer | 0 through 255 |

Each component is validated at creation time. Negative values, values greater than 255, fractional values, and unset values cannot be created.

## 9. ImageGenerationRequest

`ImageGenerationRequest` is a validated Domain Model for the image generation use case.

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

### Creation

Create each Value Object before creating the `ImageGenerationRequest`. If creation fails, the `ImageGenerationRequest` is not created.

## 10. GeneratedImage

`GeneratedImage` is a Domain Result representing a successful image generation result.

| Attribute | Description |
| --- | --- |
| `content` | Binary image data |
| `mediaType` | Image media type |
| `fileName` | Filename used for downloading |

## 11. ModelValidationError

When Model creation or invariant validation fails, a Domain error is returned so callers can classify the failure.

| Attribute | Description |
| --- | --- |
| `code` | Language-independent error code |
| `target` | Domain attribute or combination of attributes with the problem |
| `reason` | Machine-detectable failure reason represented as `ModelValidationReason` |
| `details` | Safe supplementary information, when necessary |

`ModelValidationError` does not contain display messages. `reason` is represented by the closed type `ModelValidationReason`.

Expected validation failures are handled as a `Result<T, ModelValidationError>`-equivalent return value, not as exceptions. Unexpected runtime failures are outside the Model's responsibility.

## 12. Domain Error Examples

| `code` | `target` | Condition |
| --- | --- | --- |
| `REQUIRED` | Attribute name | A required value is missing |
| `LENGTH_OUT_OF_RANGE` | Attribute name | Character count is outside the allowed range |
| `CONTROL_CHARACTER` | Attribute name | Contains a control character |
| `INVALID_HEX_COLOR` | Color attribute name | Is not in `#RRGGBB` format |
| `UNSUPPORTED_IMAGE_TYPE` | `type` | Image type is not defined |
| `VISIBLE_CHARACTER_REQUIRED` | Combination of character attributes | Both values consist only of whitespace |

## 13. Test Contract

The Model must be testable without external services or a database.

At minimum, tests must cover:

- Creating valid and rejecting undefined `ImageType` values
- `RenderText` required, length, whitespace, and control-character constraints
- `PatternCharacter` required, length, and control-character constraints
- Rejecting the state where both pattern characters consist only of whitespace
- Creating valid `HexColor` values and rejecting invalid HEX formats
- Correct HEX-to-RGB conversion
- RGB component range validation
- Creating valid `ImageGenerationRequest` values
- Rejecting invalid `ImageGenerationRequest` values

## 14. Decisions

There are no unresolved items. Character counting, generated image filename representation, and validation failure representation follow the definitions in this document.
