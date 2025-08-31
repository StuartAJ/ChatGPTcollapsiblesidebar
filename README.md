# ChatGPTcollapsiblesidebar

(() => {
  // Prevent duplicates
  if (window.__gptUiKitV5) { console.info('UI kit already loaded.'); return; }

  const NS = '__gptUiKitV5';
  const SIDEBAR_ID = 'stage-slideover-sidebar';
  const COLLAPSE_CLASS = 'gpt-hard-collapsed';
  const STYLE_ID = 'gpt-ui-kit-style';
  const BTN_ID = 'gpt-collapse-btn';
  const MODAL_ID = 'gpt-chat-modal';

  const state = {
    collapsed: false,
    mo: null,
    originals: new Map(), // CSS vars on :root + body
  };

  // ---------- helpers ----------
  const $ = (sel, root=document) => root.querySelector(sel);
  const $$ = (sel, root=document) => Array.from(root.querySelectorAll(sel));

  const isInside = (a, b) => !!(a && b && b.contains(a));

  // Only trigger when not typing into the main chat input or any editable,
  // but allow keystrokes when our modal is the active context.
  function isTypingContext(evTarget) {
    const target = evTarget || document.activeElement;
    const modal = document.getElementById(MODAL_ID);
    if (modal && isInside(target, modal)) return false; // typing inside our modal is fine
    const editable = target?.closest?.(
      'input, textarea, [contenteditable=""], [contenteditable="true"], [role="textbox"]'
    );
    return !!editable;
  }

  // ---------- styles ----------
  const style = document.createElement('style');
  style.id = STYLE_ID;
  style.textContent = `
    /* Sidebar hard-collapse */
    body.${COLLAPSE_CLASS} #${CSS.escape(SIDEBAR_ID)} { display: none !important; }
    /* Toggle button */
    #${BTN_ID} {
      position: fixed; left: 8px; bottom: 12px; z-index: 2147483647;
      padding: 6px 9px; border-radius: 8px; background: rgba(0,0,0,.6);
      color: #fff; border: 0; cursor: pointer;
      font: 13px/1 system-ui,-apple-system,Segoe UI,Roboto,Ubuntu,Cantarell,Noto Sans,Arial;
    }
    #${BTN_ID}:focus { outline: 2px solid #fff7; outline-offset: 2px; }

    /* Chat switcher modal */
    #${MODAL_ID} {
      position: fixed; inset: 0; z-index: 2147483647;
      display: flex; align-items: center; justify-content: center;
      background: rgba(0,0,0,.35);
    }
    #${MODAL_ID} .sheet {
      width: min(860px, 92vw); max-height: 78vh; overflow: hidden;
      background: var(--bg-elevated, var(--token-bg-elevated-secondary, #1f1f1f));
      color: var(--token-text-primary, #e6e6e6);
      border-radius: 14px; box-shadow: 0 10px 40px rgba(0,0,0,.35);
      display: flex; flex-direction: column;
      border: 1px solid var(--token-border-light, #333);
    }
    #${MODAL_ID} .header {
      padding: 12px 14px; border-bottom: 1px solid var(--token-border-light, #333);
      display: flex; gap: 10px; align-items: center;
    }
    #${MODAL_ID} .header input {
      width: 100%; padding: 10px 12px; border-radius: 10px; border: 1px solid var(--token-border-light, #444);
      background: var(--token-surface, #121212); color: inherit; outline: none;
      font: 14px/1.2 system-ui,-apple-system,Segoe UI,Roboto,Ubuntu,Cantarell,Noto Sans,Arial;
    }
    #${MODAL_ID} .list {
      overflow: auto; -webkit-overflow-scrolling: touch; padding: 6px;
    }
    #${MODAL_ID} .item {
      display: flex; align-items: center; gap: 10px;
      padding: 10px 12px; border-radius: 10px; cursor: pointer;
      border: 1px solid transparent;
    }
    #${MODAL_ID} .item:hover { background: rgba(255,255,255,.04); }
    #${MODAL_ID} .item.selected {
      background: rgba(255,255,255,.08);
      border-color: var(--token-border-light, #444);
      outline: 2px solid rgba(255,255,255,.12);
    }
    #${MODAL_ID} .title {
      white-space: nowrap; overflow: hidden; text-overflow: ellipsis;
      flex: 1; font-size: 14px;
    }
    #${MODAL_ID} .meta {
      color: var(--token-text-tertiary, #9aa0a6); font-size: 12px;
    }
    #${MODAL_ID} .footer {
      padding: 8px 12px; border-top: 1px solid var(--token-border-light, #333);
      display: flex; justify-content: space-between; color: var(--token-text-tertiary, #9aa0a6);
      font-size: 12px;
    }
    @media (prefers-color-scheme: light) {
      #${MODAL_ID} .sheet {
        background: #fff; color: #1b1b1b; border-color: #e5e7eb;
      }
      #${MODAL_ID} .header input { background: #fff; border-color: #e5e7eb; }
      #${MODAL_ID} .item:hover { background: #f3f4f6; }
      #${MODAL_ID} .item.selected { background: #eef2ff; outline-color: #c7d2fe; border-color: #c7d2fe; }
      #${MODAL_ID} .footer { border-top-color: #e5e7eb; color: #6b7280; }
    }
  `;
  document.head.appendChild(style);

  // ---------- sidebar collapse ----------
  const varTargets = [document.documentElement, document.body];

  function snapshotVars() {
    for (const el of varTargets) {
      state.originals.set(el, {
        sw: getComputedStyle(el).getPropertyValue('--sidebar-width').trim(),
        srw: getComputedStyle(el).getPropertyValue('--sidebar-rail-width').trim(),
      });
    }
  }
  function setVars(sw, srw) {
    for (const el of varTargets) {
      el.style.setProperty('--sidebar-width', sw);
      el.style.setProperty('--sidebar-rail-width', srw);
    }
  }

  function widenMain(save=true) {
    const main = $('[role="main"]') || $('main');
    if (!main) return;
    if (save) {
      main.dataset._mw = main.style.maxWidth || '';
      main.dataset._w  = main.style.width || '';
      main.dataset._ml = main.style.marginLeft || '';
      main.dataset._mr = main.style.marginRight || '';
      main.dataset._mwMin = main.style.minWidth || '';
    }
    Object.assign(main.style, {
      maxWidth: '100%', width: '100%',
      marginLeft: '0', marginRight: '0', minWidth: '0',
    });
  }
  function restoreMain() {
    const main = $('[role="main"]') || $('main');
    if (!main) return;
    main.style.maxWidth  = main.dataset._mw || '';
    main.style.width     = main.dataset._w  || '';
    main.style.marginLeft= main.dataset._ml || '';
    main.style.marginRight=main.dataset._mr || '';
    main.style.minWidth  = main.dataset._mwMin || '';
  }

  function collapseSidebar() {
    snapshotVars();
    document.body.classList.add(COLLAPSE_CLASS);
    setVars('0px', '0px');
    const sb = document.getElementById(SIDEBAR_ID);
    if (sb) { sb.style.display = 'none'; sb.inert = true; }

    widenMain();

    // Keep hidden if app re-renders
    state.mo?.disconnect();
    state.mo = new MutationObserver(() => {
      if (!document.body.classList.contains(COLLAPSE_CLASS)) return;
      const s = document.getElementById(SIDEBAR_ID);
      if (s && s.style.display !== 'none') { s.style.display = 'none'; s.inert = true; }
    });
    state.mo.observe(document.body, { childList: true, subtree: true });

    state.collapsed = true;
  }

  function expandSidebar() {
    document.body.classList.remove(COLLAPSE_CLASS);
    for (const [el, v] of state.originals.entries()) {
      el.style.setProperty('--sidebar-width', v.sw || '');
      el.style.setProperty('--sidebar-rail-width', v.srw || '');
    }
    const sb = document.getElementById(SIDEBAR_ID);
    if (sb) { sb.style.display = ''; sb.inert = false; }
    restoreMain();
    state.mo?.disconnect(); state.mo = null;
    state.collapsed = false;
  }

  function toggleSidebar() {
    (state.collapsed ? expandSidebar : collapseSidebar)();
  }

  // ---------- floating button (also respects typing context via keyboard handler) ----------
  const btn = document.createElement('button');
  btn.id = BTN_ID;
  btn.textContent = '☰';
  btn.title = 'Toggle sidebar (Shift+S)';
  btn.addEventListener('click', toggleSidebar);
  document.body.appendChild(btn);

  // ---------- chat switcher modal ----------
  function scanChats() {
    // Prefer the chat history area; fallback to any /c/<id> links
    const root = $('nav[aria-label="Chat history"]') || $('#history') || document;
    const anchors = $$('a[href^="/c/"]', root);
    const seen = new Set();
    return anchors.map(a => {
      const href = a.getAttribute('href');
      if (!href || seen.has(href)) return null;
      seen.add(href);
      // Try to pull a readable title; fall back to the link text
      const title =
        a.querySelector('[dir="auto"], .truncate, span')?.textContent?.trim() ||
        a.textContent.trim() || 'Untitled chat';
      return { title, href };
    }).filter(Boolean);
  }

  function buildModal(chats) {
    const overlay = document.createElement('div');
    overlay.id = MODAL_ID;
    overlay.setAttribute('role', 'dialog');
    overlay.setAttribute('aria-modal', 'true');

    overlay.innerHTML = `
      <div class="sheet" role="document">
        <div class="header">
          <input type="text" placeholder="Search chats… (↑/↓, Enter, Esc)" aria-label="Search chats" />
        </div>
        <div class="list" role="listbox" aria-label="Chats"></div>
        <div class="footer">
          <div><strong>${chats.length}</strong> chats</div>
          <div>Enter to open · Esc to close</div>
        </div>
      </div>
    `;

    // Close on backdrop click
    overlay.addEventListener('mousedown', (e) => {
      if (e.target === overlay) closeModal();
    });

    document.body.appendChild(overlay);
    document.body.style.overflow = 'hidden';

    const input = $('input', overlay);
    const list = $('.list', overlay);

    let filtered = chats.slice(0);
    let cursor = 0;

    function render(items) {
      list.innerHTML = '';
      if (!items.length) {
        const empty = document.createElement('div');
        empty.className = 'item';
        empty.style.opacity = '0.7';
        empty.textContent = 'No matching chats';
        list.appendChild(empty);
        return;
      }
      items.forEach((c, i) => {
        const div = document.createElement('div');
        div.className = 'item' + (i === cursor ? ' selected' : '');
        div.setAttribute('role', 'option');
        div.dataset.href = c.href;
        div.innerHTML = `<div class="title" title="${c.title.replace(/"/g,'&quot;')}">${c.title}</div>
                         <div class="meta">${c.href}</div>`;
        div.addEventListener('click', () => openChat(c.href));
        div.addEventListener('mousemove', () => { // hover moves selection
          const idx = Array.prototype.indexOf.call(list.children, div);
          if (idx >= 0) { cursor = idx; updateSelection(); }
        });
        list.appendChild(div);
      });
    }

    function updateSelection() {
      $$('.item', list).forEach((el, i) =>
        el.classList.toggle('selected', i === cursor)
      );
      // Ensure selected visible
      const sel = $$('.item', list)[cursor];
      if (sel) {
        const r = sel.getBoundingClientRect();
        const lr = list.getBoundingClientRect();
        if (r.top < lr.top) list.scrollTop += (r.top - lr.top) - 8;
        if (r.bottom > lr.bottom) list.scrollTop += (r.bottom - lr.bottom) + 8;
      }
    }

    function score(q, t) { // tiny fuzzy-ish score
      if (!q) return 1;
      q = q.toLowerCase(); t = t.toLowerCase();
      if (t.includes(q)) return 2;
      // all terms present?
      const terms = q.split(/\s+/).filter(Boolean);
      return terms.every(s => t.includes(s)) ? 1 : 0;
    }

    function filterList() {
      const q = input.value.trim();
      filtered = chats
        .map(c => ({ ...c, _s: score(q, c.title + ' ' + c.href) }))
        .filter(c => c._s > 0 || q === '')
        .sort((a,b) => b._s - a._s || a.title.localeCompare(b.title))
        .slice(0, 200);
      cursor = 0;
      render(filtered);
    }

    function openChat(href) {
      // Navigate in same tab
      window.location.assign(href);
    }

    // keyboard in input
    input.addEventListener('keydown', (e) => {
      if (e.key === 'Escape') { e.preventDefault(); closeModal(); return; }
      if (e.key === 'Enter')  { e.preventDefault(); const sel = filtered[cursor]; if (sel) openChat(sel.href); return; }
      if (e.key === 'ArrowDown') { e.preventDefault(); cursor = Math.min(cursor + 1, filtered.length - 1); updateSelection(); return; }
      if (e.key === 'ArrowUp')   { e.preventDefault(); cursor = Math.max(cursor - 1, 0); updateSelection(); return; }
      // Let other keys flow (for typing)
    });

    input.addEventListener('input', filterList);

    // initial
    filterList();
    input.focus();
  }

  function closeModal() {
    const overlay = document.getElementById(MODAL_ID);
    if (!overlay) return;
    overlay.remove();
    document.body.style.overflow = '';
  }

  function toggleChatModal() {
    const existing = document.getElementById(MODAL_ID);
    if (existing) { closeModal(); return; }
    const chats = scanChats();
    buildModal(chats);
  }

  // ---------- keyboard shortcuts (disabled while typing) ----------
  function onKeyDown(e) {
    if (e.isComposing || e.defaultPrevented) return;
    if (!(e.shiftKey && !e.ctrlKey && !e.metaKey && !e.altKey)) return;
    if (isTypingContext(e.target)) return; // don't fire while typing in the main textbox

    const k = e.key.toLowerCase();
    if (k === 's') { e.preventDefault(); toggleSidebar(); }
    else if (k === 'c') { e.preventDefault(); toggleChatModal(); }
  }
  window.addEventListener('keydown', onKeyDown, true);

  // Expose tiny API + uninstall
  window[NS] = {
    toggleSidebar,
    toggleChats: toggleChatModal,
    uninstall() {
      closeModal();
      document.getElementById(BTN_ID)?.remove();
      document.getElementById(STYLE_ID)?.remove();
      window.removeEventListener('keydown', onKeyDown, true);
      if (state.collapsed) expandSidebar();
      delete window[NS];
      console.info('UI kit removed.');
    }
  };

  // Start with sidebar collapsed once (optional). Comment out if you want default expanded.
  collapseSidebar();
})();
