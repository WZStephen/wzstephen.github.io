# wzstephen.github.io

Personal GitHub Pages homepage and bilingual resume source.

## Files

- `index.html`: English online resume.
- `index-cn.html`: Chinese online resume.
- `Files/EN_Resume_Weichi_Zhao.pdf`: generated English PDF resume.
- `Files/CN_Resume_Weichi_Zhao.pdf`: generated Chinese PDF resume.

## Generate PDFs

From this directory:

```bash
google-chrome --headless --disable-gpu --no-sandbox --no-pdf-header-footer --print-to-pdf=Files/EN_Resume_Weichi_Zhao.pdf index.html
google-chrome --headless --disable-gpu --no-sandbox --no-pdf-header-footer --print-to-pdf=Files/CN_Resume_Weichi_Zhao.pdf index-cn.html
```

Updated by Weichi Zhao, 2026.06.
