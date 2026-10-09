# A2A Agent Safety ruleset

A MuleSoft API Governance ruleset (AMF Validation Profile 1.0) for **A2A v1.0 Agent Cards**:
Exchange assets with classifier `a2a-v1-card`. It checks that the card publishes
only secure ways to reach and authenticate to the agent.

**Pairs with:**
- [A2A Agent Card Conformance](https://github.com/P4A-Policies-for-Agents/a2a-agent-card-conformance-ruleset),
  for the fields A2A v1.0 marks REQUIRED.
- MuleSoft's *Agent Network Best Practices*, for Agent Network specs.

## Rules

| Rule | Severity | Why | Fix |
|---|---|---|---|
| `interface-url-https` | violation | Production MUST use encrypted transport | Serve every interface on `https://` |
| `extended-card-requires-auth` | violation | The extended card MUST be fetched with authentication | Declare `securitySchemes` and `securityRequirements` |
| `oauth-no-implicit-flow` | violation | Implicit flow leaks tokens; deprecated | Use `authorizationCode` with PKCE |
| `oauth-no-password-flow` | violation | Password flow hands passwords to clients; deprecated | Use `authorizationCode`, `clientCredentials` or `deviceCode` |
| `oauth-metadata-url-https` | violation | Metadata over HTTP lets attackers swap endpoints | Use an `https://` `oauth2MetadataUrl` |
| `oidc-url-https` | violation | Discovery over HTTP lets attackers swap keys | Set an `https://` `openIdConnectUrl` |
| `card-security-json-camel-case` | violation | snake_case hides interfaces, schemes and the extended-card flag from every rule | Use camelCase |
| `oauth-flows-json-camel-case` | violation | snake_case hides flows from the PKCE rule | Use camelCase flow names |
| `oauth-flow-urls-https` | violation | Codes, tokens and secrets travel to these endpoints | Serve `authorizationUrl`, `tokenUrl`, `refreshUrl` and `deviceAuthorizationUrl` on `https://` |
| `api-key-scheme-json-camel-case` | violation | `api_key_security_scheme` hides the key from `api-key-not-in-query` | Rename to `apiKeySecurityScheme` |
| `oidc-scheme-json-camel-case` | violation | `open_id_connect_security_scheme` hides the URL from `oidc-url-https` | Rename to `openIdConnectSecurityScheme` |
| `card-security-declared` | warning | Clients can't discover how to authenticate | Declare `securitySchemes` and `securityRequirements` |
| `oauth-pkce-required` | warning | Intercepted codes can be redeemed without PKCE | Set `pkceRequired: true` |
| `api-key-not-in-query` | warning | Query strings end up in logs | Send the key in a header |
| `card-signed` | info | Unsigned cards can be altered undetected | Add a JWS `signatures` entry |

## Run it locally or in CI

> **Not deployable to hosted Anypoint API Governance today.** For `type=agent` assets, hosted
> governance validates the `agent-metadata.json` that Exchange generates (`platform` and
> `protocol` only), not the Agent Card. None of the card fields these rules read reach the
> hosted validator, so a governance profile reports false findings or none at all. This was
> tested on 2026-10-09 with probe rulesets against `a2a-v1-card` assets. This ruleset isn't
> listed in the P4A catalog until that changes.

Validate an agent card project (a folder with the card and an `exchange.json` whose `main` is the
card and whose `classifier` is `a2a-v1-card`) with governance plugin 1.1.4 or later. See
[Development](#development) for the install.

```bash
anypoint-cli-v4 governance:api:validate path/to/agent-project --rulesets ruleset.yaml
```

## Limitations

- **Hosted governance doesn't see the card.** See [Run it locally or in CI](#run-it-locally-or-in-ci).
  Validating `a2a-v1-card` projects locally needs governance plugin 1.1.4 or later.
- **Card-level findings have no source location.** `extended-card-requires-auth`,
  `card-security-declared`, `card-signed` and `card-security-json-camel-case` are reported on
  the asset, so their messages name the field.
- **`securityRequirements` can't be checked against `securitySchemes` names.** Scheme names are
  map keys, which a validation profile can't join on.
- **Signatures are checked for presence only.** JWS verification is a runtime concern.
- **`interface-url-https` also flags local development cards.** That is intended for governed
  Exchange assets.
- **snake_case fields inside schemes and flows aren't all checked.** The wrappers and flow names
  are, but an alias such as `token_url` or `pkce_required` reads as a missing field.
- **snake_case aliases are matched through `snake*` prefixes.** Compact IRIs can't contain
  `_`, so a path such as `core.security_schemes` makes the validator panic. `ruleset.yaml`
  declares prefixes whose namespace ends with the alias's leading words (`snakeSecurity:
  http://a.ml/vocabularies/core#security_`), and the rules use paths like
  `snakeSecurity.schemes`. Each alias has its own bad fixture.
- **`validate-authoring` false errors.** It has no A2A domain, so it reports `Invalid
  targetClass` for `core.supportedInterfaces`, `core.oauth2SecurityScheme`,
  `core.apiKeySecurityScheme`, `core.openIdConnectSecurityScheme` and the two snake_case
  wrapper classes.
  `governance:ruleset:validate` accepts the ruleset, and the fixtures prove each rule.

## Test on Exchange assets

The scripts below publish the fixtures to a test business group, so you can check whether hosted
governance has started reading the card. Attach the ruleset with a draft governance profile.
Today the findings won't match the descriptions (see the note above). Copy `.env.example` to `.env`
and fill in a connected app and business group ID; `.env` is gitignored.

```bash
scripts/publish-examples.sh --dry-run   # list the assets
scripts/publish-examples.sh             # <prefix>-ok plus one <prefix>-<rule-id> per rule
scripts/publish-examples.sh --all       # also every bad variant and the scope fixtures
scripts/cleanup-examples.sh             # soft-delete them all after testing (--hard, --yes)
```

`<prefix>-ok` should give 0 findings, and each `<prefix>-<rule-id>` exactly the finding named in
its description. Publishing skips versions that already exist; to republish changed fixtures,
clean up first or set `EXAMPLES_VERSION`.

## Development

`scripts/check.sh` needs governance plugin **1.1.4 or later**. One way to install it without
touching your global CLI:

```bash
mkdir -p ~/.cache/p4a-a2a-cli && cd ~/.cache/p4a-a2a-cli
echo '{"private":true,"dependencies":{"anypoint-cli-v4-public":"1.6.27"},"overrides":{"mulesoft-anypoint-cli-governance-plugin":"1.1.4"}}' > package.json
npm install --no-audit --no-fund
```

Then, from the repo root:

```bash
python3 -I scripts/fixtures.py
ANYPOINT_CLI=~/.cache/p4a-a2a-cli/node_modules/.bin/anypoint-cli-v4 scripts/check.sh
```

- `scripts/fixtures.py` generates every fixture. Its `GOOD` card and scope documents are
  identical in both A2A ruleset repos. Never edit `fixtures/` by hand.
- Bump `version` in `exchange.json` for every rule change.

## Source Ref

A2A Protocol Specification 1.0.0, https://a2a-protocol.org/latest/specification/ (snapshot
2026-10-09).
