# Website Footer Guideline for iOS Apps

This guideline is based on the implementation in:

- `EVChargingTracker/docs/index.html` (canonical, fully reviewed)

It defines a reusable standard for the landing page footer across all iOS app websites in this workspace.

---

## 1) Purpose

Provide a consistent, branded footer across all app landing pages with project identity, cross-promotion links, and GitHub resources. The footer follows the same "soft brutalism" design language used throughout each site.

---

## 2) Layout

Three-column grid with **2:1:1** proportions. Stacks to single column on mobile (below 860px).

```
+-----------------------------+-------------+-------------+
|  Brand (2fr)                | Links (1fr) | Links (1fr) |
|                             |             |             |
|  [App Icon]                 | Our projects| Github      |
|  Slogan text                | - Link 1    | - Repository|
|  Copyright line             | - Link 2    | - Privacy   |
|                             | - Link 3    | - Issues    |
|                             | - Link 4    | - License   |
|                             | - Link 5    |             |
+-----------------------------+-------------+-------------+
```

---

## 3) Column Content

### Column 1 — Brand (2fr)

| Element | Details |
|---------|---------|
| App icon | Same `app_icon.png` from `docs/assets/`, 56x56px, rounded corners, bordered |
| Slogan | One-line app tagline, muted color, semibold |
| Copyright | `© {year} Made with ❤️, coffee and claude by Maxim Gorbatyuk (link)` |

The author link must include UTM parameters:
```
https://mgorbatyuk.dev?utm_source={app-slug}&utm_medium=referral&utm_campaign=footer
```

The year uses a `<span id="current-year">` updated dynamically via JavaScript.

### Column 2 — Our Projects (1fr)

| Element | Details |
|---------|---------|
| Header | `<h4>Our projects</h4>` |
| Links | List of cross-promotion links to other apps/projects |

Links should point to the landing pages of other apps in this workspace. Use `#` as placeholder for unreleased projects.

### Column 3 — Github (1fr)

| Element | Details |
|---------|---------|
| Header | `<h4>Github</h4>` |
| Links | Repository, Privacy policy, Issues, License |

| Link | Target |
|------|--------|
| Repository | `https://github.com/maximgorbatyuk/{repo-name}` |
| Privacy policy | `./privacy-policy/` (relative, same site) |
| Issues | `https://github.com/maximgorbatyuk/{repo-name}/issues` |
| License | `https://github.com/maximgorbatyuk/{repo-name}/blob/main/LICENSE` |

All GitHub links open in `target="_blank"`. Privacy policy is a same-site relative link.

---

## 4) HTML Template

```html
<section class="footer">
  <div class="footer-grid">
    <div class="footer-brand">
      <img class="footer-logo" src="./assets/app_icon.png" alt="{App Name}" />
      <p class="footer-slogan">{App slogan}</p>
      <p class="footer-copy">&copy; <span id="current-year">2026</span> Made with &#10084;&#65039;, coffee and claude by <a
          href="https://mgorbatyuk.dev?utm_source={app-slug}&utm_medium=referral&utm_campaign=footer"
          target="_blank">Maxim Gorbatyuk</a></p>
    </div>

    <div class="footer-links">
      <h4>Our projects</h4>
      <ul>
        <li><a href="{url}">Project 1</a></li>
        <li><a href="{url}">Project 2</a></li>
        <li><a href="{url}">Project 3</a></li>
        <li><a href="{url}">Project 4</a></li>
        <li><a href="{url}">Project 5</a></li>
      </ul>
    </div>

    <div class="footer-links">
      <h4>Github</h4>
      <ul>
        <li><a href="https://github.com/maximgorbatyuk/{repo-name}" target="_blank">Repository</a></li>
        <li><a href="./privacy-policy/">Privacy policy</a></li>
        <li><a href="https://github.com/maximgorbatyuk/{repo-name}/issues" target="_blank">Issues</a></li>
        <li><a href="https://github.com/maximgorbatyuk/{repo-name}/blob/main/LICENSE" target="_blank">License</a></li>
      </ul>
    </div>
  </div>
</section>
```

**Placeholders to replace per app:**
- `{App Name}` — Display name for alt text (e.g., "EV Charge Tracker")
- `{App slogan}` — One-line tagline (e.g., "Track your EV car costs with precision")
- `{app-slug}` — URL-safe app identifier for UTM (e.g., "ev-charging-tracker")
- `{repo-name}` — GitHub repository name (e.g., "ev-charging-tracker")
- `{url}` — Cross-promotion links to other project landing pages

**JavaScript requirement:** The page must include the dynamic year updater:
```javascript
document.getElementById('current-year').textContent = new Date().getFullYear();
```

---

## 5) CSS

All styles use CSS variables from the shared `brutalism-style.css`. Each app has its own color palette, but the variable names are consistent.

```css
.footer {
  margin-top: 32px;
  padding: 32px 28px;
  border: 2px solid var(--stroke);
  border-radius: 18px;
  background: var(--card);
  box-shadow: var(--shadow);
  font-family: "IBM Plex Mono", monospace;
  font-size: 0.85rem;
}

.footer a {
  color: inherit;
  text-decoration: underline;
}

.footer a:hover {
  color: var(--call-to-action-dark);
}

.footer-grid {
  display: grid;
  grid-template-columns: 2fr 1fr 1fr;
  gap: 32px;
}

.footer-brand {
  display: flex;
  flex-direction: column;
  gap: 4px;
}

.footer-logo {
  width: 56px;
  height: 56px;
  border-radius: 12px;
  border: 2px solid var(--stroke);
}

.footer-slogan {
  margin: 8px 0 0;
  color: var(--muted);
  font-size: 0.9rem;
  font-weight: 600;
}

.footer-copy {
  margin: 12px 0 0;
  color: var(--muted);
  font-size: 0.78rem;
}

.footer-links h4 {
  margin: 0 0 12px;
  font-size: 0.9rem;
  font-weight: 700;
  color: var(--ink);
}

.footer-links ul {
  list-style: none;
  margin: 0;
  padding: 0;
  display: flex;
  flex-direction: column;
  gap: 8px;
}

.footer-links ul li a {
  color: var(--muted);
  text-decoration: none;
  font-size: 0.85rem;
  transition: color 0.2s ease;
}

.footer-links ul li a:hover {
  color: var(--call-to-action-dark);
  text-decoration: underline;
}

@media (max-width: 860px) {
  .footer-grid {
    grid-template-columns: 1fr;
    gap: 24px;
  }
}
```

---

## 6) Design Rules

| Rule | Details |
|------|---------|
| Container style | Card with border, shadow, and rounded corners — matches other page sections |
| Font | IBM Plex Mono throughout (monospace, consistent with badges and code elements) |
| Link hover color | Uses `--call-to-action-dark` from each app's palette |
| Link columns | No bullets, no underline by default, underline on hover |
| Brand column | Icon + slogan + copyright stacked vertically with small gaps |
| Mobile breakpoint | 860px — same breakpoint as the showcase section |
| Mobile layout | Single column, brand first, then link columns stacked |

---

## 7) Common Mistakes to Avoid

| Mistake | Why it's wrong | Correct approach |
|---------|---------------|-----------------|
| Hardcoding the year | Will be stale after December | Use `<span id="current-year">` + JS updater |
| Missing UTM parameters on author link | Cannot track which app drives traffic | Always include `utm_source`, `utm_medium`, `utm_campaign` |
| Using `list-style: disc` for link lists | Looks cluttered in monospace footer | Use `list-style: none` |
| Forgetting `target="_blank"` on GitHub links | External links should open in new tab | Add `target="_blank"` to all GitHub/external links |
| Privacy policy as external link | Should be same-site relative for fast load | Use `./privacy-policy/` relative path |
| Different icon sizes across apps | Visual inconsistency | Always 56x56px with 12px border-radius |

---

## 8) Checklist for New App Implementation

- [ ] Add `<section class="footer">` with the 3-column grid structure from the template
- [ ] Replace all placeholders: `{App Name}`, `{App slogan}`, `{app-slug}`, `{repo-name}`
- [ ] Set correct cross-promotion links in "Our projects" column (use `#` for unreleased)
- [ ] Verify `app_icon.png` exists in `docs/assets/`
- [ ] Add footer CSS block to `brutalism-style.css` (copy from section 5)
- [ ] Verify dynamic year JS is present: `document.getElementById('current-year').textContent = new Date().getFullYear()`
- [ ] Test responsive layout at 860px breakpoint
- [ ] Verify all GitHub links point to the correct repository
- [ ] Verify privacy policy relative link works

---

## 9) Alignment Tasks for Existing Apps

### Journey Wallet (`journey-wallet/docs/index.html`)

- [ ] Replace existing footer with the 3-column grid template
- [ ] Add footer CSS to `brutalism-style.css`
- [ ] Set slogan, UTM slug to `journey-wallet`, repo to `journey-wallet`
- [ ] Populate "Our projects" with cross-links to other app landing pages

### CreativityHub (`CreativityHub/docs/index.html`)

- [ ] Replace existing footer with the 3-column grid template
- [ ] Add footer CSS to `brutalism-style.css`
- [ ] Set slogan, UTM slug to `creativityhub`, repo to `creativityhub`
- [ ] Populate "Our projects" with cross-links to other app landing pages
