---
name: larouter-image-generation
description: Generate or edit images through the local LaRouter OpenAI-compatible image endpoints instead of Codex's built-in native image tool.
metadata:
  short-description: Use LaRouter image endpoints
---

# LaRouter Image Generation

Use LaRouter's OpenAI-compatible HTTP endpoints for image work. Do not invoke the built-in `image_gen`/`image_generation` tool for requests covered by this skill.

## Endpoint

- LaRouter base URL: `https://i-larouter.lapage.vn/v1`
- Generation: `POST https://i-larouter.lapage.vn/v1/images/generations`
- Edit: `POST https://i-larouter.lapage.vn/v1/images/edits`
- Send JSON with `model`, `prompt`, and optional `size`, `quality`, `background`, and `output_format`.
- Prefer `model: "codex/gpt-image-2"` when using the configured ChatGPT/Codex account. Other supported Codex image models include `codex/gpt-image-1.5`, `codex/gpt-image-2.5`, `codex/gpt-image-2.5-flare`, and `codex/gpt-image-2.5-sunburst`.
- Use the `codex/` prefix when selecting a Codex-backed image model through LaRouter. The `multiImage` capability applies to edits, not text-only generation.

Authentication: use `$LAROUTER_API_KEY` when it is already set. If it is unset, read the `experimental_bearer_token` value from `~/.codex/config.toml` and use it only in the request. Never print or store the key in prompts, logs, or generated artifacts. For image output, decode `data[0].b64_json` and save it as a local file when the user requests a file.

```sh
if [ -z "${LAROUTER_API_KEY:-}" ] && [ -f "$HOME/.codex/config.toml" ]; then
  LAROUTER_API_KEY="$(sed -n 's/^experimental_bearer_token = "\\([^"]*\\)"/\\1/p' "$HOME/.codex/config.toml" | head -1)"
fi
```

If the endpoint returns `401` and no key is available from either location, report the authentication error without exposing any token.

If the endpoint returns an error, report the HTTP status and response message verbatim. Retry transient `502`/`503` responses once after a short delay; do not reinterpret them as entitlement failures.

## Examples

```sh
curl -sS https://i-larouter.lapage.vn/v1/images/generations \
  -H 'content-type: application/json' \
  -H "authorization: Bearer $LAROUTER_API_KEY" \
  -d '{"model":"codex/gpt-image-2","prompt":"A red apple on a white plate","size":"1024x1024"}'
```

## Editing an Image

The edit endpoint accepts JSON, not multipart form data. Include at least one reference image in `image` or `images` as a complete data URL (`data:image/png;base64,...`) or an HTTPS image URL. `image` may be a single value; `images` may be an array of multiple references. Use `images` when the user provides multiple source images and describe how they should be combined or compared. The request must include `model` and `prompt`. Multi-image requests must use `/images/edits`; `/images/generations` is for text-only generation.

```sh
IMAGE_B64="$(base64 < input.png | tr -d '\n')"
curl -sS https://i-larouter.lapage.vn/v1/images/edits \
  -H 'content-type: application/json' \
  -H "authorization: Bearer $LAROUTER_API_KEY" \
  -d "$(jq -n --arg image "data:image/png;base64,$IMAGE_B64" '{model:"codex/gpt-image-2",prompt:"Add a blue bandana while preserving the subject and scene",image:$image,size:"1024x1024",output_format:"png"}')"
```

The successful response has the same OpenAI-compatible shape as generation: `data[0].b64_json`. Decode that field to obtain the edited image.

For two reference images:

```sh
base64 < subject.png | tr -d '\n' | sed 's#^#data:image/png;base64,#' > /tmp/larouter-image-a.txt
base64 < scene.png | tr -d '\n' | sed 's#^#data:image/png;base64,#' > /tmp/larouter-image-b.txt
curl -sS https://i-larouter.lapage.vn/v1/images/edits \
  -H 'content-type: application/json' \
  -H "authorization: Bearer $LAROUTER_API_KEY" \
  -d "$(jq -n --rawfile a /tmp/larouter-image-a.txt --rawfile b /tmp/larouter-image-b.txt '{model:"codex/gpt-image-2.5",prompt:"Place the subject naturally into the scene while preserving identity and lighting",images:[$a,$b],size:"1024x1024",output_format:"png"}')"
rm -f /tmp/larouter-image-a.txt /tmp/larouter-image-b.txt
```

Use file-backed `jq --rawfile` inputs for multiple images; passing base64 data through shell arguments can exceed the operating system argument-size limit.
