---
title: "How to Create and Deploy a Hugo Website With GitHub Pages"
date: 2026-05-09
draft: false
summary: "A simple guide to creating a Hugo website, publishing it with GitHub Actions, and using a custom domain with HTTPS."
tags: ["hugo", "github-actions", "github-pages", "dns", "website"]
categories: ["website"]
---

This guide shows a simple way to create a Hugo website and publish it with GitHub Pages. The workflow is:

- Write the site posts in markdown.
- Store the source code in GitHub.
- Let GitHub Actions build the site.
- Let GitHub Pages publish the generated files.

This setup works well for a personal blog, portfolio, project page, or documentation site.

## 1. Create A Hugo Site

Create the project:

```sh
hugo new site my-website
cd my-website
git init
```

Add a theme. This example uses PaperMod as a Git submodule:

```sh
git submodule add https://github.com/adityatelange/hugo-PaperMod.git themes/PaperMod
```

Configure the site in `hugo.toml`:

```toml
baseURL = "https://example.com/"
locale = "en-us"
title = "My Website"
theme = "PaperMod"
enableRobotsTXT = true

[pagination]
  pagerSize = 10

[params]
  author = "Your Name"
  defaultTheme = "auto"
  showReadingTime = true
  showCodeCopyButtons = true
  showtoc = true
  tocopen = false
  description = "A short description of the website."

[markup]
  [markup.highlight]
    lineNos = true
  [markup.goldmark.renderer]
    unsafe = false

[minify]
  minifyOutput = true

[caches]
  [caches.images]
    dir = ':cacheDir/images'

[outputs]
  home = ["HTML", "RSS", "JSON"]
```

Replace `https://example.com/` with your real domain when you have one.

If you do not have a custom domain yet, you can still deploy using the default GitHub Pages URL `https://<your-github-username>.github.io/`

Create a `.gitignore` file to avoid uploading generated Hugo files:

```text
/public/
/resources/
/.hugo_cache/
.hugo_build.lock
```

## 2. Push The Initial Site To GitHub

Create a repository on GitHub, then connect your local project:

```sh
git remote add origin https://github.com/<your-github-username>/<repository-name>.git
git add .
git commit -m "Create Hugo website"
git branch -M main
git push -u origin main
```

If your theme is a submodule, make sure `.gitmodules` is committed. The deployment workflow needs it to fetch the theme.

## 3. Use Branches For Changes (optional)

Use `main` as the deployment branch. For writing, editing, or changing the deployment configuration, create a separate branch.

This guide uses `update-website` as the working branch name. If you prefer another name, replace `update-website` in the commands.

```sh
git switch -c update-website
```

Make your changes, preview locally, then commit:

```sh
git add .
git commit -m "Add new website content"
git push origin update-website
```

Once you want to publish something, open a pull request from `update-website` into `main`. When the pull request is merged, GitHub Actions will build and deploy the site.

## 4. Create And Preview Posts

Create a new post:

```sh
hugo new content posts/my-first-post.md
```

Open the file and set `draft` to `false` when you want it published:

```Markdown
---
title: "My First Post"
date: 2026-05-08
draft: false
summary: "A short summary for previews."
tags: ["hugo", "website"]
---
# Post in markdown
Keep images inside `static/img/...` or another organized folder.

![example-image](/img/posts/post_name/example-image.png)
```

Preview the site locally:

```sh
hugo server
```

Then open the local URL printed by Hugo, usually:

```text
http://localhost:1313/
```

If you want to preview drafts too, run:

```sh
hugo server -D
```

Once you have finished editing your post, upload your branch to GitHub:

```sh
git add content/posts/my-first-post.md
git commit -m "My first post"
git push origin update-website
```

In GitHub, create a pull request from `update-website` to `main`. Merge it when you are ready to publish.

## 5. Configure GitHub Pages And Actions

In the GitHub repository, go to:

```text
Settings -> Pages
```

Under **Build and deployment**, set:

```text
Source: GitHub Actions
```

This tells GitHub Pages to publish the artifact produced by your workflow.

If you have a custom domain, add it in the repository's GitHub Pages  Custom domain settings and enable HTTPS. If you do not have one leave the form empty.

![GitHub Pages source set to GitHub Actions](/img/posts/how-to-website/github-pages-source-actions.png)

Also, create the file `static/CNAME` with the following content:
```text
<your-domain>.com
```

Create the workflow file:

```text
.github/workflows/hugo.yaml
```

Use a workflow like this, based on [Hugo's official GitHub Pages guide](https://gohugo.io/host-and-deploy/host-on-github-pages/#step-4) and GitHub's Pages workflow documentation:

```yaml
name: Build and deploy

on:
  push:
    branches:
      - main
  workflow_dispatch:

permissions:
  contents: read
  pages: write
  id-token: write

concurrency:
  group: pages
  cancel-in-progress: false

defaults:
  run:
    shell: bash

jobs:
  build:
    runs-on: ubuntu-latest
    env:
      HUGO_VERSION: 0.160.0
    steps:
      - name: Checkout
        uses: actions/checkout@v6
        with:
          submodules: recursive
          fetch-depth: 0

      - name: Setup Pages
        id: pages
        uses: actions/configure-pages@v6

      - name: Install Hugo
        run: |
          curl -sLJO "https://github.com/gohugoio/hugo/releases/download/v${HUGO_VERSION}/hugo_extended_${HUGO_VERSION}_linux-amd64.tar.gz"
          mkdir "${HOME}/hugo"
          tar -C "${HOME}/hugo" -xf "hugo_extended_${HUGO_VERSION}_linux-amd64.tar.gz"
          echo "${HOME}/hugo" >> "${GITHUB_PATH}"

      - name: Build the site
        run: |
          hugo build \
            --gc \
            --minify \
            --baseURL "${{ steps.pages.outputs.base_url }}/"

      - name: Upload artifact
        uses: actions/upload-pages-artifact@v5
        with:
          path: ./public

  deploy:
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    runs-on: ubuntu-latest
    needs: build
    steps:
      - name: Deploy to GitHub Pages
        id: deployment
        uses: actions/deploy-pages@v5
```

This version is intentionally simple. If your theme needs Sass, npm packages, Go modules, or image processing, start from Hugo's official workflow because it includes extra setup and caching.

## 6. Deploy The Site

Commit and push the workflow:

```sh
git add .github/workflows/hugo.yaml hugo.toml .gitignore
git commit -m "Add Hugo GitHub Pages deployment"
git push origin update-website
```

In GitHub, create a pull request from `update-website` to `main`. The site deploys when the pull request is merged into `main`.

Open:

```text
Repository -> Actions
```

Watch the workflow run. If it succeeds, GitHub Pages publishes the site.

Example:
![Successful GitHub Actions deployment](/img/posts/how-to-website/github-actions-success.png)

## 7. Configure DNS

You can skip this step if you want to use the default GitHub Pages URL.

If you have your own custom domain, at this step you have to go to your DNS provider, such as Cloudflare, Porkbun, or Namecheap, to point the domain to GitHub Pages.

At your DNS provider, configure the apex/root domain with GitHub Pages' `A` records:

| Type | Host | Value |
|---|---|---|
| A | @ | 185.199.108.153 |
| A | @ | 185.199.109.153 |
| A | @ | 185.199.110.153 |
| A | @ | 185.199.111.153 |

Optional IPv6 records:

| Type | Host | Value |
|---|---|---|
| AAAA | @ | 2606:50c0:8000::153 |
| AAAA | @ | 2606:50c0:8001::153 |
| AAAA | @ | 2606:50c0:8002::153 |
| AAAA | @ | 2606:50c0:8003::153 |

If you want to also configure the subdomain for `www.<your-domain>`, add a CNAME record:

| Type | Host | Value |
|---|---|---|
| CNAME | www | `<your-github-username>.github.io` |

For example, a DNS provider dashboard should show the GitHub Pages `A` records for the apex domain and a `www` CNAME pointing to `<your-github-username>.github.io`.

For example, [ernest.sh](https://ernest.sh) is hosted in Porkbun and this is the DNS config I use:

![DNS records for GitHub Pages](/img/posts/how-to-website/dns-records.png)

## Final Notes

I recommend using a separate branch for drafts and edits, preview locally with `hugo server`, and merge to `main` only when you are ready to publish.

You can use Hugo's in-memory rendering and a temporary cache directory to preview locally. This helps manage files in your repository.
```sh 
hugo server -D --renderToMemory --cacheDir "${TMPDIR:-/tmp}/hugo-cache" --disableFastRender

```

After that, publishing is simple: write Markdown, commit, merge, and GitHub Pages serves the updated site automatically.

Treat every committed file as public. Do not commit:
- `.env` files.
- API keys.
- Passwords.
- Cloud provider publish profiles.
- Private certificates.
- Client data.
- Internal infrastructure notes.
- Drafts with sensitive information.

## References

1. [Hugo: Host on GitHub Pages][1]  

2. [PaperMod: Variables][2]

3. [PaperMod GitHub repository][3]

4. [GitHub Docs: Using custom workflows with GitHub Pages][4]

5. [GitHub Docs: Managing a custom domain for GitHub Pages][5]

6. [GitHub Docs: Verifying a custom domain for GitHub Pages][6]

7. [Porkbun: Connect your domain to GitHub Pages][7]

[1]: https://gohugo.io/host-and-deploy/host-on-github-pages/
[2]: https://github.com/adityatelange/hugo-PaperMod/wiki/Variables
[3]: https://github.com/adityatelange/hugo-PaperMod
[4]: https://docs.github.com/en/pages/getting-started-with-github-pages/using-custom-workflows-with-github-pages
[5]: https://docs.github.com/en/pages/configuring-a-custom-domain-for-your-github-pages-site/managing-a-custom-domain-for-your-github-pages-site
[6]: https://docs.github.com/en/pages/configuring-a-custom-domain-for-your-github-pages-site/verifying-your-custom-domain-for-github-pages
[7]: https://kb.porkbun.com/article/64-how-to-connect-your-domain-to-github-pages
