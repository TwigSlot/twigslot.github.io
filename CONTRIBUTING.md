# Contributing Checklist

Before committing and pushing, verify:

- [ ] **New posts are linked from `content/index.md`** — every new blog post must be added to the home page
- [ ] **Folder names use dashes** (not spaces) — e.g. `performance-engineering/` not `performance engineering/`
- [ ] **Build docs** — run `npx quartz build --directory content -o docs` before pushing
- [ ] **Draft flag** — set `draft: false` if the post should be live
