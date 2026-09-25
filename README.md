# Light Roasted Platform Engineering

The publication site for the Light Roasted Platform Engineering blog, built with [Hugo](https://gohugo.io/) and the [Blowfish](https://blowfish.page/) theme.

## Prerequisites

- **Hugo Extended** (v0.120+ recommended, tested with v0.162.x)
- **Go** (for Hugo Modules)

## Local Development

Start the local development server:

```bash
hugo server
```

To include drafts and future-dated posts:

```bash
hugo server -D
```

Open [http://localhost:1313](http://localhost:1313) in your browser.

## Build

Compile the static site to the `public/` directory:

```bash
hugo --gc --minify
```

## Structure

- `content/`: Blog posts, pages, and research articles.
- `layouts/`: Custom template and shortcode overrides.
- `assets/`: Custom styling, scripts, and imagery.
- `config/`: Site configuration and Blowfish theme settings.
- `static/`: Static files served at the root (favicons, etc.).
