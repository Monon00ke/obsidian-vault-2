# Monon00ke's Digital Garden - Setup Guide

## Repository Info
- **URL**: https://github.com/Monon00ke/obsidian-vault
- **Site**: https://monon00ke.github.io/obsidian-vault/
- **Branch**: main

## Quick Start (New Session)

### 1. Clone Repository
```bash
git clone https://github.com/Monon00ke/obsidian-vault.git
cd obsidian-vault
```

### 2. Install Dependencies
```bash
npm install
```

### 3. Make Changes
Edit files in `content/` directory:
- `content/English/` - English notes
- `content/Math/` - Math notes
- `content/OS Labs/` - OS lab works
- `content/Functional Analysis/` - Functional Analysis
- `content/Scientific Work/` - Scientific work
- `content/Photos/` - Images

### 4. Build & Test Locally
```bash
npx quartz build
npx quartz serve
```
Open http://localhost:8080 to preview

### 5. Deploy to GitHub Pages
```bash
git add .
git commit -m "Your commit message"
git push
```
GitHub Actions will automatically build and deploy (~2-3 minutes)

## Configuration Files

### quartz.config.ts
Key settings:
- `pageTitle`: "Monon00ke's Digital Garden"
- `baseUrl`: "monon00ke.github.io/obsidian-vault"
- `locale`: "ru-RU"
- `markdownLinkResolution`: "relative"

### quartz.layout.ts
Components:
- Left sidebar: Explorer, Search, Dark mode
- Right sidebar: Graph, Table of Contents, Backlinks
- SPA enabled for smooth navigation

## File Structure
```
obsidian-vault/
├── content/              # All your notes
│   ├── index.md          # Homepage
│   ├── English/
│   ├── Math/
│   ├── OS Labs/
│   ├── Functional Analysis/
│   ├── Scientific Work/
│   └── Photos/
├── quartz.config.ts      # Site configuration
├── quartz.layout.ts      # Page layout
├── .github/workflows/
│   └── deploy.yaml       # GitHub Actions deploy
── 404.html              # SPA fallback
```

## Important Notes

### Link Format
Use Obsidian wikilinks: `[[File Name|Display Text]]`
- Links resolve relative to current file
- No need for `.md` extension in links

### Front Matter
Each note should have:
```markdown
---
title: "Note Title"
---
```

### Images
- Store in `content/Photos/` or same folder as note
- Reference: `![[Pasted image 20260518192021.png]]`

### Backlinks
Add at end of each file:
```markdown
---
## Backlinks

- [[00 OS Labs MOC]]
- [[Related Note]]
```

## GitHub Authentication
For pushing changes:
```bash
# Option 1: Use GitHub CLI
gh auth login

# Option 2: Use token (replace with your token)
git remote set-url origin https://Monon00ke:YOUR_TOKEN@github.com/Monon00ke/obsidian-vault.git
```

## Troubleshooting

### 404 Errors
- Check `baseUrl` in quartz.config.ts (should NOT have https://)
- Clear browser cache (Ctrl+Shift+R)
- Wait 2-3 minutes after push for rebuild

### Links Not Working
- Use `[[File Name]]` format (wikilinks)
- Check filename matches exactly (case-sensitive)
- Run `npx quartz build` to verify

### Build Fails
- Check for YAML syntax errors in front matter
- Ensure all required files exist
- Check GitHub Actions logs: https://github.com/Monon00ke/obsidian-vault/actions

## Custom Domain (Optional)
To use custom domain:
1. Add CNAME file to root
2. Update `baseUrl` in quartz.config.ts
3. Configure DNS settings

## Backup
Regular backups recommended:
```bash
git pull
git push
```

---
Last updated: 2026-05-22
