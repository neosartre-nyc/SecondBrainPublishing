# Workspace Setup Instructions

Using VS Code (or VSCodium) as a second brain is a powerful choice. It gives you full control over your markdown ecosystem, version control, scripting, and publishing pipeline without vendor lock-in. The key to making it work is **separating raw capture from structured output** and automating the "combine → publish" step.

Here’s a battle-tested workspace structure and workflow tailored for multi-channel publishing (Online, Ebook, PDF).

---

## 📁 Recommended Workspace Structure

```
second-brain/
├── 01-Inbox/               # Raw notes, drafts, unprocessed research
├── 02-Archive/             # Completed, reference, or low-priority content
├── 03-Projects/             # Active assembly projects (e.g., blog, ebook)
│   ├── project-a/          # Each project is self-contained
│   └── project-b/
├── 04-Published/             # Immutable final outputs (never edit here)
│   ├── online/             # HTML, static assets, blog posts
│   ├── ebook/              # EPUB/MOBI/HTML books
│   └── pdf/                # PDFs, print-ready files
├── 05-Assets/               # Images, diagrams, fonts, covers (shared)
├── 06-Scripts/              # Build & publish automation
│   ├── build-online.js     # or .py, .sh
│   ├── build-ebook.js
│   └── combine-notes.js
├── 07-Config/               # Templates, frontmatter examples, channel configs
├── .gitignore              # Exclude 04-Published/, node_modules, etc.
└── README.md               # Workspace usage guide & pipeline docs
```

**Why this works:**

- `01-Inbox` & `02-Archive` keep your capture flow frictionless.
- `03-Projects` isolates active publishing work so you don’t accidentally merge unrelated notes.
- `04-Published` is read-only in practice. You only write here via scripts.
- `05-Assets` prevents duplication and keeps markdown clean (use relative paths like `![fig](../05-Assets/chart.png)`).
- `06-Scripts` + `07-Config` make your pipeline reproducible and maintainable.

---

## 🏷️ Metadata System (YAML Frontmatter)

Every `.md` file should start with YAML frontmatter. This is your routing mechanism for combining & publishing.

```yaml
---
title: "Attention Span in Deep Work"
tags: [research, psychology, productivity]
status: draft | review | ready
channels: [blog, ebook]          # Where this content should appear
created: 2024-01-15
updated: 2024-06-10
project: project-a               # Maps to /03-Projects/project-a/
aliases: ["focus", "deep-work"]   # For internal linking & search
---
# Content starts here...
```

**Rules:**

- Keep source markdown **channel-agnostic**. No HTML/CSS/EPUB-specific markup in your notes.
- Use `channels` and `project` to drive automation.
- Store channel-specific formatting, TOC, cover images, and metadata in `07-Config` or project root.

---

## 🔀 Edit → Combine → Publish Workflow

### 1. Editing

- Use VS Code extensions: `Markdown Preview`, `GitLens`, `YAML support`, `Code Snippets`
- Enable `workspace.exclude: ["**/04-Published/**", "**/*.log"]` to keep UI responsive
- Use `[[Internal Link]]` syntax if you want Obsidian-style linking, or stick to relative paths

### 2. Combining

You need a script that:

- Reads markdown files matching `channels` or `project` tags
- Applies a template (HTML, EPUB structure, etc.)
- Generates a Table of Contents & cross-links
- Outputs to `04-Published/`

**Tooling options:**

| Approach | Best For | Setup Complexity |
| ---------- | ---------- | ------------------ |
| `pandoc` CLI + shell scripts | PDF/EPUB, minimal code | Low |
| Node.js (`markdown-it`, `jsdoc2type`-style) | Custom pipeline, web/ebook/PDF | Medium |
| Static Site Generator (Hugo/Jekyll) | Blog/online-first, auto TOC, SEO | Medium-High |
| Custom Python (`python-markdown`, `jinja2`) | Research-heavy, data export | Medium |

**Example `combine-notes.js` (Node.js + markdown-it):**

```js
// Simplified concept: reads ready notes, builds HTML with TOC
const fs = require('fs');
const path = require('path');
const { createMarkdownParser } = require('markdown-it').require('markdown-it');

function buildProject(projectPath, outputDir) {
  const notes = fs.readdirSync(path.join(projectPath))
    .filter(f => f.endsWith('.md'))
    .sort();

  const htmlParts = notes.map(f => {
    const raw = fs.readFileSync(path.join(projectPath, f), 'utf8');
    // Extract frontmatter manually or use a parser like yaml-frontmatter
    const title = extractTitle(raw); 
    return `<article><h2>${title}</h2>\n${raw}\n</article>`;
  }).join('\n\n');

  const output = `<!DOCTYPE html><html><head><title>${path.basename(projectPath)}</title>...</head><body>\n${htmlParts}\n</body></html>`;
  fs.writeFileSync(path.join(outputDir, 'index.html'), output);
}
```

### 3. Publishing per Channel

- **Online**: Serve HTML from `04-Published/online/`. Push to Vercel/Netlify/GitHub Pages. Add `robots.txt`, sitemap, Open Graph tags in build step.
- **Ebook**: Use `pandoc -f markdown -t epub input.md --metadata cover=05-Assets/cover.jpg -o 04-Published/ebook/book.epub`
- **PDF**: `pandoc -f markdown -t html --pdf-xlatex=pdftexlive-x texmymacbook -o 04-Published/pdf/book.pdf` or convert HTML with `puppeteer`/`wpdf` for print-ready CSS.

---

## ⚙️ VS Code Automation Setup

Add to `tasks.json` (`.vscode/tasks.json`):

```json
{
  "version": "2.0.0",
  "tasks": [
    {
      "label": "Build & Publish Online",
      "command": "node 06-Scripts/build-online.js",
      "isBackground": true,
      "problemMatcher": ["$none"]
    },
    {
      "label": "Build Ebook (EPUB)",
      "command": "pandoc -f markdown -t epub 03-Projects/ebook-draft/*.md --metadata cover=05-Assets/cover.jpg -o 04-Published/ebook/draft.epub",
      "isBackground": true,
      "problemMatcher": ["$none"]
    },
    {
      "label": "Build PDF",
      "command": "pandoc -f markdown -t html --pdf-xlatex=pdftexlive-x 03-Projects/ebook-draft/*.md -o 04-Published/pdf/draft.pdf",
      "isBackground": true,
      "problemMatcher": ["$none"]
    }
  ]
}
```

Run via `Ctrl+Shift+P` → `Tasks: Run Task`.

---

## 📈 Scalability & Best Practices

1. **Never edit `04-Published/` directly**. It’s a build artifact.
2. **Use Git effectively**: Branch for major ebook revisions (`/dev/ebook-v2`). Tag releases (`v1.0`, `publish-2024`).
3. **Keep markdown lean**: No inline CSS, no channel-specific markup. Let the build step handle formatting.
4. **Performance**: If you hit 10k+ files, split into sub-repos or use `.gitignore` + `workspace.exclude`. VS Code isn’t a database; it’s an editor.
5. **Backup**: Git + automated external backup (Syncthing, rclone to S3).
6. **Search**: Use `grep` or VS Code’s Multi-Cursor search on YAML fields: `files: **/*.md | grep "channels.*blog"`

---

## 🔜 Next Steps / Customization Questions

To tailor this further, tell me:

1. Which channel are you publishing **first**? (Blog, Ebook, PDF?)
2. Do you need **internal linking** (`[[ ]]`), or are raw relative paths fine?
3. Are you open to using a lightweight SSG (Hugo, Eleventy), or prefer pure scripts?
4. Do you want **versioned drafts** (e.g., `draft-v1.md`, `v2.md`) or linear overwrite?

I can generate a ready-to-paste `.vscode` workspace config, a complete Node/Python build script, or a Hugo/Pandoc pipeline based on your answer.

Integrating **Astro** with a second-brain markdown workflow is a smart move. Astro's native markdown/MDX support, automatic routing, and static output align perfectly with a script-driven publish pipeline. The key is to keep your **second-brain as the single source of truth**, and use lightweight automation to feed Astro for `online`, while routing the same files to ebook/PDF builders.

Here’s a production-ready architecture optimized for Astro + multi-channel publishing.

---

## 📁 Workspace Structure (Astro-Optimized)

```
second-brain/
├── content/                      # Single source of truth (never edit Astro's output dir directly)
│   ├── 01-Inbox/                 # Raw capture, drafts, untagged research
│   ├── 02-Archive/               # Completed, reference, low-priority
│   ├── 03-Projects/              # Active publishing buckets
│   │   ├── blog/                 # Articles, essays, research notes
│   │   └── ebook-v1/             # Book chapters (ordered by filename)
│   └── 04-Published/              # Build artifacts (gitignored)
├── astro-site/                   # Your Astro project (can be separate repo or same)
│   ├── src/
│   │   └── content/              # Sync target for online content only
│   ├── ...
├── scripts/                      # Build & publish automation
│   ├── sync-astro.js             # Filters, merges frontmatter, copies to astro-site/src/content/
│   ├── build-ebook.js            # Pandoc/Node pipeline for EPUB/PDF
│   └── full-publish.sh           # Runs sync → astro build → ebook/pandoc
├── .vscode/                      # Workspace config, tasks, snippets
└── README.md                     # Pipeline docs & channel rules
```

**Why this works with Astro:**

- `content/03-Projects/blog/` holds your markdown. Only files with `status: ready` & `channels: [online]` get synced to `astro-site/src/content/`.
- Astro gets a clean, structured content directory. No drafts, no inbox clutter.
- Same markdown files feed ebook/PDF builders via `scripts/build-ebook.js`.

---

## 🔗 The Sync → Build Pipeline

## 1. `scripts/sync-astro.js`

```js
const fs = require('fs');
const path = require('path');

const SOURCE_DIR = path.resolve(__dirname, '../../content/03-Projects/blog');
const DEST_DIR  = path.resolve(__dirname, '../astro-site/src/content/blog');

function sync() {
  if (!fs.existsSync(DEST_DIR)) fs.mkdirSync(DEST_DIR, { recursive: true });

  const files = fs.readdirSync(SOURCE_DIR).filter(f => f.endsWith('.md'));
  
  for (const file of files) {
    const srcPath = path.join(SOURCE_DIR, file);
    let mdContent = fs.readFileSync(srcPath, 'utf8');

    // Parse frontmatter (simple regex; for production use `yaml-frontmatter` or `md-yaml`)
    const match = mdContent.match(/^---\n([^-----]+)\n---/s);
    if (!match) {
      console.warn(`⚠️ Skipping ${file}: missing frontmatter`);
      continue;
    }

    const frontmatter = YAML.parse(match[1]); // requires `npm i yaml`
    
    // Filter by status & channels
    if (frontmatter.status !== 'ready' || !frontmatter.channels?.includes('online')) {
      console.log(`⏭️  Skipped ${file}: not ready/online`);
      continue;
    }

    // Inject Astro-specific fields if missing
    const astroMeta = {
      layout: '_layout.md',
      seo: { title: frontmatter.title, description: frontmatter.description || '' }
    };

    // Merge & rewrite frontmatter
    const merged = YAML.stringify({ ...frontmatter, ...astroMeta });
    const outContent = `---\n${merged}\n---\n\n${mdContent.split(/\n---/).slice(2).join('\n---')}`;

    fs.writeFileSync(path.join(DEST_DIR, file), outContent);
    console.log(`✅ Synced: ${file}`);
  }
}

sync();
```

*Note: Install `yaml` via `npm i yaml`. For production, consider `md-yaml` or `frontmatter-parser`.*

## 2. `scripts/build-ebook.js`

```js
const { execSync } = require('child_process');

function buildEbook(projectPath, outputDir) {
  const files = fs.readdirSync(projectPath).filter(f => f.endsWith('.md'));
  
  // Combine into single markdown with headings preserved for TOC
  const combined = files.map(f => {
    const raw = fs.readFileSync(path.join(projectPath, f), 'utf8');
    // Add section break if not last file
    return raw + '\n---\n\n';
  }).join('');

  const outPath = path.join(outputDir, 'draft.epub');
  
  // Pandoc command (install pandoc: brew install pandoc / apt get install pandoc)
  execSync(`pandoc -f gfm-md -t epub \
    --metadata title="My Second Brain Book" \
    --metadata author="You" \
    --metadata cover="${path.join(__dirname, '../../05-Assets/cover.jpg')}" \
    -o "${outPath}" - <(echo '${combined}')`);

  // PDF version
  execSync(`pandoc -f gfm-md -t html --pdf-xlatex=pdftexlive-x \
    --metadata title="My Second Brain Book" \
    -o "${path.join(outputDir, 'draft.pdf')}" - <(echo '${combined}')`);

  console.log('✅ Ebook & PDF built');
}

buildEbook(path.resolve(__dirname, '../../content/03-Projects/ebook-v1'), path.resolve(__dirname, '../../04-Published/ebook'));
```

## 3. `scripts/full-publish.sh`

```bash
#!/bin/bash
cd "$(dirname "$0")/.."

echo "🔄 Syncing to Astro..."
node scripts/sync-astro.js

echo "🚀 Building Astro site..."
cd astro-site && npm run build && cd ".."

echo "📚 Building Ebook & PDF..."
node scripts/build-ebook.js

echo "✅ Publish complete. Outputs in 04-Published/ & astro-site/dist/"
```

---

## ⚙️ VS Code Workspace Config (`.vscode/workspace.json`)

```json
{
  "folders": [
    { "path": "./content" },
    { "path": "./astro-site", "settings": { "files.exclude": { "**/dist/**": true } } }
  ],
  "tasks": [
    {
      "label": "🔄 Sync to Astro",
      "command": "node scripts/sync-astro.js",
      "isBackground": true,
      "problemMatcher": ["$none"]
    },
    {
      "label": "🚀 Build Astro Site",
      "command": "npm --prefix astro-site run build",
      "isBackground": true,
      "problemMatcher": ["$none"]
    },
    {
      "label": "📚 Build Ebook & PDF",
      "command": "node scripts/build-ebook.js",
      "isBackground": true,
      "problemMatcher": ["$none"]
    },
    {
      "label": "🌐 Full Publish",
      "command": "bash scripts/full-publish.sh",
      "isBackground": true,
      "problemMatcher": ["$none"],
      "dependsOn": ["🔄 Sync to Astro", "🚀 Build Astro Site", "📚 Build Ebook & PDF"]
    }
  ],
  "settings": {
    "files.exclude": { "**/04-Published/**": true, "**/astro-site/dist/**": true }
  }
}
```

---

## 🌐 Astro-Specific Tips

1. **Routing**: Astro auto-routes `src/content/blog/post-title.md` → `/blog/post-title`. Keep folder structure flat or use subfolders for categories.
2. **Frontmatter Defaults**: In `astro.config.mjs`, add:

   ```js
   export default defineConfig({
     site: 'https://yourdomain.com',
     // ...
   });
   ```

3. **MDX vs Markdown**: If you need inline code blocks, components, or dynamic data in notes, switch to `.mdx`. Astro supports it natively.
4. **Local Dev Workflow**:
   - Run `sync-astro.js` once → open Astro dev server (`npm --prefix astro-site run dev`)
   - For auto-sync on save, use a watcher script or `astro watch` with a pre-build hook.
5. **SEO & Metadata**: Keep `seo.title`, `seo.description` in frontmatter. Astro's `<seo />` component will consume it.

---

## 🔁 Recommended Daily Workflow

1. **Capture**: Drop notes into `content/01-Inbox/`. Tag with YAML frontmatter.
2. **Process**: Move to `03-Projects/blog/` or `/ebook-v1/`. Set `status: ready`, `channels: [online]` (or `[ebook, pdf]`).
3. **Publish**: Run `🌐 Full Publish` task. Astro builds online, Pandoc generates ebook/PDF.
4. **Deploy**: Push `astro-site/dist/` to Vercel/Netlify/GitHub Pages. Export ebook/PDF manually or automate via CI.

---

## 🔜 Next Steps / Customization

Tell me:

1. Are you publishing **static blog posts** or **dynamic collections/categories**?
2. Do you want **versioned drafts** (e.g., `chapter-01-v1.md`) or linear overwrite?
3. Should I generate a ready-to-paste `astro.config.mjs` + `_layout.md` template optimized for research notes?
4. Prefer Node, Python, or shell for the pipeline? (I can adapt scripts accordingly)

I’ll tailor the sync logic, Astro routing, and ebook packaging to your exact stack.
