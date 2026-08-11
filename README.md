# wzstephen.github.io

Personal GitHub Pages site for personal essays, AI infrastructure notes, and
bilingual PDF resumes.

## Files

- `index.html`: short personal homepage with PDF resume and content entrances.
- `_posts/`: manually authored personal essays.
- `resume/index.html`: English PDF source; excluded from the published Jekyll
  site.
- `resume/index-cn.html`: Chinese PDF source; excluded from the published
  Jekyll site.
- `resume/archive/`: dated resume snapshots retained for comparison.
- `Files/EN_Resume_Weichi_Zhao.pdf`: generated English PDF resume.
- `Files/CN_Resume_Weichi_Zhao.pdf`: generated Chinese PDF resume.

## Personal Essays

Personal essays and reflections should use date-prefixed filenames:

```text
YYYY-MM-DD-personal-essay-slug.md
```

Each post needs front matter:

```yaml
---
layout: post
title: '短而具体的标题'
date: YYYY-MM-DD 09:00:00 +0800
categories: [personal-essay]
---
```

Use the optional presentation fields for the full essay layout:

```yaml
eyebrow: '短分类'
description: '一到两句文章导语'
thesis: '文章中已经成立的核心判断'
visual_center: '中心概念'
visual_terms: ['关联概念一', '关联概念二', '关联概念三']
visual_caption: '解释这组关系的一句话'
```

The post layout renders these fields as a topic label, deck, lead quote, and
article-specific relationship graphic. Keep `thesis` in the author's own voice;
do not present a paraphrase as an external quotation. Fenced `text` diagrams
render as structure cards inside the article.

Write essays in first person by default, with plain language and concrete
examples. The homepage shows the newest `personal-essay` posts.

## Resume Publishing

The public homepage routes resume traffic through the About page, which links
directly to the Chinese and English PDF files. The HTML files under `resume/`
are generation sources only and are excluded from the published Jekyll site.
Edit those sources first, regenerate both PDFs with headless Chrome, and
visually verify every page before publishing.

## Generate PDFs

From this directory:

```bash
google-chrome --headless --disable-gpu --no-sandbox --no-pdf-header-footer --print-to-pdf=Files/EN_Resume_Weichi_Zhao.pdf resume/index.html
google-chrome --headless --disable-gpu --no-sandbox --no-pdf-header-footer --print-to-pdf=Files/CN_Resume_Weichi_Zhao.pdf resume/index-cn.html
```

Updated by Weichi Zhao, 2026.07.
