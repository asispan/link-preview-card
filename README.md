# Building a Link Unfurl / Link Preview UI Card with Zero Client-Side Processing
<img width="814" height="242" alt="web-link-card-view" src="https://github.com/user-attachments/assets/003e22ef-292b-40b0-833b-72d4bcdf5f63" />


## The Challenge

We wanted **rich link preview cards** (image, title, description, domain) when authors drop URLs into blog posts and project pages—similar to what link-preview APIs or link unfurl services provide. The catch: we didn’t want to depend on a third-party service at runtime, and we didn’t want any client-side JavaScript to fetch or render previews. Every card had to be **fast, self-hosted, and fully static**. In production, these cards appear across the [blog](https://asispanda23.com/blog) and [projects](https://asispanda23.com/projects) sections of asispanda23.com.


## The Goal

Although services for link-preview exist like [JSONLink](https://jsonlink.io), we wanted:

- **Author-time experience**: In the CMS (Tina), paste a URL → fetch image, title, and body (description) once → store everything in content.
- **Zero client-side processing**: The live site serves **pre-rendered HTML**. No runtime unfurl, no iframes, no external preview API calls. True Astro fashion: work done at build (or edit) time, not in the browser.
- **Load time and effort**: Save load time by avoiding extra network requests and script execution; save engineering effort by owning the pipeline end-to-end and keeping the stack simple.
- **Zero begging for free API Credits**: Save your dignity by not needing to pay for paid api credits nor begging jsonlink.io for free api credits both are fairly viable options if above two are not mission critical.


## Why Not JSONLink (or similar) at Runtime?

Services like JSONLink.io return metadata for a URL via an API call. If we called them from the client for each link, we’d add latency, depend on their availability, and still need JS to render the card. If we called them at build time only, we’d still depend on an external service and rate limits. By building our own **edit-time and build-time** unfurl and storing the result in the CMS and in the built site, we get:

- No runtime dependency on any link-preview API  
- No client-side JS for previews  
- Full control over meta extraction, image storage, and fallbacks  
- Preview data and images stored in our repo and served from our own origin  

---

## Architecture Overview

We split the system into two paths that both feed the same output: a **pure Astro component** that renders static HTML.

### Path 1: CMS edit time (Tina + Link Preview block)

1. Author inserts a **Link Preview** block in the rich-text body and pastes a URL.
2. In the Tina custom field (`LinkPreviewField`), the author clicks **“Fetch Preview.”**
3. The field calls:
   - **`POST /api/unfurl`** — fetches the URL, parses the first chunk of HTML, extracts `og:title`, `og:description`, `og:image`, favicon, and domain.
   - **`POST /api/save-preview-image`** — downloads the OG image and saves it under `public/uploads/link-previews/{slug}.{ext}`.
4. The returned title, description, image path, favicon, and domain are written into the block’s fields and persisted in the MDX body (e.g. as `<LinkPreview url="..." title="..." description="..." image="..." />`).
5. On build, Astro reads that content and renders **`LinkPreviewCard.astro`** with the stored props. No fetch at request time; no script in the browser.

### Path 2: Build time (bare URLs)

For content that contains a **bare URL** (e.g. a paragraph that is only a link) without an existing Link Preview block, we still want a card. During static generation, we:

1. Parse the body and detect “link preview” blocks that have a URL but no stored title/image/domain.
2. Call the same **`unfurlUrl()`** from `src/lib/unfurl.ts` and, if an image is present, **`savePreviewImageToDisk()`** to write the image under `public/uploads/link-previews/`.
3. Attach the resolved title, description, image path, favicon, and domain to the block.
4. Render the same **`LinkPreviewCard.astro`** with that data.

So both paths end in **stored data + static HTML**. The only difference is *when* we run unfurl: once in the CMS when the author clicks “Fetch Preview,” or once at build time for bare URLs.

---

## Implementation Highlights

### 1. Shared unfurl logic (`astro/src/lib/unfurl.ts`)

- **`unfurlUrl(url)`**: Fetches the URL with a browser-like `User-Agent`, reads the first ~100KB of HTML (streaming), and parses it with regex for:
  - `og:title`, `og:description`, `og:image`
  - Fallbacks: `<title>`, `<meta name="description">`
  - Favicon (`<link rel="icon">`) and domain from the URL
- **`savePreviewImageToDisk(imageUrl, slug, fs, path)`**: Build-time only; downloads the image and writes to `public/uploads/link-previews/{slug}.{ext}`. Returns the public path so we can store it in content or in-memory block data.

Used by:

- **Edit time**: `POST /api/unfurl` (unfurl only); `POST /api/save-preview-image` (image only, used by the Tina field after unfurl).
- **Build time**: Direct calls to `unfurlUrl()` and `savePreviewImageToDisk()` in `[slug].astro` for bare-URL link preview blocks.

### 2. Author-time API routes

- **`astro/src/pages/api/unfurl.ts`**: Accepts `POST { "url": "https://..." }`, calls `unfurlUrl(url)`, returns JSON `{ title, description, image, favicon, domain }`. Used only in the Tina admin when the author clicks “Fetch Preview.”
- **`astro/src/pages/api/save-preview-image.ts`**: Accepts `POST { "imageUrl", "slug" }`, downloads the image, saves to `public/uploads/link-previews/{slug}.{ext}`, returns `{ path: "/uploads/link-previews/..." }`. Used by the Tina field so the stored block references a local image path.

Both routes are server-only (`prerender = false`). They are not called from the static site at all.

### 3. Tina schema and custom field

- **Link Preview** is a rich-text template in Tina (`astro/tina/config.ts`) with fields: `url`, `title`, `description`, `image`, `favicon`, `domain`.
- **`LinkPreviewField`** (`astro/tina/components/LinkPreviewField.tsx`): React component that provides a URL input, a “Fetch Preview” button, and a live card preview. On fetch it calls `/api/unfurl`, then `/api/save-preview-image` for the image, then `input.onChange(newValue)` so Tina persists the full object into the MDX body.
- Authors see the card in the editor; the saved content is just structured data (and the local image path). No runtime unfurl or third-party API in production.

### 4. Output: zero-JS card (`astro/src/components/LinkPreviewCard.astro`)

- **Pure Astro component** (no `client:*`). Receives `url`, `title`, `description`, `image`, `favicon`, `domain` as props.
- Renders a single `<a>` with:
  - Left: preview image (or favicon/placeholder if no image).
  - Right: title, description, domain + favicon and arrow.
- All styling is scoped CSS in the component. No hydration, no scripts. The page is static HTML; no “loading” state or client-side fetch.

### 5. Build-time unfurl for bare URLs

In **`astro/src/pages/blog/[slug].astro`** and **`astro/src/pages/projects/[slug].astro`**:

- After parsing the body into blocks, we loop over blocks of type `linkpreview` that have `previewUrl` but no stored `previewTitle` / `previewImage` / etc.
- We call `unfurlUrl(block.previewUrl)` and, if `meta.image` exists, `savePreviewImageToDisk(meta.image, meta.title || block.previewUrl, fs, path)`.
- We set `block.previewTitle`, `block.previewDescription`, `block.previewImage`, `block.previewFavicon`, `block.previewDomain`.
- Later, when rendering blocks, we pass these into `<LinkPreviewCard ... />`. So even “bare” URLs get a full card with no client-side work.

---

## Results

| Aspect | Outcome |
|--------|--------|
| **Runtime behavior** | No client-side processing. Cards are plain HTML and CSS. |
| **Third-party at request time** | None. No external link-preview API is used in production. |
| **Data source** | Title, description, and image are fetched at **edit time** or **build time** and stored in content or block data. |
| **Images** | Served from our origin (`/uploads/link-previews/`). No hotlinking, no dependency on the target site’s CDN at view time. |
| **Load time** | No extra round-trips or script execution for previews; first paint and LCP are not delayed by link-preview logic. |
| **Effort** | One shared unfurl lib, two small API routes (edit-time only), one Tina field, one Astro component, and a small build-time loop. No ongoing cost for external API or client-side complexity. |

---

## File Reference

| Role | Path |
|------|------|
| Unfurl + save image (shared) | `astro/src/lib/unfurl.ts` |
| Edit-time unfurl API | `astro/src/pages/api/unfurl.ts` |
| Edit-time image save API | `astro/src/pages/api/save-preview-image.ts` |
| Tina Link Preview template | `astro/tina/config.ts` (LinkPreview template + body templates) |
| Tina custom field (URL + Fetch Preview + preview) | `astro/tina/components/LinkPreviewField.tsx` |
| Static card component | `astro/src/components/LinkPreviewCard.astro` |
| Build-time unfurl (blog) | `astro/src/pages/blog/[slug].astro` (loop over `contentBlocks`, unfurl bare-URL linkpreview blocks) |
| Build-time unfurl (projects) | `astro/src/pages/projects/[slug].astro` (same pattern) |

---

## Summary

We built an in-house link unfurl and link-preview UI card system that:

- Fetches **image, title, and body (description)** at **CMS edit time** (or at build time for bare URLs) and stores them in content and on disk.
- Renders cards in **true Astro fashion**: **zero client-side processing**—no runtime unfurl, no third-party preview API, no JavaScript for the card. Only static HTML and CSS.
- **Saves load time** by avoiding extra requests and script, and **saves effort** by keeping the pipeline simple, self-contained, and fully under our control.
