# Bloom Voting Desk

A responsive Nuxt page for saving voting-category URLs and opening them in an embedded viewer. Shortcuts are stored in the current browser only; they are not synced between devices.

## Local development

Requires Node.js 20 or newer.

```sh
npm install
npm run dev
```

Open the local URL printed by Nuxt. Run `npm run build` to create a production build.

## Deploy to Vercel

Import this repository into Vercel and deploy it as a Nuxt project. Vercel detects Nuxt and configures the Nitro deployment automatically; no custom preset is required.

## Embedding limitations

Some websites prevent being displayed in an iframe with their security headers. When a category does that, use **Open in new tab** in the viewer. The shortcut manager does not change or automate the voting website itself.
