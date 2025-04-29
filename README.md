# Ding's Things Blog

Welcome to **Ding's Things** — a technical blog powered by [Hugo](https://gohugo.io/) and a customized fork of the [PaperMod](https://github.com/adityatelange/hugo-PaperMod) theme.

---

## 📁 Repository Overview

This repository hosts the source code for [https://dingyu.dev](https://dingyu.dev), including:

- Blog posts (in Markdown)
- Configuration files (Hugo settings, SEO, sitemap, etc.)
- Customized theme (`hugo-PaperMod`) as a Git submodule

> **Important**: The theme is managed as a submodule pointing to [https://github.com/dings-things/hugo-PaperMod](https://github.com/dings-things/hugo-PaperMod).


## ✨ Live Site

- **URL**: [https://dingyu.dev](https://dingyu.dev)
- **Deployment**: Automated via CI (Continuous Integration)
  - Push to `main` branch triggers build & deployment.


## 🛠️ Getting Started (Local Setup)

### 1. Clone the Repository

```bash
git clone --recurse-submodules https://github.com/dings-things/blog.git
cd blog
```

If you already cloned without `--recurse-submodules`, run:

```bash
git submodule update --init --recursive
```


### 2. Install Hugo

Make sure you have **Hugo Extended Version** installed. Check with:

```bash
hugo version
```

If not installed:

- [Installation Guide](https://gohugo.io/getting-started/installing/)


### 3. Run Local Development Server

```bash
hugo server -D
```

- Access the local server at: `http://localhost:1313`
- `-D` flag builds draft posts as well.


## 💼 Folder Structure

```
blog/
├── config/             # Hugo config files
├── content/            # Blog content (posts, pages)
├── static/             # Static assets (images, favicon, etc.)
├── themes/
│   └── hugo-PaperMod/ # Customized PaperMod theme (submodule)
├── layouts/            # Custom Hugo layouts and partials
├── assets/             # CSS, JS assets
├── README.md           # This file
└── ...
```


## 🔄 Theme Customization

This project uses a forked and customized version of **PaperMod**.

- Repository: [https://github.com/dings-things/hugo-PaperMod](https://github.com/dings-things/hugo-PaperMod)
- Major Customizations:
  - Mermaid.js native integration (optimized for mobile)
  - Improved dark/light theme toggling
  - Minor UI/UX improvements

> Any updates to the theme should be managed via updating the submodule reference.


## 📊 CI/CD Deployment

- GitHub Actions or other CI tools automatically:
  - Build the Hugo project
  - Deploy to hosting (e.g., GitHub Pages, AWS S3, or others)

**Deployment flow**

```
Push to main ➔ CI Build ➔ Deploy to https://dingyu.dev
```


## 💬 Contributions

This repository is mainly for personal usage.
However, issues or suggestions are welcome!

Feel free to open an [Issue](https://github.com/dings-things/blog/issues) or [PR](https://github.com/dings-things/blog/pulls) if you have any ideas to improve the blog structure, performance, or styling.


---

Thanks for visiting!

> Beyond a programmer — a problem solver without boundaries.