# Samya Traveler — travel blog

Source for **https://samya-traveler.github.io/**. It's built with [Hugo](https://gohugo.io/) and the
[PaperMod](https://github.com/adityatelange/hugo-PaperMod) theme. GitHub Actions builds and deploys it
every time you push to `main`.

## Why Hugo + PaperMod

The reference blog ([bestsamina/blog](https://github.com/bestsamina/blog)) is also a **Hugo** site. It
uses the `personal-web` theme through git submodules and deploys with a manual `deploy.sh`, which
pushes `public/` into a second repo. This setup keeps Hugo but changes the rest:

| Need | How this setup covers it |
|---|---|
| Markdown authoring | Each post is one folder: `index.md` plus its photos. GitHub-style callouts (`> [!TIP]`) also render in GitHub's preview. |
| Rich image layouts | Hugo resizes images at build time and outputs WebP. Shortcodes for a full-width hero, a photo grid gallery and a captioned figure. |
| Responsive design | PaperMod is mobile-first with light/dark/auto mode. The extra CSS was tested at 390px and 1366px wide. |
| Stable automated deploy | Hugo is a single pinned binary: no Ruby/Bundler/Node dependency drift. Uses the official GitHub Pages Actions flow, with no second repo and no `deploy.sh`. |
| Search / taxonomies / analytics | Full-text search in the browser (Fuse.js). Categories, tags and destinations. GA4, Plausible, Umami and GoatCounter placeholders. giscus comments. |

Jekyll + Chirpy is the closest alternative. It has great built-ins, but it's slower to build and
depends on a Ruby toolchain, and it doesn't match the Hugo setup you already know.

## Folder structure

```
Samya-Traveler.github.io/
├── .github/workflows/deploy.yml     # CI: build with Hugo → deploy to GitHub Pages
├── archetypes/posts.md              # template used by `hugo new content posts/...`
├── assets/css/extended/travel.css   # hero / timeline / callout / gallery styles
├── content/
│   ├── about.md
│   ├── archives.md                  # /archives/
│   ├── search.md                    # /search/ (full-text search)
│   └── posts/
│       ├── _index.md
│       └── 2026-09-20-nantou-aowanda/   # one folder per post ("page bundle")
│           ├── index.md
│           ├── cover.jpg
│           └── gallery/01-….jpg …
├── layouts/
│   ├── _markup/render-blockquote.html   # > [!TIP] callouts
│   ├── _partials/comments.html          # giscus
│   ├── _partials/extend_head.html       # Plausible / Umami / GoatCounter
│   └── _shortcodes/
│       ├── hero.html                    # {{< hero >}}
│       ├── timeline.html                # {{< timeline >}}
│       ├── stop.html                    # {{< stop >}} (timeline entry)
│       └── gallery.html                 # {{< gallery >}}
├── static/images/                   # favicon.png, og-default.jpg (served as-is)
├── themes/PaperMod/                 # git submodule (added in step 2 below)
├── hugo.yaml                        # site configuration
└── .gitignore
```

> The sample photos (`cover.jpg`, `gallery/*`, `static/images/*`) are generated placeholders.
> Replace them with your own photos.

---

## 1. Create the repository in the organization

1. Open https://github.com/organizations/Samya-Traveler/repositories/new
2. Set **Repository name** to `Samya-Traveler.github.io`. It must be exactly `<org>.github.io` to serve
   the organization's root site.
3. Set visibility to **Public**. GitHub Pages on a private repo needs a paid plan.
4. Don't add a README, .gitignore or license, because this folder already has them.

Or use the GitHub CLI: `gh repo create Samya-Traveler/Samya-Traveler.github.io --public`

## 2. Copy the site in, add the theme, push

These files currently live in the `blog/` folder of `sufuf3/samya_traveler`. Clone that repo on the
branch that contains them (`claude/elegant-newton-noubli`, or `main` once merged):

```bash
git clone -b claude/elegant-newton-noubli https://github.com/sufuf3/samya_traveler.git
git clone https://github.com/Samya-Traveler/Samya-Traveler.github.io.git

cp -R samya_traveler/blog/. Samya-Traveler.github.io/     # includes .github/ and .gitignore
cd Samya-Traveler.github.io

# Theme as a git submodule, pinned to the commit this site was tested with
git submodule add https://github.com/adityatelange/hugo-PaperMod.git themes/PaperMod
git -C themes/PaperMod checkout d3768854d00ad003b0a8dbdba254ce9224377a01

git add -A
git commit -m "Initial Samya Traveler site (Hugo + PaperMod)"
git branch -M main
git push -u origin main
```

## 3. GitHub Pages settings (one time)

In the **repository**:

1. **Settings → Pages → Build and deployment → Source**: select **GitHub Actions**. Don't use
   "Deploy from a branch". No `gh-pages` branch is needed.
2. **Actions** tab: the *Deploy Hugo site to GitHub Pages* run should turn green in about a minute.
   If it ran before step 1 and failed, open it and click **Re-run all jobs**.
3. (Optional) **Settings → Pages → Custom domain**: add the domain, then tick **Enforce HTTPS**
   once the certificate is issued. Also change `baseURL` in `hugo.yaml`.

In the **organization** (only if the run is blocked):

- **Org Settings → Actions → General → Policies**: allow at least *actions created by GitHub*.
- **Org Settings → Member privileges → Pages creation**: allow **Public** sites.

The site is then live at **https://samya-traveler.github.io/**.

## 4. Local development

Install **Hugo extended ≥ 0.146**. CI uses 0.166.0. Go is **not** required.

```bash
# macOS                          # Windows                          # Linux
brew install hugo                winget install Hugo.Hugo.Extended  sudo snap install hugo
```

```bash
git clone --recurse-submodules https://github.com/Samya-Traveler/Samya-Traveler.github.io.git
cd Samya-Traveler.github.io
# already cloned without the theme?  git submodule update --init --recursive

hugo server -D                         # http://localhost:1313, live reload, shows drafts
hugo --gc --minify                     # production build into ./public (same as CI)
```

`enableGitInfo` is on, so build inside the git clone. A folder downloaded as a ZIP won't build.

## 5. Adding a new travel post

```bash
git pull
hugo new content posts/2026-10-15-taitung-chishang/index.md   # folder = date + slug
```

1. Put `cover.jpg` in that folder, and gallery photos in `gallery/`. Name them in order
   (`01-…jpg`, `02-…jpg`) because the gallery sorts by file name. Shrink originals to about 2400px on
   the long edge first to keep the repo small; Hugo generates the smaller WebP versions.
   - macOS: `sips -Z 2400 *.jpg`
   - ImageMagick: `magick mogrify -resize '2400x2400>' *.jpg`
2. Write the post with `hugo server -D` running. The page reloads as you save.
3. When it's ready, set `draft: false`, then:

```bash
git add content/posts/2026-10-15-taitung-chishang
git commit -m "Add post: 台東池上"
git push            # GitHub Actions publishes it in about a minute
```

The URL becomes `/posts/2026/10/taitung-chishang/`, built from `date` and `slug` in the front matter.

### Authoring cheatsheet

```markdown
{{< hero src="cover.jpg" title="池上慢旅" subtitle="稻浪與山" position="center 60%" >}}

> [!TIP] 省錢小撇步
> Types: NOTE, TIP, IMPORTANT, WARNING, CAUTION. The text after [!TIP] is an optional custom title.

{{< timeline title="Day 1" >}}
{{< stop time="08:30" title="伯朗大道騎單車" icon="🚲" place="池上" >}}
Any **Markdown** here.
{{< /stop >}}
{{< /timeline >}}

{{< gallery match="gallery/*" cols="3" ratio="4x3" >}}   <!-- ratio: 4x3, 3x2, 1x1, 3x4 … -->

{{< figure src="gallery/01-rice-field.jpg" caption="說明文字" align="center" >}}

| 項目 | 內容 |
|---|---|
| 📍 地點 | … |
{.trip-facts}
```

The full template is in [`archetypes/posts.md`](archetypes/posts.md), and a complete example is in
[`content/posts/2026-09-20-nantou-aowanda/index.md`](content/posts/2026-09-20-nantou-aowanda/index.md).

## 6. Placeholders to fill in (`hugo.yaml`)

| Key | What |
|---|---|
| `params.socialIcons` | Instagram / Facebook / YouTube / Threads / email links. Delete the ones you don't use. |
| `services.googleAnalytics.ID` | GA4 ID `G-XXXXXXXXXX`. Only injected in production builds. |
| `params.privacyAnalytics.*` | Plausible domain, Umami website ID or GoatCounter code (optional, cookie-less). |
| `params.analytics.google.SiteVerificationTag` | Google Search Console verification. |
| `params.giscus.repoId` / `categoryId` | From https://giscus.app. First enable Discussions on the repo, then set `params.comments: true`. |
| `params.homeInfoParams`, `content/about.md` | Home page intro and About page text. |
| `locale`, `defaultContentLanguage` | `zh-TW` / `zh-tw` today. Use `en-US` / `en` for an English UI. |

Update the theme later with
`git submodule update --remote themes/PaperMod`, run `hugo server` to check it, then commit.
