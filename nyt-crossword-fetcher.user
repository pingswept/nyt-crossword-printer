// ==UserScript==
// @name         NYT Crossword Batch Fetcher
// @namespace    nyt-crossword-batch-fetcher
// @version      1.2
// @description  Download & merge a batch of NYT Crossword ink-saver PDFs (default Mon-Sat) without a server.
// @match        https://www.nytimes.com/crosswords/*
// @match        https://www.nytimes.com/svc/crosswords/*
// @grant        none
// @run-at       document-idle
// ==/UserScript==

(function () {
  "use strict";

  const PDF_LIB_URL = "https://cdn.jsdelivr.net/npm/pdf-lib@1.17.1/dist/pdf-lib.min.js";
  const JSZIP_URL = "https://cdn.jsdelivr.net/npm/jszip@3.10.1/dist/jszip.min.js";

  // Tampermonkey sometimes runs userscripts in a context where `window`
  // isn't the page's real window. `unsafeWindow` (when available) always
  // is. Fall back to `window` for engines that don't expose it.
  function getPageWindow() {
    /* eslint-disable no-undef */
    return typeof unsafeWindow !== "undefined" ? unsafeWindow : window;
    /* eslint-enable no-undef */
  }

  // Injects a real <script src> tag into the page DOM so the library
  // attaches to the page's actual global object, sidestepping any
  // @require sandboxing quirks.
  function loadScriptIntoPage(url) {
    return new Promise((resolve, reject) => {
      const s = document.createElement("script");
      s.src = url;
      s.onload = () => resolve();
      s.onerror = () => reject(new Error(`Failed to load ${url} (network blocked, or CSP disallowed it -- check the browser console for a red error mentioning this URL)`));
      (document.head || document.documentElement).appendChild(s);
    });
  }

  async function ensureLibs(diag) {
    const g = getPageWindow();
    diag.push(`unsafeWindow available: ${typeof unsafeWindow !== "undefined"}`);
    diag.push(`Before load -- PDFLib: ${typeof g.PDFLib}, JSZip: ${typeof g.JSZip}`);

    if (typeof g.PDFLib === "undefined") {
      await loadScriptIntoPage(PDF_LIB_URL);
    }
    if (typeof g.JSZip === "undefined") {
      await loadScriptIntoPage(JSZIP_URL);
    }

    diag.push(`After load -- PDFLib: ${typeof g.PDFLib}, JSZip: ${typeof g.JSZip}`);
    return { PDFLib: g.PDFLib, JSZip: g.JSZip, ok: typeof g.PDFLib !== "undefined" && typeof g.JSZip !== "undefined" };
  }

  const DEFAULT_WEEKDAYS = [1, 2, 3, 4, 5, 6]; // JS Date.getDay(): Sun=0 ... Sat=6; Mon-Sat

  // ---------- date helpers ----------

  function toISODate(d) {
    return d.toISOString().slice(0, 10);
  }

  function pad2(n) {
    return String(n).padStart(2, "0");
  }

  // Walk backward from asOf (inclusive) collecting dates whose JS getDay()
  // is in `weekdays`, until `count` dates are found. Returns oldest-first.
  function collectDates(asOf, weekdays, count) {
    const found = [];
    const d = new Date(asOf.getTime());
    const maxIterations = count * 14 + 30;
    for (let i = 0; i < maxIterations && found.length < count; i++) {
      if (weekdays.includes(d.getDay())) {
        found.push(new Date(d.getTime()));
      }
      d.setDate(d.getDate() - 1);
    }
    found.reverse();
    return found;
  }

  // ---------- NYT API calls (same-origin, cookies sent automatically) ----------

  async function fetchPuzzleIds(dates) {
    const sorted = [...dates].sort((a, b) => a - b);
    const dateStart = toISODate(sorted[0]);
    const dateEnd = toISODate(sorted[sorted.length - 1]);
    const url =
      `/svc/crosswords/v3/puzzles.json?publish_type=daily&sort_order=asc` +
      `&sort_by=print_date&date_start=${dateStart}&date_end=${dateEnd}`;
    log(`[meta] requesting ${url}`);
    const resp = await fetch(url, { credentials: "same-origin" });
    log(`[meta] status ${resp.status}`);
    if (!resp.ok) {
      throw new Error(`Puzzle metadata lookup failed (status ${resp.status}). Are you logged in?`);
    }
    const data = await resp.json();
    const byDate = {};
    for (const item of data.results || []) {
      if (item.print_date && item.puzzle_id != null) {
        byDate[item.print_date] = item.puzzle_id;
      }
    }
    log(`[meta] found ${Object.keys(byDate).length} puzzle id(s) for ${dates.length} requested date(s)`);
    return byDate;
  }

  async function fetchPdfBytes(puzzleId, opacity) {
    const url = `/svc/crosswords/v2/puzzle/${puzzleId}.pdf?block_opacity=${opacity}`;
    log(`[fetch] requesting ${url}`);
    const resp = await fetch(url, { credentials: "same-origin" });
    log(`[fetch] status ${resp.status}`);
    if (!resp.ok) {
      throw new Error(`Failed to fetch puzzle ${puzzleId} (status ${resp.status})`);
    }
    const buf = await resp.arrayBuffer();
    const head = new Uint8Array(buf.slice(0, 4));
    const isPdf = String.fromCharCode(...head) === "%PDF";
    if (!isPdf) {
      throw new Error(`Response for puzzle ${puzzleId} wasn't a PDF (cookie/session issue?)`);
    }
    return buf;
  }

  // ---------- merging / zipping ----------

  async function mergePdfs(buffers) {
    const { PDFDocument } = getPageWindow().PDFLib;
    const out = await PDFDocument.create();
    for (const buf of buffers) {
      const src = await PDFDocument.load(buf);
      const pages = await out.copyPages(src, src.getPageIndices());
      pages.forEach((p) => out.addPage(p));
    }
    return await out.save();
  }

  async function zipPdfs(entries) {
    const zip = new (getPageWindow().JSZip)();
    for (const { date, buf } of entries) {
      zip.file(`${date}-crossword.pdf`, buf);
    }
    return await zip.generateAsync({ type: "blob" });
  }

  function downloadBlob(blob, filename) {
    const url = URL.createObjectURL(blob);
    const a = document.createElement("a");
    a.href = url;
    a.download = filename;
    document.body.appendChild(a);
    a.click();
    a.remove();
    URL.revokeObjectURL(url);
  }

  // ---------- UI ----------

  let logEl;
  function log(msg) {
    console.log("[nyt-crossword-fetcher]", msg);
    if (logEl) {
      logEl.textContent += msg + "\n";
      logEl.scrollTop = logEl.scrollHeight;
    }
  }

  function buildPanel(libsOk, diagLines) {
    const panel = document.createElement("div");
    panel.id = "ncf-panel";
    panel.innerHTML = `
      <style>
        #ncf-panel {
          position: fixed; top: 80px; right: 20px; width: 300px;
          background: #fff; border: 1px solid #d8d5cc; border-radius: 6px;
          box-shadow: 0 4px 16px rgba(0,0,0,0.2); font-family: Georgia, serif;
          font-size: 13px; z-index: 999999; color: #121212;
        }
        #ncf-panel .ncf-header {
          background: #a3170e; color: #fff; padding: 8px 12px;
          border-radius: 6px 6px 0 0; font-weight: bold; cursor: move;
          display: flex; justify-content: space-between; align-items: center;
        }
        #ncf-panel .ncf-body { padding: 12px; }
        #ncf-panel.collapsed .ncf-body { display: none; }
        #ncf-panel label { display: block; margin: 8px 0 3px; font-weight: bold; }
        #ncf-panel input[type=date], #ncf-panel input[type=number] {
          width: 100%; padding: 4px 6px; font-family: inherit; box-sizing: border-box;
        }
        #ncf-panel .ncf-weekdays { display: flex; flex-wrap: wrap; gap: 6px; margin-top: 4px; }
        #ncf-panel .ncf-weekdays label {
          font-weight: normal; margin: 0; display: flex; align-items: center; gap: 3px;
        }
        #ncf-panel .ncf-mode { display: flex; gap: 12px; margin-top: 4px; }
        #ncf-panel .ncf-mode label { font-weight: normal; margin: 0; display: flex; align-items: center; gap: 4px; }
        #ncf-panel button.ncf-go {
          width: 100%; margin-top: 12px; background: #a3170e; color: #fff;
          border: none; padding: 8px; border-radius: 4px; cursor: pointer; font-size: 13px;
        }
        #ncf-panel button.ncf-go:disabled { opacity: 0.5; cursor: default; }
        #ncf-panel .ncf-toggle { background: none; border: none; color: #fff; cursor: pointer; font-size: 14px; }
        #ncf-panel .ncf-log {
          margin-top: 10px; max-height: 100px; overflow-y: auto; background: #f4f2ee;
          border: 1px solid #d8d5cc; border-radius: 3px; padding: 6px; font-family: monospace;
          font-size: 10px; white-space: pre-wrap;
        }
      </style>
      <div class="ncf-header">
        <span>Crossword Batch Fetcher</span>
        <button class="ncf-toggle" title="Collapse">&minus;</button>
      </div>
      <div class="ncf-body">
        ${libsOk ? "" : `<div style="color:#a3170e; font-weight:bold; margin-bottom:8px; font-size:11px;">
          Warning: pdf-lib/JSZip failed to load. Fetch is disabled. Diagnostics:<br>
          <pre style="white-space:pre-wrap; font-weight:normal; margin:4px 0 0;">${(diagLines || []).join("\n")}</pre>
          Check the browser console (F12) for the specific network/CSP error.
        </div>`}
        <label>As of date</label>
        <input type="date" class="ncf-asof">

        <label>Number of puzzles</label>
        <input type="number" class="ncf-count" value="6" min="1" max="31">

        <label>Days to include</label>
        <div class="ncf-weekdays">
          <label><input type="checkbox" value="1" checked> Mon</label>
          <label><input type="checkbox" value="2" checked> Tue</label>
          <label><input type="checkbox" value="3" checked> Wed</label>
          <label><input type="checkbox" value="4" checked> Thu</label>
          <label><input type="checkbox" value="5" checked> Fri</label>
          <label><input type="checkbox" value="6" checked> Sat</label>
          <label><input type="checkbox" value="0"> Sun</label>
        </div>

        <label>Block opacity (ink saver): <span class="ncf-opacity-val">100</span></label>
        <input type="range" class="ncf-opacity" min="0" max="100" value="100">

        <label>Output</label>
        <div class="ncf-mode">
          <label><input type="radio" name="ncf-mode" value="merged" checked> Merged PDF</label>
          <label><input type="radio" name="ncf-mode" value="zip"> ZIP</label>
        </div>

        <button class="ncf-go">Fetch &amp; Download</button>
        <div class="ncf-log"></div>
      </div>
    `;
    document.body.appendChild(panel);

    panel.querySelector(".ncf-asof").value = toISODate(new Date());
    logEl = panel.querySelector(".ncf-log");

    const opacityInput = panel.querySelector(".ncf-opacity");
    const opacityVal = panel.querySelector(".ncf-opacity-val");
    opacityInput.addEventListener("input", () => {
      opacityVal.textContent = opacityInput.value;
    });

    panel.querySelector(".ncf-toggle").addEventListener("click", () => {
      panel.classList.toggle("collapsed");
      const btn = panel.querySelector(".ncf-toggle");
      btn.textContent = panel.classList.contains("collapsed") ? "+" : "\u2212";
    });

    // simple drag support on the header
    const header = panel.querySelector(".ncf-header");
    let dragging = false, offsetX = 0, offsetY = 0;
    header.addEventListener("mousedown", (e) => {
      if (e.target.closest(".ncf-toggle")) return;
      dragging = true;
      const rect = panel.getBoundingClientRect();
      offsetX = e.clientX - rect.left;
      offsetY = e.clientY - rect.top;
    });
    window.addEventListener("mousemove", (e) => {
      if (!dragging) return;
      panel.style.left = `${e.clientX - offsetX}px`;
      panel.style.top = `${e.clientY - offsetY}px`;
      panel.style.right = "auto";
    });
    window.addEventListener("mouseup", () => { dragging = false; });

    const goBtn = panel.querySelector(".ncf-go");
    if (!libsOk) {
      goBtn.disabled = true;
    }
    goBtn.addEventListener("click", async () => {
      goBtn.disabled = true;
      logEl.textContent = "";
      try {
        const asOf = new Date(panel.querySelector(".ncf-asof").value + "T00:00:00");
        const count = parseInt(panel.querySelector(".ncf-count").value, 10) || 6;
        const weekdays = Array.from(panel.querySelectorAll(".ncf-weekdays input:checked")).map((cb) =>
          parseInt(cb.value, 10)
        );
        const opacity = parseInt(opacityInput.value, 10);
        const mode = panel.querySelector('input[name="ncf-mode"]:checked').value;

        const dates = collectDates(asOf, weekdays, count);
        log(`Target dates: ${dates.map(toISODate).join(", ")}`);

        const idByDate = await fetchPuzzleIds(dates);

        const fetched = [];
        const errors = [];
        for (const d of dates) {
          const iso = toISODate(d);
          const puzzleId = idByDate[iso];
          if (puzzleId == null) {
            errors.push(`No puzzle id for ${iso}`);
            continue;
          }
          try {
            const buf = await fetchPdfBytes(puzzleId, opacity);
            fetched.push({ date: iso, buf });
          } catch (e) {
            errors.push(e.message);
          }
        }

        if (fetched.length === 0) {
          throw new Error("No puzzles could be fetched. " + errors.join("; "));
        }

        if (mode === "zip") {
          const blob = await zipPdfs(fetched);
          downloadBlob(blob, `nyt-crosswords-${fetched[0].date}-to-${fetched[fetched.length - 1].date}.zip`);
        } else {
          const merged = await mergePdfs(fetched.map((f) => f.buf));
          const blob = new Blob([merged], { type: "application/pdf" });
          downloadBlob(blob, `nyt-crosswords-${fetched[0].date}-to-${fetched[fetched.length - 1].date}.pdf`);
        }

        log(errors.length ? `Done, with ${errors.length} error(s): ${errors.join("; ")}` : "Done!");
      } catch (e) {
        log("ERROR: " + e.message);
      } finally {
        goBtn.disabled = false;
      }
    });
  }

  (async function start() {
    const diagLines = [];
    let libsOk = false;
    try {
      const result = await ensureLibs(diagLines);
      libsOk = result.ok;
    } catch (e) {
      diagLines.push(`Error while loading: ${e.message}`);
    }
    if (!libsOk) {
      console.log("[nyt-crossword-fetcher] library load diagnostics:\n" + diagLines.join("\n"));
    }
    buildPanel(libsOk, diagLines);
  })();
})();
