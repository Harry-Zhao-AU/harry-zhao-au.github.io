---
layout: post
title: "Building a personalised storybook with gpt-image-2"
date: 2026-05-20
categories: [AI, Projects]
tags: [gpt-image-2, azure, storybook, image-generation]
excerpt: "How I built a pipeline that turns a child's photo into a fully illustrated storybook — and what I learned about character consistency, Google auth, and CORS."
---

# Building a personalised storybook with gpt-image-2

My daughter loves books where she's the main character. So I built one — a pipeline that takes a real photo of a child and produces a fully illustrated children's storybook, page by page, in a consistent style.

The hard part isn't generating a good-looking image. That's table stakes now. The hard part is getting the same character to look like the same character across twelve pages of independent generations.

## The image pipeline

I used `gpt-image-2` via Azure OpenAI, which supports multi-image input — you can pass reference images alongside your prompt. The whole thing runs in three phases with a human review step between each.

**Phase 1 — Character reference sheet.**

The model receives the child's real photo and produces a reference sheet: three views of the character (front, three-quarter, side profile) on a white background, name labelled underneath. This becomes the visual ground truth for every image that follows.

**Phase 2 — Storyboard grid.**

With the character reference locked, the model generates a single 4×3 panel grid containing all story scenes at thumbnail scale. The character reference is passed as a reference input. The value isn't the tiny panels themselves — it's forcing the model to commit to consistent lighting, palette, and backgrounds before any full illustrations are generated.

**Phase 3 — Individual pages.**

Each page is generated with three reference inputs:

1. The cropped storyboard panel (composition guide)
2. The character reference sheet (likeness anchor)
3. The previous finished page, if one exists (scene continuity)

```
Page 3 of a children's storybook. [scene description]
Maintain consistent character appearance with the reference sheet.
Full illustration, no text or page numbers.
Studio Ghibli anime style, rich oil-painting-like texture with visible
brushstrokes, dramatic cinematic golden-hour lighting with deep warm shadows,
expressive Ghibli characters, vivid saturated colour palette.
```

After each generation I either approve it and move to the next panel, or give free-text feedback that gets appended to the prompt for a regeneration.

The first completed story came out as a 12-page illustrated adventure. The character is recognisably consistent across all twelve pages — same face, same hair, same outfit. Not pixel-perfect, but well within what you'd accept from a human illustrator working quickly.

**Key takeaway:** visual consistency is a workflow problem, not a prompt problem. The fix is making each new image *see* the previously approved images as explicit reference inputs, combined with a human review gate so errors don't compound across pages.

## The web reader

The finished pages are served through a web reader hosted on Azure Static Web Apps, with images sitting in Azure Blob Storage. Simple enough — but I ran into a couple of real issues getting auth and storage access right.

### Google login

Azure Static Web Apps has built-in support for Google OAuth, configured entirely in `staticwebapp.config.json`:

```json
{
  "auth": {
    "identityProviders": {
      "google": {
        "registration": {
          "clientIdSettingName": "GOOGLE_CLIENT_ID",
          "clientSecretSettingName": "GOOGLE_CLIENT_SECRET"
        }
      }
    }
  },
  "routes": [
    { "route": "/*", "allowedRoles": ["authenticated"] }
  ],
  "responseOverrides": {
    "401": { "redirect": "/.auth/login/google", "statusCode": 302 }
  }
}
```

The `/*` route requiring `authenticated` means every page, asset, and script is protected — unauthenticated requests redirect straight to Google. The client ID and secret live in the SWA app settings, never in source.

This works well. The only friction was that the Google OAuth app registration needs the exact SWA callback URL whitelisted, and that URL isn't known until after first deploy. So there's a round-trip: deploy → get the hostname → add it to the Google Cloud Console → done.

### CORS for Blob Storage

The images can't be served through the SWA itself (they're too large and generated separately), so the frontend fetches them directly from Azure Blob Storage using a SAS token appended to each URL.

This introduces two problems.

**Problem 1 — CORS.** The browser blocks cross-origin requests from the SWA domain to the storage account unless CORS is explicitly configured on the blob service. In Bicep:

```bicep
properties: {
  cors: {
    corsRules: [
      {
        allowedOrigins: ['https://<swa-hostname>', 'http://localhost:4280']
        allowedMethods: ['GET']
        allowedHeaders: ['*']
        maxAgeInSeconds: 3600
      }
    ]
  }
}
```

The catch: you can't set the SWA hostname in CORS until you know it, and you only know it after first deploy. The fix is a two-step deploy — first deploy with only `localhost` in CORS, note the hostname, then redeploy with the real hostname added. A minor operational nuisance, but worth knowing before you deploy to production.

**Problem 2 — SAS token exposure.** The SAS token lives in a JavaScript config file served to the browser. Anyone who is authenticated and loads the page can read the token from the source and use it to access the storage directly, bypassing the Google auth entirely.

I accepted this tradeoff but mitigated it:

- The storage account has `allowBlobPublicAccess: false` — the SAS is required for every request, no guessing raw URLs
- The SAS is read-only (GET only) and CORS is locked to the SWA domain, so a leaked token can read stories but can't write or delete anything, and can't be used from arbitrary origins
- Short SAS expiry, rotated via a GitHub Actions secret

For a storybook app this risk profile is acceptable. For anything with more sensitive content, the right fix is a server-side proxy that validates the SWA auth token before issuing short-lived SAS tokens — but that's more infrastructure than this project needed.

## What's next

The image pipeline is solid and reusable for any story. Next I want to look at generating game assets the same way — consistent characters and environments for a 2D game. That's a different challenge because game assets need to tile, animate, and maintain scale consistency across a lot more variations than a storybook does. More on that in a follow-up post.
