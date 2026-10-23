# magpie-newapi

A [magpie](https://usemagpie.ai) provider plugin for [New API](https://github.com/QuantumNous/new-api) (and the One API family).

Add a relay site's **Base URL + API key** as one magpie account; several accounts pool into one model family that magpie routes and fails over across. Each account can also carry an optional **system access token** to read its wallet balance and subscription remainder, shown on the account row, in `magpie quota`, and at the gateway's `GET /v1/magpie/quotas`.

## Features

- **One account = one complete instance** — `Base URL + API key`, plus an optional **system access token** used only for balance.
- **Many instances, one model pool** — sign in many times under the one provider; each account keeps its own `baseURL + apiKey`; they jointly serve the same models. A 401 marks an account for re-login, a 429/5xx fails over to the next.
- **Balance / subscription (optional)** — shown when a system access token is saved; otherwise it only serves models.
- **Group by site (optional)** — list sites in the plugin's options; each becomes its own provider, and can be limited to a subset of models.

## Icon

The icon is `assets/logo.png` (180×180 PNG), embedded as a `data:image/png;base64,…` URI in `package.json`'s `magpie.icon`. magpie's `magpie.icon` accepts only an `https://` address or a `data:image/…` URI — not a local path — so embedding keeps it local and offline while still readable by magpie.

To replace it:

```bash
cp new-icon.png ~/Code/magpie-newapi/assets/logo.png
cd ~/Code/magpie-newapi
python3 - <<'EOF'
import base64, json
uri = "data:image/png;base64," + base64.b64encode(open("assets/logo.png","rb").read()).decode()
d = json.load(open("package.json")); d["magpie"]["icon"] = uri
json.dump(d, open("package.json","w"), indent=2, ensure_ascii=False); open("package.json","a").write("\n")
EOF
```

Then toggle the plugin off and on in the app.

## Install

```bash
magpie plugin add @jingyi0605/magpie-newapi        # from npm
magpie plugin add github:jingyi0605/magpie-newapi  # from GitHub
magpie plugin add ~/Code/magpie-newapi             # from a folder (loaded in place)
```

## Add an account (one instance in one go)

One method, **New API API key**, asking in this order:

| # | Prompt | What to fill |
| --- | --- | --- |
| 1 | New API address | the site, e.g. `https://api.example.com` (with or without `/v1`) |
| 2 | API key | this instance's relay **`sk-…`** key |
| 3 | System access token | **optional**; needed to show balance / subscription |
| 4 | User id | only for a **legacy** access token |

So an account is `baseURL` + `API Key` (+ optional access token). Add a second instance by adding another account under New API, with another `baseURL + API key` (another key on the same site, or another site).

**Validated at sign-in**: the key is checked against `/v1/models`, the token against `/api/user/self`; a wrong credential fails immediately instead of saving a broken account.

```bash
printf 'https://site\nsk-yourkey\n\n\n' | magpie plugin login newapi            # models only
printf 'https://site\nsk-yourkey\nnap_yourtoken\n\n' | magpie plugin login newapi  # + balance
```

> It is an oauth method with an auto callback rather than an api method: magpie's api method always asks its key box *after* the prompts, so the natural order isn't possible there; oauth has no key box, and the callback validates and returns the key.

## What balance shows

- Balance: `/api/user/self` `quota` ÷ `/api/status` `quota_per_unit`, prefixed with the site's currency symbol (`¤` shown as `$`).
- Subscriptions: active rows from `/api/subscription/self`, each a window with used/total, percent, and next reset; titles from `/api/subscription/plans`.
- **Balance-only sites show only the balance, with no quota bar**: a wallet has no cap, and drawing it as a window makes magpie show a fake "100% left". A window is created only when the site reports a real subscription amount (`amount_total`).

## One record or many? Naming and model combinations

Each row on magpie's Providers page is one provider id. **Its display name and the subset of models it offers are stored per provider by magpie itself** (`providers.json`: `name`, `models`, `pickedFrom`), so you can rename a row, tick which models it exposes, and use "Duplicate" to make another row with a different subset.

The plugin only decides **how many provider records exist**:

- **One (default, no options)** — a single `newapi`; every sign-in is an account under it, models are the union.
- **Many (`sites`)** — each site is its own record, with its own name and model filter:

```json
{
  "sites": [
    { "id": "claude-pool", "name": "Claude Pool", "models": ["claude-*"] },
    { "id": "codex-pool",  "name": "Codex Pool",  "models": ["gpt-*", "codex-*"] },
    { "id": "gap-claude",  "name": "GAP-Claude",  "url": "https://api.example.com", "models": ["claude-*"] }
  ]
}
```

```bash
magpie plugin options magpie-newapi '{"sites":[...]}'
magpie plugin options magpie-newapi off      # back to the single record
```

| field | meaning |
| --- | --- |
| `id` | provider id (lower-case letters/digits/`-`/`_`, ≤40); also the `<id>/<model>` prefix |
| `name` | the row's name (any language); defaults to `id` |
| `url` | the site address. **Optional** — without it the provider only defines a name + model combination, and each account supplies its own address at sign-in |
| `models` | only expose matching models: exact id, `claude-*` prefix, or `/regex/` |
| `exclude` | hide matching models (wins over `models`) |

The plugin names the row: the default provider is **New API**, and `sites[].name` names the others (any language). To rename the single default provider, declare one site — even a single entry.

## Protocols

The **plugin** picks the endpoint (magpie uses the `api.npm` / `api.url` each model carries). Precedence: the relay's own `supported_endpoint_types` from `GET /v1/models` (`openai`, `openai-response`, `anthropic`, `gemini`, …), then the model name.

| endpoint | AI SDK package | path |
| --- | --- | --- |
| `anthropic` | `@ai-sdk/anthropic` | `/v1/messages` |
| `openai-response` | `@ai-sdk/openai` | `/v1/responses` |
| `openai` | `@ai-sdk/openai-compatible` | `/v1/chat/completions` |

Models that only list non-chat endpoints (`embeddings`, `image-generation`, …) are dropped.

[models.dev](https://models.dev) is used **only** for metadata (context/output limits, reasoning flag, effort levels, interleaved); cached daily, never blocks the list, and does not decide protocols.

## Where credentials live

- The relay key, the access token and any cached dashboard token live only in magpie's `plugin-auth.json` (mode 600), the same place magpie keeps its own sign-ins.
- Nothing is written elsewhere; the models.dev cache goes to `~/.cache/magpie-newapi/`.

## Tests

```bash
npm test        # unit tests against a built-in mock New API
```

Try it in a sandbox:

```bash
sb=$(mktemp -d)
m() { env HOME=$sb XDG_CONFIG_HOME=$sb/.config XDG_CACHE_HOME=$sb/.cache MAGPIE_ADDR=127.0.0.1:3499 magpie "$@"; }
m plugin add ~/Code/magpie-newapi </dev/null
m plugin login newapi
m plugin
m quota
m provider test newapi <model>
```

## Known limits

- Balance / subscription can only be read with a **system access token** (Personal Settings → Security Settings); a relay key cannot call the console API.
- Models that only list non-chat endpoints (image / embeddings) are dropped.
- A legacy access token needs the `user id` prompt filled; a new `nap_…` token does not.

## License

MIT
