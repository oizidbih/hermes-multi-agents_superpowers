# Travel2Adrar Mockup Session — Worked Example

## Context

- Client: Ahmed A
- Project: Travel2Adrar travel agency website (Algeria)
- Budget: under $5K
- Approach: Static site (Git+Markdown CMS), PWA
- Pages: Home, About, Packages, Gallery, Contact, Blog, FAQ

## What Went Well

- **Interactive mockup** with 7 tabbed pages helped the user visualize before committing to architecture
- **Single nav bar** replaced an earlier dual-nav layout based on user feedback
- **Colour palette matching** — user pointed to https://www.wildfrontierstravel.com/en_GB/destination/algeria as inspiration; extracted terracotta (#BF6740), gold (#E5B567), cream (#F4EBDC) and rebuilt the mockup
- **Live filter demo** — user asked why filters weren't active; added `data-category` attributes + JS filter functions so clicking "Desert" or "Mountains" actually filtered the cards

## What Went Wrong

### Cloudflare tunnel lifecycle

The quick tunnel expired between user reviews. Restarting required:
1. Server restart → new port + new content directory
2. Re-writing the mockup HTML
3. New tunnel → new URL

**The bug:** When the server restarted, the content copied from `/tmp/travel2adrar-mockup.html` was the **v1 backup**, not the latest v2. The user saw old content and thought updates hadn't been applied.

**Fix:** Always write the mockup fresh to the new screen_dir after restart. Never copy from a stale backup.

### Background shell messages

Cloudflared background messages like `[1] 698134` confused the user into thinking the tunnel died. These are normal — the wrapper shell exits but cloudflared keeps running. Verify with `curl` before telling the user.

## Key Commands Used

```bash
# Start server
cd ~/.hermes/profiles/productlead/skills/brainstorming/scripts && bash start-server.sh --background
# Output: {"port":54426,"url":"http://localhost:54426","screen_dir":"/tmp/brainstorm-xxx/content/"}

# Write mockup to screen_dir
# Tunnel
cloudflared tunnel --url http://localhost:54426 &>/tmp/cloudflared.log &
sleep 10 && grep "trycloudflare" /tmp/cloudflared.log | tail -1
# Output: https://xxx.trycloudflare.com
```

## Reference Sites Used

- Wild Frontiers Algeria page: https://www.wildfrontierstravel.com/en_GB/destination/algeria
- Used browser_console + CSS inspection to extract: terracotta (#BF6740), gold (#E5B567), cream (#F4EBDC), Playfair Display serif headings