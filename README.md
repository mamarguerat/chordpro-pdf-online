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

`corsproxy.io` no longer serves anonymous legacy URLs and now requires an API key,
so it is only used when one is configured. To enable it, set `VITE_CORSPROXY_API_KEY`
in a `.env` file (or in the build environment):

```
VITE_CORSPROXY_API_KEY=your-corsproxy-api-key
```

Keys are issued at <https://console.corsproxy.io/>. Note that the value is embedded in
the built static bundle, so only use a key you are willing to expose publicly.

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
