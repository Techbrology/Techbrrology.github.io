<!doctype html>
<html lang="en">
<head>
  <meta charset="utf-8">
  <title>Project Launcher</title>
  <meta name="viewport" content="width=device-width, initial-scale=1">

  <style>
    body {
      margin: 0;
      font-family: system-ui, -apple-system, Segoe UI, Roboto, Arial;
      background: #0f1117;
      color: #e7e7e7;
    }

    header {
      padding: 20px;
      border-bottom: 1px solid #232a3a;
      background: #151826;
    }

    h1 {
      margin: 0;
      font-size: 20px;
    }

    .sub {
      margin-top: 6px;
      font-size: 13px;
      color: #9aa4b2;
    }

    main {
      max-width: 1000px;
      margin: 0 auto;
      padding: 20px;
    }

    .card {
      background: #0f1322;
      border: 1px solid #1f2638;
      border-radius: 14px;
      padding: 16px;
    }

    ul {
      list-style: none;
      padding: 0;
      margin: 0;
    }

    li {
      padding: 12px;
      border-radius: 12px;
      display: flex;
      justify-content: space-between;
      align-items: center;
      gap: 12px;
    }

    li:hover {
      background: rgba(16,20,40,.6);
    }

    a {
      color: #8fb5ff;
      text-decoration: none;
      font-weight: 800;
    }

    a:hover {
      text-decoration: underline;
    }

    .muted {
      font-size: 12px;
      color: #9aa4b2;
      word-break: break-all;
    }

    .pill {
      font-size: 12px;
      padding: 4px 8px;
      border-radius: 999px;
      border: 1px solid #2a3455;
      color: #9aa4b2;
      white-space: nowrap;
    }

    .error {
      color: #ff9aa2;
      white-space: pre-wrap;
      margin-top: 10px;
    }
  </style>
</head>
<body>

<header>
  <h1>HTML Project Launcher</h1>
  <div class="sub">
    techbrology • GitHub Pages
  </div>
</header>

<main>
  <div class="card">
    <div class="muted" id="status">Loading…</div>
    <div class="error" id="error" style="display:none;"></div>
    <ul id="list" style="margin-top:12px;"></ul>
  </div>
</main>

<script>
/* ============================
   AUTO-DETECT REPO
============================ */

function detectRepo() {
  const host = location.hostname;
  const pathParts = location.pathname.split("/").filter(Boolean);

  // techbrology.github.io/repo-name/
  if (host === "techbrology.github.io" && pathParts.length > 0) {
    return {
      owner: "techbrology",
      repo: pathParts[0]
    };
  }

  return null;
}

const detected = detectRepo();

if (!detected) {
  document.getElementById("error").style.display = "block";
  document.getElementById("error").textContent =
    "Could not detect repository from URL. This page must be served from techbrology.github.io/<repo>/";
  throw new Error("Repo detection failed");
}

const OWNER = detected.owner;
const REPO  = detected.repo;

/* ============================
   GITHUB API
============================ */

const statusEl = document.getElementById("status");
const errorEl  = document.getElementById("error");
const listEl   = document.getElementById("list");

async function fetchJson(url) {
  const res = await fetch(url, {
    headers: { "Accept": "application/vnd.github+json" }
  });

  const data = await res.json();
  if (!res.ok) {
    throw new Error(`GitHub API error ${res.status}: ${data.message}`);
  }
  return data;
}

async function findHtmlFiles(path = "") {
  const url = `https://api.github.com/repos/${OWNER}/${REPO}/contents/${path}`;
  const items = await fetchJson(url);
  let files = [];

  for (const item of items) {
    if (item.type === "file" && item.name.toLowerCase().endsWith(".html")) {
      files.push(item);
    }
    if (item.type === "dir") {
      const sub = await findHtmlFiles(item.path);
      files = files.concat(sub);
    }
  }

  return files;
}

/* ============================
   RENDER
============================ */

function render(files) {
  listEl.innerHTML = "";

  if (!files.length) {
    listEl.innerHTML = `<li class="muted">No HTML files found.</li>`;
    return;
  }

  for (const f of files) {
    const li = document.createElement("li");
    const url = `/${REPO}/${f.path}`;

    li.innerHTML = `
      <div>
        <a href="${url}">${f.path}</a>
        <div class="muted">${url}</div>
      </div>
      <span class="pill">HTML</span>
    `;

    listEl.appendChild(li);
  }
}

/* ============================
   INIT
============================ */

(async () => {
  try {
    statusEl.textContent = `Loading HTML projects from "${REPO}"…`;
    const files = await findHtmlFiles();
    statusEl.textContent = `Found ${files.length} HTML project(s)`;
    render(files);
  } catch (err) {
    statusEl.textContent = "Failed to load.";
    errorEl.style.display = "block";
    errorEl.textContent = err.message;
  }
})();
</script>

</body>
</html>
