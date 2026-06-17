# Design Tools & Techniques

Quick-reference patterns for building mockups from reference sites and pushing to GitHub.

## Reference Site Colour Extraction

When a user provides a URL as a "design inspiration" reference, extract the colour palette programmatically rather than guessing:

```javascript
// Run in browser_console() on the reference page
// Get key design tokens
JSON.stringify({
  bg: getComputedStyle(document.body).backgroundColor,
  text: getComputedStyle(document.body).color,
  links: document.querySelector('a') ? getComputedStyle(document.querySelector('a')).color : 'n/a',
})

// Get more specific elements
(function() {
  const els = document.querySelectorAll('h1, h2, h3, .btn, [class*="cta"], nav a, [class*="accent"], [class*="primary"]');
  const styles = {};
  els.forEach(el => {
    styles[el.tagName + '.' + (el.className || '').substring(0,40)] = {
      color: getComputedStyle(el).color,
      bg: getComputedStyle(el).backgroundColor,
      font: getComputedStyle(el).fontFamily,
      weight: getComputedStyle(el).fontWeight
    };
  });
  return JSON.stringify(styles, null, 2);
})()
```

Map the extracted colours to CSS variable names for the mockup:

| Source colour | Mockup variable | Example |
|---|---|---|
| Primary link/button colour | `--terracotta` / `--primary` | `#BF6740` |
| Gold/amber accent | `--gold` / `--accent` | `#E5B567` |
| Cream background | `--cream` / `--bg` | `#F4EBDC` |
| Dark text | `--dark` / `--text` | `#212121` |

Also inspect typography — serif vs sans-serif, font families, weight, letter-spacing.

## Colour Palette Communication

When presenting updated mockups to users, name the palette shift explicitly and connect it to the reference:

```md
| Before | After | Reference |
|---|---|---|
| Navy (#1a3c5e) + gold (#c8a45c) | Terracotta (#BF6740) + warm gold (#E5B567) | Wild Frontiers |
| System sans-serif | Playfair Display (serif headings) + Inter (body) | Wild Frontiers |
| Blue gradient hero | Rust/terracotta gradient hero | Wild Frontiers |
```

## SSH Deploy Key + Git Push

When pushing mockups to a GitHub repo using an SSH deploy key (the secure alternative to a long-lived PAT):

### SSH Config Entry

Create an SSH host alias in `~/.ssh/config` so git uses the correct key:

```text
Host github.com-<project>
    HostName github.com
    User git
    IdentityFile /root/.ssh/<project>_deploy
    IdentitiesOnly yes
```

### Git Remote

Set the remote to use the host alias:

```bash
git remote set-url origin git@github.com-<project>:<owner>/<repo>.git
```

### Push

```bash
git push -u origin main
```

### When Remote Has Commits Already

If the remote repo was already initialized (e.g. by the user via browser), you'll get a `[rejected] main -> main (fetch first)` error. Handle it:

```bash
# Option 1: rebase (preserves remote history)
git pull --rebase origin main  # Resolve conflicts if any
git push origin main

# Option 2: reset to remote + apply your changes (cleaner when your local is the definitive version)
git fetch origin main
git reset --hard origin/main
# Then copy your approved files over
cp <approved-mockup> index.html
git add -A && git commit -m "Update to approved v2: <changes>"
git push origin main
```

## Multi-file Project Commit

When committing logos, PWA icons, and other binary assets alongside the mockup:

```bash
# Stage everything
git add -A

# Verify what you're committing before pushing
git status --short

# Commit with a descriptive message listing each asset category
git commit -m "Add branded assets: Direction A SVG, favicon set, PWA icons, OG card"

# Push
git push origin main
```

Binary files (PNG, SVG) are tracked normally in git. Watch the repo size if the images are large — optimise PNGs before committing if needed.