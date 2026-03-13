// ==UserScript==
// @name         Discord Search Pager UI
// @namespace    local.discord.search.pager
// @version      0.4.0
// @description  Панель поверх Discord Web для постраничного сбора результатов поиска
// @match        https://discord.com/channels/*
// @grant        GM_setClipboard
// ==/UserScript==

(function () {
  'use strict';

  if (window.__discordPagerUiLoaded) return;
  window.__discordPagerUiLoaded = true;

  const STATE = {
    running: false,
    stopRequested: false,
    pages: [],
    lastStatus: '',
    storageKey: 'discordPagerStateV4',
    uiKey: 'discordPagerUiStateV4',
  };

  function cleanText(s) {
    return String(s || '')
      .replace(/\u00A0/g, ' ')
      .replace(/\r/g, '')
      .replace(/[ \t]+/g, ' ')
      .replace(/\n{3,}/g, '\n\n')
      .trim();
  }

  function sleep(ms) {
    return new Promise((resolve) => setTimeout(resolve, ms));
  }

  function normalizeForCompare(s) {
    return cleanText(s)
      .toLowerCase()
      .replace(/\s+/g, ' ')
      .trim();
  }

  function isDigit(ch) {
    return ch >= '0' && ch <= '9';
  }

  function hasClockTime(s) {
    for (let i = 0; i < s.length - 4; i++) {
      if (
        isDigit(s[i]) &&
        isDigit(s[i + 1]) &&
        s[i + 2] === ':' &&
        isDigit(s[i + 3]) &&
        isDigit(s[i + 4])
      ) {
        return true;
      }
    }
    return false;
  }

  function isStandaloneClockTime(line) {
    return /^\d{1,2}:\d{2}$/.test(cleanText(line));
  }

  function isRelativeTimeLine(line) {
    const s = normalizeForCompare(line);
    return s.startsWith('сегодня') || s.startsWith('вчера') || s.startsWith('today') || s.startsWith('yesterday');
  }

  function isAbsoluteTimeLine(line) {
    const s = normalizeForCompare(line);

    const ruMonths = ['январ', 'феврал', 'март', 'апрел', 'мая', 'июн', 'июл', 'август', 'сентябр', 'октябр', 'ноябр', 'декабр'];
    const enMonths = ['january', 'february', 'march', 'april', 'may', 'june', 'july', 'august', 'september', 'october', 'november', 'december'];
    const ruWeekdays = ['понедельник', 'вторник', 'среда', 'четверг', 'пятница', 'суббота', 'воскресенье'];
    const enWeekdays = ['monday', 'tuesday', 'wednesday', 'thursday', 'friday', 'saturday', 'sunday'];

    const hasYear = /\b20\d{2}\b/.test(s);
    const hasMonth = ruMonths.some((x) => s.includes(x)) || enMonths.some((x) => s.includes(x));
    const hasWeekday = ruWeekdays.some((x) => s.includes(x)) || enWeekdays.some((x) => s.includes(x));

    return hasClockTime(s) && (hasYear || hasMonth || hasWeekday);
  }

  function isNavigationLine(line) {
    const s = normalizeForCompare(line);
    if (!s) return false;
    if ((s.includes('далее') || s.includes('назад') || s.includes('next') || s.includes('previous')) && /\d/.test(s)) return true;
    if (/^(далее|назад|next|previous|prev|page)\b/.test(s)) return true;
    if (/^\d+(\s+\d+)*$/.test(s)) return true;
    return false;
  }

  function isUiNoiseLine(line) {
    const s = cleanText(line);
    const lower = normalizeForCompare(line);
    if (!s) return true;
    if (isRelativeTimeLine(s) || isStandaloneClockTime(s) || isNavigationLine(s)) return true;
    if (lower.includes('добавить в избранное')) return true;
    if (lower.includes('удалить из избранного')) return true;
    if (lower.includes('нажмите, чтобы узнать больше')) return true;
    if (lower.includes('не удается загрузить сообщение')) return true;
    if (/[▬│]{2,}/.test(s)) return true;
    if (/^#?[a-z0-9._-]{2,40}\s+#?[a-z0-9._-]{2,40}$/i.test(s)) return true;
    return false;
  }

  function looksLikeRoleLine(line, authorHint) {
    const s = cleanText(line);
    const lower = normalizeForCompare(line);
    if (!s) return false;

    if (
      lower.includes('(admin)') ||
      lower.includes('(moderator)') ||
      lower.includes('(mod)') ||
      lower.includes('(owner)') ||
      lower.includes('(staff)') ||
      lower.includes('(helper)') ||
      lower.endsWith(' developer') ||
      lower === 'developer'
    ) {
      return true;
    }

    const normHint = normalizeForCompare(authorHint || '');
    if (normHint && normalizeForCompare(s) === normHint) return false;

    const oneWord = s.split(/\s+/).filter(Boolean).length === 1;
    if (oneWord && /^[A-ZА-ЯЁ][A-Za-zА-Яа-яЁё0-9_-]{2,24}$/.test(s)) {
      return true;
    }

    if (/^[A-ZА-Я0-9 _.,\-\[\]\(\)]+$/.test(s) && s.length <= 50) {
      return true;
    }

    return false;
  }

  function sanitizeAuthor(s) {
    return cleanText(s)
      .replace(/\s*\[[^\]]+\]/g, '')
      .replace(/\s*\((admin|moderator|mod|owner|staff|helper)[^)]*\)/gi, '')
      .replace(/\s*,.*$/, '')
      .replace(/[—–-]+\s*$/, '')
      .replace(/\s{2,}/g, ' ')
      .trim();
  }

  function readAuthorHint() {
    const el = document.getElementById('dsp-author-hint');
    return sanitizeAuthor(el ? el.value : '');
  }

  function stripKnownUiFragments(s) {
    let out = cleanText(s);
    out = out.replace(/\b(Добавить|Удалить) в избранное\b/gi, '');
    out = out.replace(/\bНе удается загрузить сообщение\b/gi, '');
    out = out.replace(/\bНажмите, чтобы узнать больше\b/gi, '');
    out = out.replace(/[—–-]+\s*$/g, '');
    out = out.replace(/\s{2,}/g, ' ').trim();
    return out;
  }

  function stripRepeatedChannelNames(s) {
    let out = cleanText(s);
    let prev;
    do {
      prev = out;
      out = out.replace(/^#?([a-z0-9._-]{2,40})\s+#?\1\b\s*/i, '');
      out = out.replace(/^([A-Za-zА-Яа-яЁё0-9._-]{2,40})\s+\1\b\s*/i, '');
    } while (out !== prev);
    return out.trim();
  }

  function cleanupContentLine(line, authorHint) {
    let out = stripKnownUiFragments(line);
    out = stripRepeatedChannelNames(out);
    out = out.replace(/^<@!?\d+>\s*/g, '');
    out = out.replace(/^@!?[A-Za-z0-9._-]+\s*/g, '');
    out = out.replace(/^replying to\s+/i, '');
    out = out.replace(/^ответ на\s+/i, '');

    const hint = sanitizeAuthor(authorHint || '');
    if (hint) {
      const escaped = hint.replace(/[.*+?^${}()|[\]\\]/g, '\\$&');
      out = out.replace(new RegExp(`\\s+${escaped}\\s*$`, 'i'), '');
    }

    out = out.replace(/\s{2,}/g, ' ').trim();
    return out;
  }

  function splitTrailingAuthor(line, authorHint) {
    let out = cleanText(line);
    const hint = sanitizeAuthor(authorHint || '');

    if (hint) {
      const escaped = hint.replace(/[.*+?^${}()|[\]\\]/g, '\\$&');
      const re = new RegExp(`^(.*?)(?:\\s+${escaped})\\s*$`, 'i');
      const m = out.match(re);
      if (m) {
        return {
          text: cleanText(m[1]),
          author: hint,
        };
      }
    }

    return {
      text: out,
      author: '',
    };
  }

  function guessAuthorFromBlock(lines, authorHint) {
    const hint = sanitizeAuthor(authorHint || '');
    if (hint) return hint;

    for (let i = lines.length - 1; i >= 0; i--) {
      const s = sanitizeAuthor(lines[i]);
      if (!s) continue;
      if (s.length > 1 && s.length <= 32 && /^[A-Za-zА-Яа-яЁё0-9_. -]+$/.test(s) && !s.includes(':')) {
        return s;
      }
    }

    return '';
  }

  function normalizePageText(text) {
    const lines = String(text || '')
      .split('\n')
      .map((x) => cleanText(x))
      .filter(Boolean);

    const authorHint = readAuthorHint();
    const out = [];

    for (let i = 0; i < lines.length; i++) {
      if (!isAbsoluteTimeLine(lines[i])) continue;

      const timestamp = lines[i];
      const block = [];
      let j = i + 1;

      while (j < lines.length && !isAbsoluteTimeLine(lines[j])) {
        block.push(lines[j]);
        j += 1;
      }

      const filtered = block.filter((line) => !isUiNoiseLine(line) && !looksLikeRoleLine(line, authorHint));
      if (!filtered.length) {
        i = j - 1;
        continue;
      }

      let author = guessAuthorFromBlock(filtered, authorHint);
      let contentLines = filtered.slice();

      if (!author && contentLines.length) {
        const split = splitTrailingAuthor(contentLines[contentLines.length - 1], authorHint);
        contentLines[contentLines.length - 1] = split.text;
        if (split.author) author = split.author;
      }

      if (author) {
        const normAuthor = normalizeForCompare(author);
        contentLines = contentLines.filter((line) => normalizeForCompare(sanitizeAuthor(line)) !== normAuthor);
      }

      contentLines = contentLines
        .map((line) => cleanupContentLine(line, author))
        .filter(Boolean)
        .filter((line) => !looksLikeRoleLine(line, author) && !isUiNoiseLine(line));

      if (!contentLines.length) {
        i = j - 1;
        continue;
      }

      if (!author) {
        const split = splitTrailingAuthor(contentLines[contentLines.length - 1], authorHint);
        if (split.author) author = split.author;
        contentLines[contentLines.length - 1] = split.text;
      }

      author = sanitizeAuthor(author || authorHint || '');
      if (!author) {
        i = j - 1;
        continue;
      }

      const mainMessage = contentLines[contentLines.length - 1] || '';
      const replyPreview = contentLines.slice(0, -1).join(' ').trim();

      const finalMain = cleanupContentLine(mainMessage, author);
      const finalReply = cleanupContentLine(replyPreview, author);

      if (!finalMain) {
        i = j - 1;
        continue;
      }

      if (finalReply) {
        out.push(`${timestamp} | ${author} | ↳ ${finalReply} || ${finalMain}`);
      } else {
        out.push(`${timestamp} | ${author} | ${finalMain}`);
      }

      i = j - 1;
    }

    return out.join('\n');
  }

  function getMergedText() {
    return STATE.pages
      .map((p) => normalizePageText(p.text))
      .filter(Boolean)
      .join('\n');
  }

  function saveUiState() {
    const host = document.getElementById('discord-search-pager-ui');
    const payload = {
      authorHint: readAuthorHint(),
      maxPages: document.getElementById('dsp-max-pages')?.value || '20',
      delayMs: document.getElementById('dsp-delay-ms')?.value || '4500',
      continueMode: !!document.getElementById('dsp-continue-mode')?.checked,
      right: host?.style.right || '16px',
      bottom: host?.style.bottom || '16px',
    };
    localStorage.setItem(STATE.uiKey, JSON.stringify(payload));
  }

  function loadUiState() {
    try {
      const raw = localStorage.getItem(STATE.uiKey);
      return raw ? JSON.parse(raw) : {};
    } catch {
      return {};
    }
  }

  function saveState() {
    const payload = {
      pages: STATE.pages,
      savedAt: new Date().toISOString(),
    };
    localStorage.setItem(STATE.storageKey, JSON.stringify(payload));
    window.__discordPagesDump = STATE.pages;
    window.__discordMergedDump = getMergedText();
    saveUiState();
  }

  function loadState() {
    try {
      const raw = localStorage.getItem(STATE.storageKey);
      if (!raw) return;
      const parsed = JSON.parse(raw);
      if (Array.isArray(parsed.pages)) STATE.pages = parsed.pages;
      window.__discordPagesDump = STATE.pages;
      window.__discordMergedDump = getMergedText();
    } catch (e) {
      console.warn('Не удалось прочитать сохранённый прогресс', e);
    }
  }

  function clearState() {
    STATE.pages = [];
    localStorage.removeItem(STATE.storageKey);
    window.__discordPagesDump = [];
    window.__discordMergedDump = '';
    renderStats();
  }

  function setStatus(text) {
    STATE.lastStatus = text;
    const node = document.getElementById('dsp-status');
    if (node) node.textContent = text;
    console.log('[DiscordPager]', text);
  }

  function renderStats() {
    const pagesNode = document.getElementById('dsp-pages-count');
    const charsNode = document.getElementById('dsp-chars-count');
    if (pagesNode) pagesNode.textContent = String(STATE.pages.length);
    if (charsNode) charsNode.textContent = String(getMergedText().length);
  }

  function findRightScrollablePanel() {
    const divs = Array.from(document.querySelectorAll('div'));

    const candidates = divs.filter((el) => {
      const r = el.getBoundingClientRect();
      const style = getComputedStyle(el);
      const visible = style.display !== 'none' && style.visibility !== 'hidden';
      const scrollable = el.scrollHeight > el.clientHeight + 50;
      const onRight = r.left > window.innerWidth * 0.5;
      const tall = r.height > 220;
      const wide = r.width > 220;
      return visible && scrollable && onRight && tall && wide;
    });

    candidates.sort((a, b) => {
      const ra = a.getBoundingClientRect();
      const rb = b.getBoundingClientRect();
      const score = (el, r) => (el.scrollHeight - el.clientHeight) + r.left + r.height + (cleanText(el.innerText).length || 0);
      return score(b, rb) - score(a, ra);
    });

    return candidates[0] || null;
  }

  function findPaginationRoot(panel) {
    let node = panel;
    for (let i = 0; i < 8 && node; i++) {
      const parent = node.parentElement;
      if (!parent) break;
      const buttons = parent.querySelectorAll('button');
      if (buttons.length >= 2) return parent;
      node = parent;
    }
    return panel.parentElement || document;
  }

  function getCurrentPageInfo(scope) {
    const txt = cleanText(scope.innerText);
    const m = txt.match(/\bpage\s+(\d+)\s+of\s+(\d+)\b/i) || txt.match(/\b(\d+)\s*\/\s*(\d+)\b/);
    if (!m) return null;
    return { page: Number(m[1]), total: Number(m[2]) };
  }

  function findNextPageButton(scope) {
    const buttons = Array.from(scope.querySelectorAll('button'));

    const labeled = buttons.filter((btn) => {
      const text = cleanText(btn.innerText);
      const aria = cleanText(btn.getAttribute('aria-label'));
      const title = cleanText(btn.getAttribute('title'));
      const full = `${text} ${aria} ${title}`.toLowerCase();
      const disabled = btn.disabled || btn.getAttribute('aria-disabled') === 'true';
      if (disabled) return false;
      return /next|след|вперед|forward|older|стар/i.test(full) || /›|»|→/.test(full);
    });

    if (labeled.length) return labeled[0];

    const visibleButtons = buttons.filter((btn) => {
      const r = btn.getBoundingClientRect();
      const style = getComputedStyle(btn);
      return style.display !== 'none' && style.visibility !== 'hidden' && r.width > 10 && r.height > 10;
    });

    if (visibleButtons.length >= 2) return visibleButtons[visibleButtons.length - 1];
    return null;
  }

  function collectPanelText(panel) {
    return cleanText(panel.innerText);
  }

  async function waitForPageChange(panel, beforeText, timeoutMs) {
    const started = Date.now();
    while (Date.now() - started < timeoutMs) {
      if (STATE.stopRequested) return false;
      await sleep(500);
      const nowText = cleanText(panel.innerText);
      if (nowText && nowText !== beforeText) return true;
    }
    return false;
  }

  function pageAlreadySaved(pageNo, text) {
    const signature = text.slice(0, 1200);
    return STATE.pages.some((p) => p.page === pageNo || p.signature === signature);
  }

  async function startCollection() {
    if (STATE.running) return;

    const maxPages = Math.max(1, Number(document.getElementById('dsp-max-pages').value || 1));
    const delayMs = Math.max(500, Number(document.getElementById('dsp-delay-ms').value || 4500));
    const continueMode = document.getElementById('dsp-continue-mode').checked;

    if (!continueMode) clearState();

    const panel = findRightScrollablePanel();
    if (!panel) {
      setStatus('Не удалось найти правую панель Search Results');
      return;
    }

    const paginationRoot = findPaginationRoot(panel);
    STATE.running = true;
    STATE.stopRequested = false;
    setButtonsState();
    saveUiState();

    try {
      for (let i = 0; i < maxPages; i++) {
        if (STATE.stopRequested) {
          setStatus('Остановлено пользователем');
          break;
        }

        panel.scrollTop = 0;
        await sleep(400);

        const pageInfo = getCurrentPageInfo(paginationRoot);
        const pageNo = pageInfo?.page ?? (STATE.pages.length + 1);
        const totalPages = pageInfo?.total ?? null;
        const pageText = collectPanelText(panel);

        if (!pageText) {
          setStatus(`Страница ${pageNo}: пустой текст, остановка`);
          break;
        }

        if (!pageAlreadySaved(pageNo, pageText)) {
          STATE.pages.push({
            page: pageNo,
            total: totalPages,
            text: pageText,
            signature: pageText.slice(0, 1200),
            savedAt: new Date().toISOString(),
          });
          saveState();
          renderStats();
        }

        setStatus(`Собрана страница ${pageNo}${totalPages ? '/' + totalPages : ''}`);

        const nextBtn = findNextPageButton(paginationRoot);
        if (!nextBtn) {
          setStatus('Кнопка следующей страницы не найдена. Завершено');
          break;
        }

        const beforeText = pageText;
        nextBtn.click();

        const changed = await waitForPageChange(panel, beforeText, Math.max(8000, delayMs + 6000));
        if (!changed) {
          setStatus('Страница не изменилась после клика. Завершено');
          break;
        }

        await sleep(delayMs);
      }
    } finally {
      STATE.running = false;
      STATE.stopRequested = false;
      setButtonsState();
      renderStats();
      saveUiState();
    }
  }

  function stopCollection() {
    STATE.stopRequested = true;
    setStatus('Запрошена остановка...');
  }

  async function copyMerged() {
    const txt = getMergedText();
    if (!txt) {
      setStatus('Пока нечего копировать');
      return;
    }

    try {
      if (typeof GM_setClipboard === 'function') {
        GM_setClipboard(txt);
      } else {
        await navigator.clipboard.writeText(txt);
      }
      setStatus(`Скопировано ${txt.length} символов`);
    } catch (e) {
      console.error(e);
      setStatus('Не удалось скопировать в буфер');
    }
  }

  function downloadTextFile(name, text, mimeType) {
    const blob = new Blob([text], { type: mimeType || 'text/plain;charset=utf-8' });
    const url = URL.createObjectURL(blob);
    const a = document.createElement('a');
    a.href = url;
    a.download = name;
    document.body.appendChild(a);
    a.click();
    a.remove();
    URL.revokeObjectURL(url);
  }

  function exportTxt() {
    const txt = getMergedText();
    if (!txt) {
      setStatus('Нет данных для экспорта');
      return;
    }
    downloadTextFile('discord_search_results.txt', txt, 'text/plain;charset=utf-8');
    setStatus('TXT экспортирован');
  }

  function exportJson() {
    if (!STATE.pages.length) {
      setStatus('Нет данных для экспорта');
      return;
    }

    const normalized = STATE.pages.map((p) => ({
      page: p.page,
      total: p.total,
      savedAt: p.savedAt,
      normalizedText: normalizePageText(p.text),
      rawText: p.text,
    }));

    downloadTextFile('discord_search_results.json', JSON.stringify(normalized, null, 2), 'application/json;charset=utf-8');
    setStatus('JSON экспортирован');
  }

  function setButtonsState() {
    const startBtn = document.getElementById('dsp-start');
    const stopBtn = document.getElementById('dsp-stop');
    if (startBtn) startBtn.disabled = STATE.running;
    if (stopBtn) stopBtn.disabled = !STATE.running;
  }

  function createUI() {
    const savedUi = loadUiState();

    const host = document.createElement('div');
    host.id = 'discord-search-pager-ui';
    host.innerHTML = `
      <div id="dsp-wrap">
        <div id="dsp-title">Discord Search Pager</div>
        <label class="dsp-row">
          <span>Ник автора</span>
          <input id="dsp-author-hint" type="text" placeholder="например, Frol" value="${cleanText(savedUi.authorHint || '')}" />
        </label>
        <label class="dsp-row">
          <span>Лимит страниц</span>
          <input id="dsp-max-pages" type="number" min="1" value="${cleanText(savedUi.maxPages || '20')}" />
        </label>
        <label class="dsp-row">
          <span>Пауза, мс</span>
          <input id="dsp-delay-ms" type="number" min="500" step="250" value="${cleanText(savedUi.delayMs || '4500')}" />
        </label>
        <label class="dsp-check">
          <input id="dsp-continue-mode" type="checkbox" ${savedUi.continueMode !== false ? 'checked' : ''} />
          <span>Продолжать, не очищая прогресс</span>
        </label>
        <div class="dsp-buttons">
          <button id="dsp-start">Старт</button>
          <button id="dsp-stop" disabled>Стоп</button>
        </div>
        <div class="dsp-buttons">
          <button id="dsp-copy">Копировать</button>
          <button id="dsp-export-txt">TXT</button>
          <button id="dsp-export-json">JSON</button>
        </div>
        <div class="dsp-buttons">
          <button id="dsp-clear">Очистить</button>
        </div>
        <div id="dsp-stats">
          <div>Страниц: <b id="dsp-pages-count">0</b></div>
          <div>Символов: <b id="dsp-chars-count">0</b></div>
        </div>
        <div id="dsp-status">Готово</div>
      </div>
    `;

    const style = document.createElement('style');
    style.textContent = `
      #discord-search-pager-ui {
        position: fixed;
        right: ${cleanText(savedUi.right || '16px')};
        bottom: ${cleanText(savedUi.bottom || '16px')};
        z-index: 999999;
        font-family: Inter, system-ui, sans-serif;
      }
      #dsp-wrap {
        width: 320px;
        background: rgba(18, 18, 24, 0.95);
        color: #fff;
        border: 1px solid rgba(255,255,255,0.12);
        border-radius: 12px;
        box-shadow: 0 10px 30px rgba(0,0,0,0.35);
        padding: 12px;
        backdrop-filter: blur(8px);
      }
      #dsp-title {
        font-weight: 700;
        font-size: 14px;
        margin-bottom: 10px;
        cursor: move;
        user-select: none;
      }
      .dsp-row {
        display: flex;
        align-items: center;
        justify-content: space-between;
        gap: 8px;
        margin-bottom: 8px;
        font-size: 12px;
      }
      .dsp-row input {
        width: 150px;
        background: #11131a;
        color: #fff;
        border: 1px solid rgba(255,255,255,0.12);
        border-radius: 8px;
        padding: 6px 8px;
      }
      .dsp-check {
        display: flex;
        gap: 8px;
        align-items: center;
        font-size: 12px;
        margin: 8px 0 10px;
      }
      .dsp-buttons {
        display: grid;
        grid-template-columns: 1fr 1fr;
        gap: 8px;
        margin-bottom: 8px;
      }
      .dsp-buttons button {
        background: #5865f2;
        color: #fff;
        border: none;
        border-radius: 8px;
        padding: 8px 10px;
        cursor: pointer;
        font-size: 12px;
        font-weight: 600;
      }
      .dsp-buttons button:disabled {
        opacity: 0.5;
        cursor: default;
      }
      #dsp-clear {
        background: #3a3f4b;
        grid-column: 1 / -1;
      }
      #dsp-stats {
        font-size: 12px;
        opacity: 0.9;
        margin-top: 8px;
        display: grid;
        gap: 4px;
      }
      #dsp-status {
        margin-top: 10px;
        font-size: 12px;
        line-height: 1.35;
        opacity: 0.95;
      }
    `;

    document.documentElement.appendChild(style);
    document.documentElement.appendChild(host);

    document.getElementById('dsp-start').addEventListener('click', startCollection);
    document.getElementById('dsp-stop').addEventListener('click', stopCollection);
    document.getElementById('dsp-copy').addEventListener('click', copyMerged);
    document.getElementById('dsp-export-txt').addEventListener('click', exportTxt);
    document.getElementById('dsp-export-json').addEventListener('click', exportJson);
    document.getElementById('dsp-clear').addEventListener('click', clearState);

    for (const id of ['dsp-author-hint', 'dsp-max-pages', 'dsp-delay-ms', 'dsp-continue-mode']) {
      document.getElementById(id).addEventListener('input', saveUiState);
      document.getElementById(id).addEventListener('change', saveUiState);
    }

    const title = document.getElementById('dsp-title');
    let dragging = false;
    let startX = 0;
    let startY = 0;
    let startRight = 16;
    let startBottom = 16;

    title.addEventListener('mousedown', (e) => {
      dragging = true;
      startX = e.clientX;
      startY = e.clientY;
      const computed = getComputedStyle(host);
      startRight = parseFloat(computed.right) || 16;
      startBottom = parseFloat(computed.bottom) || 16;
      e.preventDefault();
    });

    document.addEventListener('mousemove', (e) => {
      if (!dragging) return;
      const dx = e.clientX - startX;
      const dy = e.clientY - startY;
      const maxRight = Math.max(0, window.innerWidth - host.offsetWidth);
      const maxBottom = Math.max(0, window.innerHeight - host.offsetHeight);
      const nextRight = Math.min(Math.max(startRight - dx, 0), maxRight);
      const nextBottom = Math.min(Math.max(startBottom + dy, 0), maxBottom);
      host.style.right = `${nextRight}px`;
      host.style.bottom = `${nextBottom}px`;
      host.style.top = 'auto';
      host.style.left = 'auto';
      saveUiState();
    });

    document.addEventListener('mouseup', () => {
      dragging = false;
    });

    renderStats();
    setButtonsState();
  }

  loadState();
  createUI();
  setStatus('Панель загружена. Укажите ник автора, откройте поиск в Discord и нажмите Старт');
})();
