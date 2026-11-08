# tylers-2026.github.io

Public GitHub Pages host for the Parker Tesla Fleet API developer app. It publishes three setup addresses and the application public key. It does not sign in to Tesla, exchange OAuth codes, pair a virtual key, bill an account, or send vehicle commands.

## Public setup values

Register these strings exactly.

| Tesla field | Value |
| --- | --- |
| Allowed Origin | `https://tylers-2026.github.io` |
| Allowed Redirect URI | `https://tylers-2026.github.io/callback.html` |
| Public key URL | `https://tylers-2026.github.io/.well-known/appspecific/com.tesla.3p.public-key.pem` |

Allowed Origin is the browser origin: scheme and host, with no path and no trailing slash. A browser on this site sends `Origin: https://tylers-2026.github.io`.

Allowed Redirect URI is the full callback URL. Tesla requires the redirect URI on the authorize request and the redirect URI on the token request to match each other and the value registered for the app. Copy it with no trailing slash, query, or fragment.

The public-key host must stay `tylers-2026.github.io`. Tesla requires a PEM-encoded prime256v1 (P-256 / secp256r1) public key to remain available at `/.well-known/appspecific/com.tesla.3p.public-key.pem` on the application domain, and that domain must share its root with an allowed origin.

## Files

- `index.html` is the landing page at `/`.
- `callback.html` reserves the redirect path.
- `.well-known/appspecific/com.tesla.3p.public-key.pem` is the supplied public key, copied byte for byte. No key pair was generated here, and no private key is in this repository.
- `.nojekyll` turns off Jekyll. GitHub Pages otherwise uses Jekyll, and Jekyll omits directories whose names start with a dot, which would hide the public key. With `.nojekyll`, the key file is published as a static file.

Expected SHA-256 of the public key file:

`ba1a22ba0dfd9129fadc3c7d357627f878127352e017ca32ce1715d6b7761f22`

The inspected key is an EC public key on prime256v1, 256 bits. Confirm the deployed body still matches this digest before anyone registers the domain. Legacy GitHub Pages does not provide a way to set a custom `Content-Type` for `.pem`; Tesla's documented check is that the PEM body is publicly readable over HTTPS.

## Limits of the static callback

`callback.html` only reserves the redirect address so it can be entered in the Tesla app settings. It is not a functioning OAuth backend.

The page is self-contained. It sets a no-referrer policy, allows no external resources, forms, or third-party scripts, and does not read the query string into the page. A same-origin inline script replaces the visible address with `/callback.html`, which drops a query or fragment from the address bar. The HTML never prints those parameters and never forwards them.

That is not OAuth completion. An authorization code in a redirect is delivered as the query string of the request the browser makes to GitHub Pages. This repository cannot stop that request, cannot keep the code out of GitHub's request logs, and cannot exchange it. The page then discards it. Treat a callback URL that already contains `code`, `state`, or a token as sensitive and keep it out of ordinary chat.

Do not send a person through Tesla's authorize endpoint against this redirect. The code would arrive at a page that throws it away.

## What a later secure completion needs

Tesla's published third-party authorization-code flow needs a confidential server step. Build that on infrastructure you control before any real sign-in. This static site cannot hold it.

1. **State.** Before redirecting to `https://auth.tesla.com/oauth2/v3/authorize`, create a high-entropy random `state`, store it server-side (or in an encrypted cookie the server can read), and send it on the authorize URL. When the browser returns, continue only if the returned `state` matches the stored value, then discard that stored value. Tesla documents `state` as required.
2. **PKCE.** Also create a high-entropy `code_verifier`, store it in the same server-side session, and send only its S256 `code_challenge` on the authorize request. Send the verifier later, with the code exchange, and never put it in this public site. Tesla's published third-party token parameter list does not currently document PKCE fields. Confirm that the authorize and token endpoints accept `code_challenge` / `code_verifier` when the backend is built, and keep the state check and the confidential code exchange either way.
3. **Code exchange.** From the server, `POST` the authorization code to `https://fleet-auth.prd.vn.cloud.tesla.com/oauth2/v3/token` with `grant_type=authorization_code`, the client id, the client secret, the audience, and the same redirect URI registered above. Tesla documents the client secret as required on that call. The secret, access token, and refresh token must stay on the server. They must never be committed, embedded in these pages, or placed in a public log.
4. **Redirect ownership.** Point the registered redirect URI at that server (or change this path so the server receives the request) before starting the flow. A static GitHub Pages file cannot meet steps 1–3.

Partner-account registration, billing, virtual-key pairing, and vehicle commands are separate later steps. They are intentionally not performed from this repository. Pairing still requires the private key, which is not here, and Tesla's register endpoint, which this site does not call.

## Checks

From a checkout of this branch:

```sh
sha256sum .well-known/appspecific/com.tesla.3p.public-key.pem
openssl pkey -pubin -in .well-known/appspecific/com.tesla.3p.public-key.pem -noout -text
python3 -m http.server 8765
```

Then request `/`, `/callback.html`, and `/.well-known/appspecific/com.tesla.3p.public-key.pem`. The key response body must match the digest above. The callback response body must not echo query parameters.

After this branch is merged to `main`, repeat the same requests against `https://tylers-2026.github.io`. GitHub Pages is already enabled for this public repository (legacy build from `main` `/`, HTTPS enforced, no custom domain). A merge publishes the files. Opening a pull request does not.

## Publishing notes

The origin, redirect path, and public key URL are public on purpose. Secrets are not. Keep the private key, client secret, authorization codes, refresh tokens, and any callback URL that already carries those values off this site and out of git history.
