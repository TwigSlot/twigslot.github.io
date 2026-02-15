# twigslot.github.io

A digital garden built with [Quartz 4](https://quartz.jzhao.xyz/).

## Local development

```bash
npm i
npx quartz build --serve
```

The site will be available at `http://localhost:8080`.

## Adding a new blog post

1. Create a markdown file in `content/`, e.g. `content/2026-02-15-my-post.md`
2. Add frontmatter at the top:
   ```yaml
   ---
   title: "My post title"
   date: 2026-02-15
   tags:
     - topic
   ---
   ```
3. Write your content below the frontmatter.
4. Build and deploy:
   ```bash
   npx quartz build
   mv public docs
   git add -A && git commit -m "new post" && git push
   ```
