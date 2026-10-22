# haonguy3n.github.io

[Live site](https://haonguy3n.github.io/) ·
[![Build and Deploy](https://github.com/haonguy3n/haonguy3n.github.io/actions/workflows/pages-deploy.yml/badge.svg?branch=main)](https://github.com/haonguy3n/haonguy3n.github.io/actions/workflows/pages-deploy.yml)

My personal site: notes on embedded Linux, platform security, and the edge-vision work I do on
NVIDIA Jetson Orin.

## What is here

- `_tabs/ml.md` — *Machine Learning on the Edge*, a field guide from ONNX to a CPU, GPU or NPU.
- `_tabs/jetson-orin.md` + `jetson-orin/` — TensorRT inference, precision and memory, and a
  sensor-to-GPU frame path that stays in NVMM.
- `_tabs/engineering.md` — coding rules and a copy-paste PR checklist.
- `_posts/` — short notes, including the launch note for the Jetson Orin section.
- `_includes/jetson-orin-style.html` — the responsive flow-box styles used by the notes.

## Build locally

```sh
bundle install
bundle exec jekyll serve
```

## Deployment

`.github/workflows/pages-deploy.yml` runs on content changes to `main`: it builds the site with
Jekyll, runs `html-proofer`, uploads the artifact and deploys it to GitHub Pages. A README-only
change is intentionally ignored by the workflow, so the badge above reflects the last content
build.

## License

The site content is published under CC BY 4.0; the Chirpy theme is MIT. See `LICENSE` for the
theme licence.
