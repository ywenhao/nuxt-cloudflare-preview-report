# Pure Nitro 2 Cloudflare preview warning reproduction

This repository reproduces the Cloudflare preview import warnings originally reported in [nuxt/nuxt#36530](https://github.com/nuxt/nuxt/issues/36530), without Nuxt, Vue, or any application modules.

It is based on the [Nitro 2 hello-world reproduction sandbox](https://stackblitz.com/github/nitrojs/nitro/tree/v2/examples/hello-world) suggested by the Nuxt maintainer.

[Open this reproduction in StackBlitz](https://stackblitz.com/github/ywenhao/nuxt-cloudflare-preview-report?file=nitro.config.ts)

## Reproduce

```sh
npm ci
npm run build
npm run preview
```

Wrangler preview requires the native workerd runtime. If your StackBlitz environment cannot execute it, clone this repository and run the commands locally. No Cloudflare account, login, or deployment is needed.

StackBlitz starts with `npm run build`, so the generated Worker can be inspected without automatically starting the native runtime.

The build targets `cloudflare_module`, so the preview uses `wrangler dev`, not `node .output/server/index.mjs`.

Visit `http://localhost:8787/`. The route returns:

```text
Pure Nitro Cloudflare preview warning reproduction.
```

The build and route work, but Wrangler emits two `ignored-bare-import` warnings for generated bare `node:process` imports in `.output/server/index.mjs` and `.output/server/chunks/routes/index.mjs`. The application source does not import `node:process`.

```text
▲ [WARNING] Ignoring this import because "node_modules/@cloudflare/unenv-preset/dist/runtime/node/process.mjs" was marked as having no side effects [ignored-bare-import]

    .output/server/index.mjs:1:114:
      1 │ ...s/nitro/nitro.mjs";import"node:process";import"cloudflare:worker...
        ╵                             ~~~~~~~~~~~~~~

  It was excluded from the "sideEffects" array in the enclosing "package.json" file:

    node_modules/@cloudflare/unenv-preset/package.json:27:2:
      27 │   "sideEffects": [
         ╵   ~~~~~~~~~~~~~
```

A `Proxy environment variables detected` notice, if present, is environment-specific and unrelated to the reproduced import warnings.

## Verified environment

- Node.js: `24.21.0`
- npm: `12.2.0`
- Nitro: `2.13.4`
- Wrangler: `4.149.0`
- Rollup: `4.64.3`
- `@cloudflare/unenv-preset`: `2.16.2`
- workerd: `1.20261006.1`

The fresh dependency installation, build, and local preview were verified. The `/` route returns HTTP 200 while the two warnings appear during startup.

## Configuration

```ts
export default defineNitroConfig({
  compatibilityDate: "2025-07-15",
  srcDir: "server",
  preset: "cloudflare_module",
});
```

Wrangler enables `nodejs_compat` and reads `.output/server/index.mjs`. The dependency versions are pinned to Nitro `2.13.4` and Wrangler `4.149.0`; `package-lock.json` pins their transitive dependencies.

## Expected result

A minimal Nitro application using the Cloudflare preset should preview without warnings caused by redundant framework-generated imports. Genuine application or Wrangler warnings should remain visible.

## Workaround to compare

Add this setting to `nitro.config.ts`, then rebuild and preview:

```ts
rollupConfig: {
  output: {
    hoistTransitiveImports: false,
  },
},
```

The committed configuration intentionally does **not** enable the workaround so that the problem remains reproducible.

The comparison build was also verified: disabling import hoisting removes both warnings, preserves the actual process compatibility import, and still returns HTTP 200.

## Related reports

- Original Nuxt report: https://github.com/nuxt/nuxt/issues/36530
- Related Nitro 3 handling: https://github.com/nitrojs/nitro/pull/4164
- Similar Wrangler warning reports: https://github.com/cloudflare/workers-sdk/issues/8619
