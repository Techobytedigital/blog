---
title: "Host Hugo with Netlify"
date: 2026-01-09T01:56:08-05:00
draft: true
categories: ["DevOps", "CI/CD"]
tags: ["git", "netlify", "devops", "ci-cd", "hosting", "cloudflare", "static-site"]
author: "me"
description: "Build & deploy a Hugo site from a Github/Gitlab repository and host it on Netlify behind a custom domain (with optional Cloudflare)."
showToc: true
TocOpen: false
hidemeta: false
comments: false
searchHidden: false
---

[Hugo](https://gohugo.com) is a [static site generator](https://www.cloudflare.com/learning/performance/static-site-generator/) that enables you to write your website's content as Markdown files, and render them to nice-looking websites using [Hugo themes](https://themes.gohugo.io). The word "static" in this context means that the files Hugo renders are meant to be served as-is, there is no "backend" for the site (unless you add custom Javascript code).

Static sites are perfect for many different kinds of websites, from personal blogs like this site to product sites (i.e. [Brave browser's website](https://brave.com)) to an organization's home page (i.e. [the LetsEncrypt project's site](https://letsencrypt.org)) to documentation sites (i.e. [DigitalOcean's documentation](https://docs.digitalocean.com)). With a static site generator, you do not have to deal with web code like HTML or Javascript, but you still have the option of [creating your own custom themes](https://gohugo.io/commands/hugo_new_theme/) if that appeals to you.

When Hugo renders your source code into a website, it outputs everything you need to start serving the site to a `public/` directory, allowing you to host the page in [Docker](https://hugomods.com), or on a VPS you control, or with a service like [Netlify](https://www.netlify.com). You could even host your site on a [Raspberry Pi you own](https://snapcraft.io/install/hugo/raspbian) (although I would recommend looking into running the site in a Docker container, instead of a Canonical Snap). Anywhere that can serve HTML should be able to serve your compiled Hugo website.

This blog will run through the basic steps to initialize a Hugo project, host it on Github, and setup hosting with Netlify.

## Requirements

* [Hugo](https://gohugo.io/installation/)
  * You should also pick a [Hugo theme](https://themes.gohugo.io), i.e. [PaperMod](https://themes.gohugo.io/themes/hugo-papermod/)
* [Go](https://go.dev/doc/install) (optional)
  * You can [add themes as a Hugo module](https://gohugo.io/hugo-modules/use-modules/) instead of using Git submodules.
* [Git](https://git-scm.org)
* [Github account](https://github.com)
* [Netlify account](https://netlify.com)

## Initialize Hugo repository

There are different ways of structuring the repository, like putting the site's content in a `src/` directory or storing multiple Hugo sites in a single monorepo. This guide assumes you follow the [default steps for initializing a Hugo project](https://gohugo.io/getting-started/quick-start/#create-a-site), where all source code is in the "root" of the repository.

First, `cd` to a path where you want to initialize your Hugo site, i.e. `~/git` or just `~/` and initialize the site with:

```shell
hugo new site <your-blog-name>
```

Replace `<your-blog-name>` above with the name of your blog. The documentation uses `quickstart` as an example. You can name the site whatever you want, and can change it later by editing the `hugo.toml`/`hugo.yml` file the `hugo new site` command creates. When you run the Hugo command, you will see output with some first steps and instructions for editing the `hugo.toml` to change the site's configuration:

```shell
$> hugo new site your-site-name
Congratulations! Your new Hugo site was created in /home/username/git/your-site-name.

Just a few more steps...

1. Change the current directory to /home/username/git/your-site-name.
2. Create or install a theme:
   - Create a new theme with the command "hugo new theme <THEMENAME>"
   - Or, install a theme from https://themes.gohugo.io/
3. Edit hugo.toml, setting the "theme" property to the theme name.
4. Create new content with the command "hugo new content <SECTIONNAME>/<FILENAME>.<FORMAT>".
5. Start the embedded web server with the command "hugo server --buildDrafts".

See documentation at https://gohugo.io/.
```

This is an example of the files `hugo new site` creates:

```shell
your-site-name/
├── themes
├── static
├── layouts
├── i18n
├── hugo.toml
├── data
├── content
├── assets
└── archetypes
    └── default.md
```

Since this blog post is not meant to be an in-depth tutorial of how to build a website with Hugo, we will not go into detail on every single file and folder this command creates, but some important things to know are:

* The `themes/` directory is where you store themes added as Git submodules.
  * This guide assumes you are using Hugo modules to install themes, instead of the old Git submodule way, so you will not need to interact with the `themes/` directory.
  * This is also where you would store a custom theme, if you wanted to create your own Hugo theme for your site.
* The `static/` directory is where you would store static assets for your site, including images and a `favicon.ico` for the site.
  * Although counter-intuitive, it is generally considered best practice to include the binaries for your static resources in the Git repository.
  * Save your `favicon.ico` to `static/favicon.ico` (if you have one), and any images you use in your posts in `static/img` (this directory doesn't exist by default, you have to create it).
* The `layouts/` directory is where you can create [Hugo templates](https://gohugo.io/content-management/data-sources/) for finer-grained control of how content is displayed on your site.

The `hugo.toml` file is the main configuration file for your site. If you prefer YAML, you can copy the contents of `hugo.toml` into [convertsimple.com](https://www.convertsimple.com/convert-toml-to-yaml/), delete or rename `hugo.toml` to `hugo.yaml`/`hugo.yml`, and paste the converted configuration into the YAML file. Hugo will detect a file named `hugo.yml` or `hugo.toml` wherever you run `hugo` commands from. This guide assumes you are using YAML for your configuration.

{{< notice tip >}}
You can initialize Hugo with a YAML config file instead of TOML by running the  `hugo new site` command with a `--format yaml` flag:

```shell
hugo new site site-name --format yaml
```

{{< /notice >}}

## Convert Site to Hugo Module

Next, initialize your site as a [Hugo module](https://gohugo.io/commands/hugo_mod_init/):

```shell
hugo mod init github.com/username/your-site-name
```

After running this, you will see a `go.mod` file. This allows Go to mannage your site so you can install themes like they are Go packages. For example, to install the `PaperMod` theme by editing your `hugo.yml` file and adding the following block of code:

```yaml
module:
  imports:
    - path: github.com/adityatelange/hugo-PaperMod
```

The full `hugo.yml` should now look like this:

```yaml
baseURL: 'https://example.org/'
languageCode: en-us
title: My New Hugo Site

module:
  imports:
    - path: github.com/adityatelange/hugo-PaperMod
```

{{< notice note >}}
Run `hugo mod tidy`; this will download the theme(s)/module(s) you declare, remove old versions, and update your `go.mod` file for you.
{{< /notice >}}

## Run Local Hugo Development Server

Now you can test running the server with `hugo serve`, which will compile the site and start serving it at `http://localhost:1313/` by default. To change the host address, i.e. to access it from another machine on the network, use `--bind 0.0.0.0`, and to change the port use `-p <port>`. This is Hugo's development server. As you edit files, the server will 'hot reload,' re-compiling the Markdown you write and restarting the server.

Hugo does this *very* quickly. Each page takes mere milliseconds to render, even those with images or a lot of text, and it caches between rebuilds, only re-rendering the new content. This leads to a pleasant "change something and see it instantly" writing experience.

If you start the server with `-D`, Hugo will also render your drafts in the development server.

## Git Repository Setup

Run `git init -b main` in your Hugo site's directory to initialize a new Git repository. Before committing any code, add a `.gitignore` that ignores Hugo's `public/` and `resources/` directories that it creates when you run the development server:

```plaintext
/public/
/resources/
```

Then, do your initial commit:

```shell
git add *
git commit -m "Initial commit"
```

Create a repository on Github for your Hugo site. This should match the URL you used when you create the Hugo module with `hugo mod init github.com/user/your-site-name`. Add the remote to your local repository:

```shell
git remote set origin git@github.com:username/your-site-name.git
```

And push your code with `git push -u origin main`.

## Netlify Setup

Create an account on [Netlify](https://netlify.com). During the setup process, you will be asked to create a project. Allow Netlify to integrate with your Github and find the repository with your Hugo site. If you already have a Netlify site, add a new project and use "Import an existing project" and find your Hugo repository.

When you are prompted to input build settings like the base directory/package directory, the build command, the publish directory, etc, leave all of these blank. You can optionally leave the functions directory as `netlify/functions`. If you set values here, the site will fail to build because the `hugo.yml` interferes with these settings. Do your site configuration in `hugo.yml`, not Netlify.

### netlify.toml Config

After finishing the setup, create a `netlify.toml` file in the root of your Hugo repository :

{{< notice note >}}
If you have a domain name, you can set it in `HUGO_BASEURL` instead of the Netlify app URL. Also, make sure to check the current versions of [Hugo](https://github.com/gohugoio/hugo/releases) and [Node](https://nodejs.org/en/download/current). for NPM versions, just put the major version number, i.e. `24`, `25`, etc.
{{< /notice >}}

```toml
[build.environment]
HUGO_BASEURL = "https://netlify-project-name.netlify.app"
HUGO_VERSION = "0.152.2"
HUGO_ENV = "production"
NODE_VERSION = "24"
HUGO_ENABLEGITINFO = "true"

[build]
command = "hugo --gc --minify"
publish = "public"
```

This file tells Netlify how to build your Hugo site. Commit it to git with `git add * && git commit -m "Add netlify config"` and push to main with `git push`. Even better, create a new branch before adding/committing your code, i.e. `git switch -c feat/netlify-setup`, then add the files and commit message. When you push, use `git push -u origin feat/netlify-setup`. Merge the branch into the `main` branch in Github.

## Github Action: Publish to Netlify

Before setting up a Github action, you need to do some setup in Github itself. Open the repository's settingss and make the following changes:

* In `Code and Automation > Actions > General`, select `Allow all actions and reusable workflows`
* In `Code and Automation > Environments`, create a 'production' environment.
  * In `Environment Secrets`, add the following secrets:
    * `NETLIFY_AUTH_TOKEN`
      * API token created on Netlify's site.
      * Follow the [Netlify docs - Authentication](https://docs.netlify.com/api-and-cli-guides/api-guides/get-started-with-api/#authentication) steps to create an API key for the Github Action.
    * `NETLIFY_SITE_ID`
      * Find this in your Netlify project's configuration, under `General > Project details > Project information`.
      * Use the `Project ID` value.
  * In `Environment varaibles`, set `HUGO_BASEURL` to your Netlify app URL or your custom domain.

In your local Git repository, create a file at `.github/workflows/build-and-deploy.yml` (you can name the file whatever you want).

```yaml
---
name: "Netlify Build & Release"
env:
  ## https://github.com/gohugoio/hugo/releases
  HUGO_VERSION: "0.154.3"

on:
  pull_request:
    branches: [main]
    ## Only run on PRs that have a label. Use a conditional in a later step
    #  to determine if a build/deploy should run on a PR open
    types: [labeled, synchronize]

permissions:
  contents: read
  pull-requests: write

jobs:
  test:
    ## Check if PR has 'netlify-release' label
    if: contains(github.event.pull_request.labels.*.name, 'netlify-release')
    runs-on: ubuntu-latest
    outputs:
      success: ${{ steps.build.outcome == 'success' }}
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Setup Hugo
        uses: peaceiris/actions-hugo@v3
        with:
          hugo-version: ${{ env.HUGO_VERSION }}
          extended: true

      - name: Build site
        id: build
        run: hugo --gc --minify

      ## Optional: Use lychee to test links in rendered files before deploying
      # - name: Test links
      #   uses: lycheeverse/lychee-action@v1
      #   with:
      #     args: "public/**/*.html --base-verification false"

      ## Optional: Ensure RSS feed generated correctly
      # - name: Check RSS
      #   run: |
      #     curl --fail -s public/rss.xml | grep -q "${{ vars.HUGO_BASEURL }}" || exit 1

      ## Create pipeline artifact for deploy stage
      - name: Upload site artifact
        if: steps.build.outcome == 'success'
        uses: actions/upload-artifact@v4
        with:
          name: hugo-site
          path: public
          retention-days: 1

      ## Create a deploy preview in Netlify.
      #  This is basically a feature environment, which can be promoted to the Production slot.
      - name: Deploy Preview
        if: steps.build.outcome == 'success'
        id: preview
        uses: nwtgck/actions-netlify@v3
        with:
          publish-dir: "./public"
          production-branch: main
          deploy-message: "Preview PR #${{ github.event.pull_request.number }}"
          ## Posts URL as PR comment
          enable-pull-request-comment: true
        env:
          NETLIFY_AUTH_TOKEN: ${{ secrets.NETLIFY_AUTH_TOKEN }}
          NETLIFY_SITE_ID: ${{ secrets.NETLIFY_SITE_ID }}

  deploy:
    needs: test
    ## Ensure build/test passed, check if PR has 'netlify-release' label
    if: needs.test.outputs.success == 'true' && contains(github.event.pull_request.labels.*.name, 'netlify-release')
    runs-on: ubuntu-latest
    environment: production
    steps:
      - name: Download built site
        uses: actions/download-artifact@v4
        with:
          name: hugo-site
          path: public

      - name: Deploy to Netlify
        uses: nwtgck/actions-netlify@v3
        with:
          publish-dir: "./public"
          production-branch: main
          deploy-message: "Deploy from PR #${{ github.event.pull_request.number }}"
        env:
          NETLIFY_AUTH_TOKEN: ${{ secrets.NETLIFY_AUTH_TOKEN }}
          NETLIFY_SITE_ID: ${{ secrets.NETLIFY_SITE_ID }}
```

When you want to deploy changes to your live site, create a branch with your changes, and when you open a pull request into the `main` branch, add a label `netlify-release`. Any PR with this label will trigger this Action.

Before completing the pull request, go to `Deploys` in your Netlify project and test the changes in the Deploy Preview environment. When you are satisfied, go back to the pull request in Github and click the link to the deploy pipeline. The pipeline should be paused, waiting for you to approve the release to the Production slot.

## Bonus: A Production Hugo Dockerfile

If you plan to host your Hugo site yourself, you can set up a Dockerfile to build your site and serve it behind a proxy like [Caddy](https://caddyserver.com) or [NGINX](https://nginx.org/en/).

You can use a multi-stage Docker build to have a smaller final image that only has your rendered HTML and the Caddy server to serve it:

```dockerfile
FROM hugomods/hugo:exts AS builder

WORKDIR /src

## Copy Hugo site files & config
COPY archetypes/ archetypes/
COPY assets/ assets/
COPY content/ content/
COPY layouts/ layouts/
COPY static/ static/
COPY hugo.yml go.mod go.sum .hugo_build.lock ./

ARG HUGO_BASEURL=http://localhost/

RUN hugo \
  --minify \
  --gc \
  --baseURL="${HUGO_BASEURL}" \
  --destination /output/public

FROM caddy:alpine

COPY --from=builder /output/public /usr/share/caddy

COPY ./Caddyfile /etc/caddy/Caddyfile

EXPOSE 80 443

CMD ["caddy", "run", "--config", "/etc/caddy/Caddyfile"]

```

Create a `Caddyfile` the container will use to configure the Caddy server:

```caddyfile
{
  # debug
  email {$CADDY_EMAIL}
}

{$CADDY_SITE_ADDRESS} {
  root * /usr/share/caddy
  file_server
  encode gzip

  header {
    X-Content-Type-Options nosniff
    X-Frame-Options DENY
  }
}
```

When you build the container, you will need to set environment variables for `CADDY_EMAIL` (for automated LetsEncrypt SSL certificates) and `CADDY_SITE_ADDRESS` (for handling connections coming to the Caddy URL). The 2 env vars are ["build arguments"](https://docs.docker.com/build/building/variables/), which you can inject in your `docker build` command using `--build-arg ARG_NAME=value`. For example, to build this Dockerfile:

```shell
docker build --build-arg CADDY_EMAIL="youremail@address.com" --build-arg CADDY_SITE_ADDRESS="myblog.com" .
```

## Closing

Building a blog with Hugo is fun, and relatively quick and easy to get up and running. You can build a complex site for your product, or a simple blog you update a couple times a year, all with the same tooling. The minimal amount of configuration you need to get started lends itself to quick development, while still allowing you to build up [a more complex configuration over time](https://gohugo.io/configuration/). Hugo modules make it even easier to install themes and site extensions, and the ability to host the site anywhere that can serve static web files gives you a ton of flexibility to where you put the site online.

This guide did not go in-depth on configuring and customizing Hugo, nor did it walk through creating content for the site. The [Hugo quickstart guide](https://gohugo.io/getting-started/) will do a better job of walking you through building your first blog, and I highly recommend keeping this site open and using the search bar at the top as you're learning to find new ways of using Hugo.
