# 🧭 Mega Dashboard Scripts — Creator Guide and Safe Patterns

This guide is the canonical reference to create Dashboard Scripts for Mega with:

- Safe DOM hooks.
- Idempotent behavior.
- Low-fragility runtime strategies.
- Clear guidance on when to use Dashboard Script vs Dashboard App.

It also complements the reusable AI skill stored in `skills/skills.md`.

---

## 🌐 Languages

- English: `README.md`
- Portuguese (Brazil): `README-PT-BR.md`
- Spanish: `README-ES.md`

---

## 🔐 Accessing Dashboard Scripts

To add a script, log in to the Mega Super Admin panel:

- Visit (replace `your-domain.com`):

  ```text
   https://your-domain.com/super_admin/app_config?config=internal
   ```

- Find **Dashboard Scripts**.
- Paste your script.
- Click **Save** and reload the dashboard.

---

## 🧠 AI Skill Location

The skill spec for AI agents is available at:

- `skills/skills.md`

Use this skill when the request is about:

- `crear dashboard script`
- `script para dashboard`
- `customizar dashboard`
- `super_admin/dashboard_scripts`
- `dashboard scripts`

---

## 🏗️ Codebase Facts You Must Respect

### Injection pipeline

- Dashboard scripts are assembled in `DashboardController#set_dashboard_scripts` from:
  - Active DB records (`DashboardScript.active.ordered.pluck(:content)`).
  - Global config (`DASHBOARD_SCRIPTS`).
- Final content is injected in `app/views/layouts/vueapp.html.erb` via `@dashboard_scripts.html_safe`.
- Scripts are skipped on sensitive paths by `DashboardController#sensitive_path?`.

### Scope mode: global or single-account

- Global mode:
  - Keep script active in Super Admin or use `DASHBOARD_SCRIPTS`.
  - Behavior runs for all accounts on dashboard routes.
- Single-account mode:
  - Injection is still global, so add runtime account guard in the script.
  - Read account from URL (`/app/accounts/:id/...`) and exit early if it does not match.
  - For strict per-account integrations, prefer `DashboardApp` records.

### Dashboard Script model rules

- Model: `app/models/dashboard_script.rb`.
- Required fields: `name`, `content`.
- `content` max length: `Limits::DASHBOARD_SCRIPT_CONTENT_MAX_LENGTH`.
- Only scripts with `active: true` are loaded.

### Super Admin editor and preview behavior

- Editor view: `app/views/super_admin/dashboard_scripts/show.html.erb`.
- New form: `app/views/super_admin/dashboard_scripts/new.html.erb`.
- Preview runs in iframe URL `/app/accounts/1/dashboard` with desktop/mobile toggles.
- Combined scripts add `data-name` on the first `<script` tag when payload is built.

### Alternative architecture when script is not ideal

- `DashboardApp` model: `app/models/dashboard_app.rb`.
- Content schema expects array items like `{ type: "frame", url: "https://..." }`.
- Frontend bridge in `DashboardApp/Frame.vue` posts `appContext` (conversation/contact/agent).
- Use Dashboard App for larger framed third-party modules.

---

## 🧭 Decision Matrix: Script vs App

Use **Dashboard Script** when:

- You need lightweight DOM customization.
- Scope is small/local (labels, markers, hide/show one control).
- You can implement idempotently with stable selectors.

Use **Dashboard App** when:

- You need a full external UI/module.
- You want stronger isolation from dashboard DOM changes.
- You need structured conversation/contact/agent context in frame bridge.
- You integrate a third-party product with independent release cycle.

---

## ✅ Required Workflow

### 1) Clarify target behavior

Capture:

- Which route/page should run.
- Trigger action or UI state.
- Desktop, mobile, or both.
- Scope mode (global or single-account).

### 2) Map stable anchors first

Selector priority:

1. Stable ids or data attributes.
2. Short class selectors with clear intent.
3. Structural selectors only as last resort.

Avoid long full-DOM chains whenever possible.

### 3) Build idempotent script

- Add global init guard key.
- Use `<script data-name="feature-name">`.
- Mark processed nodes with `dataset` flags.
- Keep side effects local.
- Add account guard for single-account mode.

### 4) Handle dynamic rendering safely

- Use `MutationObserver` only on required subtree.
- Disconnect observer when feasible.
- If interval fallback is needed, use low frequency plus stop conditions.

### 5) Add safety guards

- Check required elements before mutation.
- Gate mobile-only behavior by viewport.
- Avoid overriding critical actions unless explicitly requested.
- Avoid remote script injection unless requirement is explicit and trusted.

### 6) Validate with dashboard preview

- Validate in desktop and mobile preview.
- Confirm no duplicate UI on re-render.
- Confirm no console errors.

### 7) Deliver complete output

Provide:

- Final script snippet.
- Selector rationale.
- Runtime strategy (observer/guards).
- Known risks and fallback plan.
- Recommendation if Dashboard App is better.

---

## 🧱 Implementation Template (Preferred Baseline)

```html
<script data-name="feature-name">
  (function () {
    const SCRIPT_KEY = 'mega_dashboard_feature_name_v1';
    if (window[SCRIPT_KEY]) return;
    window[SCRIPT_KEY] = true;

    const TARGET_ACCOUNT_ID = null; // Example: 1 for single-account mode, null for global mode

    function getAccountIdFromPath() {
      const match = window.location.pathname.match(/\/app\/accounts\/(\d+)/);
      return match ? Number(match[1]) : null;
    }

    function isScopeAllowed() {
      if (TARGET_ACCOUNT_ID === null) return true;
      return getAccountIdFromPath() === TARGET_ACCOUNT_ID;
    }

    const isMobile = () => window.innerWidth <= 768;

    function applyFeature() {
      if (!isScopeAllowed()) return;

      const target = document.querySelector('[data-your-anchor]');
      if (!target) return;
      if (target.dataset.featureApplied === 'true') return;

      target.dataset.featureApplied = 'true';
      // Apply your scoped DOM change here.
    }

    function bootstrap() {
      applyFeature();

      const observer = new MutationObserver(() => {
        applyFeature();
      });

      observer.observe(document.body, {
        childList: true,
        subtree: true,
      });

      // Optional guard to avoid permanent observers when not needed.
      setTimeout(() => observer.disconnect(), 20000);

      window.addEventListener('resize', () => {
        // Keep lightweight; only run if feature depends on viewport.
        if (isMobile()) applyFeature();
      });
    }

    if (document.readyState === 'loading') {
      document.addEventListener('DOMContentLoaded', bootstrap, { once: true });
    } else {
      bootstrap();
    }
  })();
</script>
```

---

## 🎯 Reference Example: Sidebar Item with Embedded App

Use this production-ready example to add a custom sidebar item that opens an embedded iframe panel inside Mega.

### Configuration (`CFG` block)

```js
const CFG = {
  LINK_TEXT: 'My Application',
  LINK_ICON: '🧭',
  IFRAME_URL: 'https://www.google.com',
  TARGET_ACCOUNT_ID: null, // Set a number for single-account mode
  POSITION: {
    mode: 'afterLabel', // 'start' | 'end' | 'index' | 'afterLabel' | 'beforeLabel'
    label: 'Conversations',
    index: 0,
  },
  PADDING: 0,
  SHOW_BORDER: true,
  MOBILE_MAX_WIDTH: 768,
};
```

### Positioning modes

| `mode` | Description | Example |
| --- | --- | --- |
| `start` | Place at top of sidebar. | `{ mode: 'start' }` |
| `end` | Place at bottom (default). | `{ mode: 'end' }` |
| `index` | Place at exact index (0-based). | `{ mode: 'index', index: 3 }` |
| `afterLabel` | Insert after a visible menu label. | `{ mode: 'afterLabel', label: 'Conversations' }` |
| `beforeLabel` | Insert before a visible menu label. | `{ mode: 'beforeLabel', label: 'Campaigns' }` |

### Full JavaScript code

```html
<script data-name="sidebar-embed-app">
  (function () {
    const SCRIPT_KEY = 'mega_sidebar_embed_app_v2';
    if (window[SCRIPT_KEY]) return;
    window[SCRIPT_KEY] = true;

    const CFG = {
      LINK_TEXT: 'My Application',
      LINK_ICON: '🧭',
      IFRAME_URL: 'https://www.google.com',
      TARGET_ACCOUNT_ID: null,
      POSITION: { mode: 'afterLabel', label: 'Conversations', index: 0 },
      PADDING: 0,
      SHOW_BORDER: true,
      MOBILE_MAX_WIDTH: 768,
    };

    const $ = (s, r = document) => {
      try {
        return r.querySelector(s);
      } catch {
        return null;
      }
    };
    const $$ = (s, r = document) => {
      try {
        return Array.from(r.querySelectorAll(s));
      } catch {
        return [];
      }
    };
    const on = (el, ev, fn, opts) => {
      if (el) el.addEventListener(ev, fn, opts);
    };
    const normalize = value => (value || '').toLowerCase().replace(/\s+/g, ' ').trim();

    let panel = null;

    function getAccountIdFromPath() {
      const match = window.location.pathname.match(/\/app\/accounts\/(\d+)/);
      return match ? Number(match[1]) : null;
    }

    function isScopeAllowed() {
      if (CFG.TARGET_ACCOUNT_ID === null) return true;
      return getAccountIdFromPath() === CFG.TARGET_ACCOUNT_ID;
    }

    function findSidebar() {
      return $('[data-testid="sidebar-primary"]') || $('aside');
    }

    function findAnyList() {
      const sb = findSidebar();
      if (!sb) return null;
      return $('ul', sb);
    }

    function findListAndRefByLabel(labelWanted) {
      const sb = findSidebar();
      if (!sb) return { list: null, ref: null };

      const wanted = normalize(labelWanted);
      const uls = $$('ul', sb);
      for (const ul of uls) {
        const lis = $$(':scope > li', ul);
        for (const li of lis) {
          const text = normalize(li.textContent);
          if (text === wanted || text.includes(wanted)) {
            return { list: ul, ref: li };
          }
        }
      }
      return { list: null, ref: null };
    }

    function getThemeTokens() {
      const rootStyle = getComputedStyle(document.documentElement);
      const bodyStyle = getComputedStyle(document.body);

      return {
        background: rootStyle.getPropertyValue('--w-bg-color').trim() || bodyStyle.backgroundColor,
        text: rootStyle.getPropertyValue('--w-text-color').trim() || bodyStyle.color,
        border: rootStyle.getPropertyValue('--w-border-color').trim() || 'rgba(148, 163, 184, 0.35)',
      };
    }

    function applyPanelTheme() {
      if (!panel) return;
      const tokens = getThemeTokens();
      panel.style.background = tokens.background;
      panel.style.color = tokens.text;
      panel.style.padding = `${CFG.PADDING}px`;
      panel.style.borderLeft = CFG.SHOW_BORDER ? `1px solid ${tokens.border}` : '0';
    }

    function layoutPanel() {
      if (!panel || panel.dataset.visible !== 'true') return;
      const sb = findSidebar();
      if (!sb) return;

      const rect = sb.getBoundingClientRect();
      panel.style.left = `${Math.max(0, rect.right)}px`;
      panel.style.width = `${Math.max(320, window.innerWidth - rect.right)}px`;
    }

    function ensurePanel() {
      if (panel && document.body.contains(panel)) return panel;

      panel = document.createElement('section');
      panel.id = 'mega-embed-panel';
      panel.dataset.visible = 'false';
      panel.style.cssText = [
        'position:fixed',
        'top:0',
        'right:0',
        'bottom:0',
        'z-index:9',
        'display:none',
        'overflow:hidden',
      ].join(';');

      panel.innerHTML = '<iframe id="mega-embed-iframe" title="Embedded dashboard app" style="width:100%;height:100%;border:0;"></iframe>';
      document.body.appendChild(panel);

      applyPanelTheme();
      layoutPanel();
      return panel;
    }

    function hidePanel() {
      if (!panel) return;
      panel.dataset.visible = 'false';
      panel.style.display = 'none';

      const iframe = $('#mega-embed-iframe');
      if (iframe) iframe.src = 'about:blank';
    }

    function showPanel() {
      ensurePanel();
      const iframe = $('#mega-embed-iframe');
      if (iframe && iframe.src !== CFG.IFRAME_URL) iframe.src = CFG.IFRAME_URL;

      panel.dataset.visible = 'true';
      panel.style.display = 'block';
      applyPanelTheme();
      layoutPanel();
    }

    function ensureSidebarLink() {
      if ($('#mega-embed-link')) return;

      const li = document.createElement('li');
      li.id = 'mega-embed-link';
      li.style.listStyle = 'none';
      li.innerHTML = `
        <a href="#" data-mega-embed-link="true" style="display:flex;align-items:center;gap:10px;padding:8px 12px;border-radius:8px;text-decoration:none;color:inherit;">
          <span aria-hidden="true">${CFG.LINK_ICON}</span><span>${CFG.LINK_TEXT}</span>
        </a>
      `;

      const link = $('a', li);
      let parentList = null;
      let ref = null;

      if (CFG.POSITION.mode === 'afterLabel' || CFG.POSITION.mode === 'beforeLabel') {
        const found = findListAndRefByLabel(CFG.POSITION.label);
        parentList = found.list;
        ref = found.ref;
      }

      if (!parentList) parentList = findAnyList();
      if (!parentList) return;

      const items = $$(':scope > li', parentList);
      switch (CFG.POSITION.mode) {
        case 'start':
          parentList.insertBefore(li, items[0] || null);
          break;
        case 'index':
          parentList.insertBefore(li, items[Math.max(0, Math.min(CFG.POSITION.index, items.length))] || null);
          break;
        case 'afterLabel':
          if (ref && ref.nextSibling) parentList.insertBefore(li, ref.nextSibling);
          else parentList.appendChild(li);
          break;
        case 'beforeLabel':
          if (ref) parentList.insertBefore(li, ref);
          else parentList.insertBefore(li, items[0] || null);
          break;
        case 'end':
        default:
          parentList.appendChild(li);
      }

      on(link, 'click', event => {
        event.preventDefault();
        showPanel();
      });
    }

    function bindSidebarCloseHandler() {
      const sb = findSidebar();
      if (!sb || sb.dataset.embedCloseBound === 'true') return;
      sb.dataset.embedCloseBound = 'true';

      on(
        sb,
        'click',
        event => {
          const customLink = event.target && event.target.closest('[data-mega-embed-link="true"]');
          if (!customLink) hidePanel();
        },
        { capture: true, passive: true }
      );
    }

    function applyFeature() {
      if (!isScopeAllowed()) return;
      ensurePanel();
      ensureSidebarLink();
      bindSidebarCloseHandler();
      applyPanelTheme();
    }

    function bootstrap() {
      applyFeature();

      const observer = new MutationObserver(() => applyFeature());
      observer.observe(document.body, { childList: true, subtree: true });

      const themeObserver = new MutationObserver(() => applyPanelTheme());
      themeObserver.observe(document.documentElement, { attributes: true, attributeFilter: ['class', 'data-theme'] });
      themeObserver.observe(document.body, { attributes: true, attributeFilter: ['class', 'data-theme'] });

      on(window, 'resize', () => {
        if (window.innerWidth <= CFG.MOBILE_MAX_WIDTH) {
          hidePanel();
        } else {
          applyPanelTheme();
          layoutPanel();
        }
      });

      setTimeout(() => observer.disconnect(), 20000);
    }

    if (document.readyState === 'loading') {
      document.addEventListener('DOMContentLoaded', bootstrap, { once: true });
    } else {
      bootstrap();
    }
  })();
</script>
```

---

## ✅ Output Quality Checklist

Before finalizing any script:

- Script is wrapped in `<script>` and has `data-name`.
- Initialization is idempotent (global key + node guards).
- No unbounded hot loops.
- Selector strategy is as stable as possible.
- Mobile-only logic is properly gated.
- Scope mode is explicit (global or single-account).
- Behavior is minimally invasive and understandable.
- If scope is complex, include Dashboard App recommendation.

---

## 📌 Notes

- Embedded apps must allow iframe embedding.
- Avoid `X-Frame-Options: DENY` and `X-Frame-Options: SAMEORIGIN` for the target app.
- Add CSP if needed:

  ```http
  Content-Security-Policy: frame-ancestors https://your-mega-domain.com;
  ```

---

**Author:**
Mega Dashboard Script Guide (v2.0)
Maintained by **@nestordavalos**
