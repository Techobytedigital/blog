# Development Documentation

[!WARNING]
This documentation is most likely incomplete. If I ever feel like this site is "well documented," I'll remove this message.

## Creating new content pages

Content served on the site is stored in the [`content/' directory](../content). While you can manually create directories (pages) and Markdown files (pages), it's easier to use Hugo for this:

```shell
hugo new <folder-name>/<file-name>.md
```

For example, to create a new [post](../content/posts) named `example.md`:

```shell
hugo new post/example.md
```

If you're creating a new page based on an existing [archetype](../archetypes), can use:

```shell
hugo new --kind <archetype> <name>
```

## Serving the site for development

Serve on `localhost:1313`:

```shell
hugo server -D
```

Serve on `0.0.0.0:1313`:

```shell
hugo server -D --bind 0.0.0.0
```

Serve on `0.0.0.0:8000`:

```shell
hugo server -D --bind 0.0.0.0 -p 8000
```

## Building the site

Simply running `hugo` builds the site in a `public/` directory at the repository root. You can add additional flags to control the way Hugo builds/runs.

| Flag                    | Description                                       |
| ----------------------- | ------------------------------------------------- |
| `--minify`              | Optimize code files for a production environment. |
| `--gc`                  | Do garbage collection/cleanup unused files.       |
| `-D`                    | Include drafts.                                   |
| `--cleanDestinationDir` | Do a fresh build by removing old files.           |

A good command to run in a production/live environment is:

```shell
hugo --minify --gc --cleanDestinationDir
```
