<!doctype html>
<html lang="en">
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1" />
  <title>HTML Launcher</title>
  <style>
    body { font-family: system-ui, -apple-system, Segoe UI, Roboto, Arial; margin: 0; background: #0f1117; color: #e7e7e7; }
    header { padding: 18px 16px; border-bottom: 1px solid #232a3a; background: #151826; }
    h1 { margin: 0; font-size: 18px; }
    .sub { margin-top: 6px; color: #9aa4b2; font-size: 13px; }
    main { padding: 16px; max-width: 980px; margin: 0 auto; }
    .row { display: flex; gap: 10px; flex-wrap: wrap; align-items: center; }
    input { flex: 1; min-width: 260px; padding: 10px 12px; border-radius: 10px; border: 1px solid #273055; background: #0f1322; color: #e7e7e7; }
    button { padding: 10px 12px; border-radius: 10px; border: 1px solid #273055; background: #121528; color: #e7e7e7; cursor: pointer; font-weight: 700; }
    button:hover { border-color: #3a4a85; }
    .card { margin-top: 14px; padding: 14px; border-radius: 14px; border: 1px solid #1f2638; background: #0f1322; }
    .muted { color: #9aa4b2; font-size: 13px; }
    ul { list-style: none; padding: 0; margin: 0; }
    li { padding: 10px 10px; border-radius: 12px; display: flex; justify-content: space-between; gap: 12px; align-items: center; }
    li:hover { background: rgba(16,20,40,.6); }
    a { color: #8fb5ff; text-decoration: none; font-weight: 800; }
    a:hover { text-decoration: underline; }
    .pill { font-size: 12px; padding: 3px 8px; border-radius: 999px; border: 1px solid #2a3455; color: #9aa4b2; }
    .error { color: #ff9aa2; white-space: pre-wrap; }
    .footer { margin-top: 16px; color: #9aa4b2; font-size: 12px; }
  </style>
</head>
<body>
  <header>
    <h1>HTML Launcher</h1>
    <div class="sub">Browse and open HTML files from this GitHub Pages repo</div>
  </header>

  <main>
    <div class="row">
      <input id="path" placeholder="Start path (e.g. / or demos or docs)" />
      <button id="reload">Reload</button>
      <button id="toggle">Recursive: ON</button>
    </div>

    <div class="card">
      <div class="muted" id="status">Loading…</div>
      <div class="error" id="err" style="display:none;"></div>
      <ul id="list" style="margin-top:12px;"></ul>
      <div class="footer">
        Tip: If you want this to live alongside a README, you can link to <code>index.html</code> from the README.
      </div>
    </div>
  </main>

  <script>
    // ======= CONFIG =======
   
     function detectRepoFromLocation() {
  const host = window.location.hostname;
  const path = window.location.pathname.split("/").filter(Boolean);

  // USERNAME.github.io
  if (host.endsWith(".github.io")) {
    const owner = host.replace(".github.io", "");

    // https://username.github.io/repo/
    if (path.length > 0) {
      return { owner, repo: path[0] };
    }

    // https://username.github.io/ (user page)
    return { owner, repo: null };
  }

  return { owner: null, repo: null };
}

const detected = detectRepoFromLocation();

if (!detected.owner || !detected.repo) {
  throw new Error("Could not auto-detect GitHub repo from URL.");
}

const OWNER = detected.owner;
const REPO  = detected.repo;


    // If your GitHub Pages is built from /docs or a subfolder, set BASE_PATH.
    // For normal Pages from root: ""  (example: https://user.github.io/repo)
    const BASE_PATH = ""; 

    // Default folder to scan for HTML files ("" means repo root)
    const DEFAULT_START_PATH = ""; // e.g. "demos" or "docs" or ""

    // ======= STATE =======
    let recursive = true;

    const $ = (id) => document.getElementById(id);
    const statusEl = $("status");
    const errEl = $("err");
    const listEl = $("list");

    function setStatus(msg) {
      statusEl.textContent = msg;
    }
    function setError(msg) {
      errEl.style.display = msg ? "block" : "none";
      errEl.textContent = msg || "";
    }

    function normalizePath(p) {
      p = (p || "").trim();
      p = p.replace(/^\/+/, "").replace(/\/+$/, "");
      return p;
    }

    function pagesUrlForFile(filePath) {
      // filePath is repo-relative (e.g. "demos/test.html")
      // Pages URL: BASE_PATH + "/" + filePath
      const prefix = BASE_PATH ? (BASE_PATH.replace(/\/+$/,"")) : "";
      return prefix + "/" + filePath;
    }

    async function ghFetchJson(url) {
      const res = await fetch(url, {
        headers: {
          "Accept": "application/vnd.github+json"
        }
      });
      const text = await res.text();
      let data;
      try { data = JSON.parse(text); } catch { data = null; }

      if (!res.ok) {
        const msg = (data && (data.message || data.error)) ? (data.message || data.error) : text;
        throw new Error(`GitHub API error (${res.status}): ${msg}`);
      }
      return data;
    }

    async function listHtmlFiles(startPath) {
      const start = normalizePath(startPath);
      const url = `https://api.github.com/repos/${OWNER}/${REPO}/contents/${start ? start : ""}`;
      const root = await ghFetchJson(url);

      const out = [];

      async function walk(node, basePath) {
        if (Array.isArray(node)) {
          for (const item of node) {
            if (item.type === "file" && item.name.toLowerCase().endsWith(".html")) {
              out.push({
                name: item.name,
                path: item.path,
                size: item.size || 0
              });
            } else if (item.type === "dir" && recursive) {
              const child = await ghFetchJson(`https://api.github.com/repos/${OWNER}/${REPO}/contents/${item.path}`);
              await walk(child, item.path);
            }
          }
        } else if (node && node.type === "file") {
          if (node.name.toLowerCase().endsWith(".html")) {
            out.push({ name: node.name, path: node.path, size: node.size || 0 });
          }
        }
      }

      await walk(root, start);

      // Sort: folder depth, then name
      out.sort((a,b) => a.path.localeCompare(b.path));
      return out;
    }

    function render(files, startPath) {
      listEl.innerHTML = "";
      if (!files.length) {
        listEl.innerHTML = `<li><span class="muted">No .html files found in "${startPath || "/"}"</span></li>`;
        return;
      }

      for (const f of files) {
        const li = document.createElement("li");
        const url = pagesUrlForFile(f.path);
        li.innerHTML = `
          <div style="min-width:0;">
            <a href="${url}">${escapeHtml(f.path)}</a>
            <div class="muted">${escapeHtml(url)}</div>
          </div>
          <span class="pill">${Math.max(1, Math.round((f.size||0)/1024))} KB</span>
        `;
        listEl.appendChild(li);
      }
    }

    function escapeHtml(s) {
      return (s || "").replace(/[&<>"']/g, c => ({
        "&":"&amp;","<":"&lt;",">":"&gt;",'"':"&quot;","'":"&#039;"
      })[c]);
    }

    async function load() {
      setError("");
      const startPath = normalizePath($("path").value);
      setStatus(`Loading .html files from "${startPath || "/"}"…`);

      try {
        const files = await listHtmlFiles(startPath);
        setStatus(`Found ${files.length} HTML file(s) in "${startPath || "/"}"`);
        render(files, startPath);
      } catch (e) {
        setStatus("Failed to load.");
        setError(String(e));
      }
    }

    // UI
    $("reload").onclick = load;
    $("toggle").onclick = () => {
      recursive = !recursive;
      $("toggle").textContent = `Recursive: ${recursive ? "ON" : "OFF"}`;
      load();
    };

    // init
    $("path").value = DEFAULT_START_PATH;
    load();
  </script>
</body>
</html>

