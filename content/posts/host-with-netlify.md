---
title: "Host Hugo with Netlify"
date: 2026-01-09T01:56:08-05:00
draft: true
categories: ["DevOps", "CI/CD"]
tags: ["git", "netlify", "devops", "ci-cd", "hosting", "cloudflare", "static-site"]
author: "me"
description: "Build & deploy a Hugo site from a Github/Gitlab repository and host it on Netlify behind a custom domain (with optional Cloudflare)."
showToc: false
TocOpen: false
hidemeta: false
comments: false
searchHidden: false
---

## Outline

* [x] Summary
  * [x] Article summary describing the purpose and tools/requirements.
* [ ] Hugo setup
  * [ ] Git repo init
  * [x] Hugo project init
  * [x] Adding themes
  * [ ] The `hugo.yml` file
* [ ] Git repo setup
  * [ ] `task` file
  * [ ] `netlify.toml` file
  * [ ] Github action
* [ ] Netlify setup
  * [ ] Repo integration
  * [ ] Disable automated deployments (handled by Github action)

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
