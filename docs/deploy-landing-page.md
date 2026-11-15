# Deploying the landing page

The landing page is a static Vite site in `site/`. No tracking, no cookies, no backend.

## Local checks

```sh
cd site
npm ci
npm run build    # output in site/dist
```

## Vercel settings

`site/vercel.json` holds the build settings, cache headers and security headers. In the Vercel project set **Root Directory** to `site`; Vercel then picks up the framework (Vite), `npm run build` and `dist` from that file. Every pull request gets a preview URL; the production branch is `main`.

## Dani: manual steps

1. In Vercel, create a new project and import `kupferco/openpocket`.
2. Set **Root Directory** to `site`. Leave the rest as detected (Vite).
3. Set the production branch to `main` and deploy.
4. Optional: add a custom domain under Project Settings, Domains.
5. When the demo video is recorded, replace the placeholder block in `site/index.html` (the `<figure class="video">`) with a `<video>` tag or an embed.
