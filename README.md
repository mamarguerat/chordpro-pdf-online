# chordpro-pdf-online
A static website to convert ChordPro chord sheets to PDF 

## Import workflows

- Google Drive import by query param: `?googleDriveFileId=<file-id>`
- JEM import by song number query param: `?jemNumber=959`

JEM import URL pattern used by the app:

`https://jemaf.fr/ressources/chordPro/jem<NUMBER>.chordpro`

If a JEM import fails (including potential CORS restrictions), the app shows an error and does not auto-fallback to sample content.

The app tries a direct request first, then falls back to public CORS proxies
(`api.allorigins.win`, `api.codetabs.com`). Responses that are not valid ChordPro
(HTTP errors, JSON or HTML error pages returned by a proxy) are skipped instead of
being imported as song content.

`corsproxy.io` no longer serves the anonymous legacy `?<url>` form. Its modern keyless
`?url=<url>` form is still used as a last resort — it is accepted from `github.io`
origins, which covers this GitHub Pages deployment, but it is rate limited.

An API key is optional and only raises those limits (plus dashboard analytics). To use
one, get it from <https://console.corsproxy.io/> and expose it to the build as
`VITE_CORSPROXY_API_KEY`:

- locally, in a `.env` file: `VITE_CORSPROXY_API_KEY=your-corsproxy-api-key`
- for the GitHub Pages deploy, as a repository secret of the same name
  (Settings → Secrets and variables → Actions → New repository secret). The
  `Deploy App` workflow already passes it to `npm run build`, and builds fine when
  the secret is absent.

Vite inlines `VITE_*` values into the built bundle, so this key is publicly readable on
the deployed site. Restrict it by origin in the corsproxy console and treat it as a
quota token, not a secret.

## Heading textboxes

The settings panel extracts and allows editing these ChordPro directives:

- `title`
- `artist`
- `subtitle`
- `key`
- `time`
- `tempo`
- `copyright`
- `footer`

Edits in the textboxes are synced back to the ChordPro input by updating directive lines.

## Recommended IDE Setup

[VSCode](https://code.visualstudio.com/) + [Volar](https://marketplace.visualstudio.com/items?itemName=Vue.volar) (and disable Vetur).

## Customize configuration

See [Vite Configuration Reference](https://vitejs.dev/config/).

## Project Setup

```sh
npm install
```

### Compile and Hot-Reload for Development

```sh
npm run dev
```

### Compile and Minify for Production

```sh
npm run build
```

### Run Unit Tests with [Vitest](https://vitest.dev/)

```sh
npm run test:unit
```

### Lint with [ESLint](https://eslint.org/)

```sh
npm run lint
```
