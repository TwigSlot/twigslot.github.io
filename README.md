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

## Editing with Obsidian via Syncthing

You can edit the `content/` folder from Obsidian on another machine (e.g. Windows) using [Syncthing](https://syncthing.net/) to keep files in sync with the server.

### Server (headless, via SSH)

```bash
sudo apt install syncthing
syncthing --no-browser &
```

The web UI listens on `localhost:8384`. Tunnel to it from your local machine:

```bash
ssh -L 8384:localhost:8384 user@server
```

Then open `http://localhost:8384` in your browser.

### Local machine (Windows/Mac/Linux)

1. Install Syncthing from https://syncthing.net/
2. On the server's web UI, go to **Actions > Show ID** and copy the device ID
3. On your local Syncthing, click **Add Remote Device** and paste the server's ID
4. Accept the device on the server side when prompted
5. Add the `content/` folder as a shared folder on either side, share it with the other device
6. On your local machine, set the folder path to your Obsidian vault location

Edits in Obsidian will sync to the server within seconds. Run `npx quartz build --serve` on the server to get live hot-reload as files change.
