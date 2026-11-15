// ==UserScript==
// @name         夸克网盘文件树下载器
// @name:en      Quark Drive File Tree Downloader
// @version      1.0
// @description  扫描夸克网盘当前目录，下载一个自包含的 HTML 文件树查看器（支持多级折叠/展开、深色/浅色主题、搜索）。
// @description:en Scan the current Quark Cloud Disk folder and download a self-contained HTML file-tree viewer (collapsible tree, dark/light theme, search).
// @license      LGPL-3.0
// @author       hedefu999
// @namespace    https://github.com/hedefu999
// @icon         https://pan.quark.cn/favicon.ico
// @match        https://pan.quark.cn/*
// @grant        GM_addStyle
// @run-at       document-idle
// ==/UserScript==

(function () {
    'use strict';

    let processing = false; // 是否正在扫描
    let cancelled = false; // 是否被用户取消
    let buttonAdded = false; // 按钮是否已添加
    let fileCount = 0; // 已收集的文件数量

    const sleep = (ms) => new Promise((resolve) => setTimeout(resolve, ms));

    // ============ 夸克网盘 API 相关 ============

    // 获取当前路径信息（包括 FID 和路径名称）
    function getCurrentPathInfo() {
        let hash = window.location.hash;

        if (hash && hash.includes('/list/')) {
            let parts = hash.split('/');

            if (parts.length > 3 && parts[3] && parts[3] !== 'all') {
                let pathParts = [];
                for (let i = 3; i < parts.length; i++) {
                    let part = parts[i];
                    if (part.includes('-')) {
                        let dashIndex = part.indexOf('-');
                        if (dashIndex === 32) {
                            let fid = part.substring(0, 32);
                            let name = decodeURIComponent(part.substring(33));
                            pathParts.push({ fid: fid, name: name });
                        } else {
                            pathParts.push({ fid: '', name: decodeURIComponent(part) });
                        }
                    }
                }

                if (pathParts.length > 0) {
                    let lastPart = pathParts[pathParts.length - 1];
                    let pathStr = pathParts.map(p => p.name).join('/');
                    return {
                        fid: lastPart.fid,
                        name: lastPart.name,
                        fullPath: '/' + pathStr
                    };
                }
            }
        }

        return {
            fid: '0',
            name: '根目录',
            fullPath: '/'
        };
    }

    // 获取文件列表（支持分页）
    async function listFile(fid, page = 1, pageSize = 100) {
        try {
            let url = 'https://drive-pc.quark.cn/1/clouddrive/file/sort';

            let params = {
                pr: 'ucpro',
                fr: 'pc',
                uc_param_str: '',
                pdir_fid: fid || '0',
                _page: page,
                _size: pageSize,
                _fetch_total: 1,
                _fetch_sub_dirs: 0,
                _sort: 'file_type:asc,updated_at:desc',
                fetch_all_file: 1,
                fetch_risk_file_name: 1
            };

            let response = await fetch(url + '?' + new URLSearchParams(params), {
                method: 'GET',
                credentials: 'include',
                headers: { 'Content-Type': 'application/json' }
            });

            if (!response.ok) {
                console.error('API 请求失败:', response.status, response.statusText);
                return [];
            }

            let data = await response.json();

            if (data.status === 200 && data.data && data.data.list) {
                return data.data.list;
            } else {
                console.error('API 返回格式异常:', data);
            }

            return [];
        } catch (error) {
            console.error('获取文件列表失败:', error);
            return [];
        }
    }

    // 递归扫描目录，构建文件树（只收集文件名）
    async function scanDir(fid, name, parentPath, onDir, onFile) {
        const fullPath = parentPath ? parentPath + '/' + name : (name || '/');
        onDir(fullPath);

        const children = [];
        let page = 1;
        const pageSize = 100;

        while (processing) {
            let files = await listFile(fid, page, pageSize);

            if (!files || files.length === 0) break;

            for (const f of files) {
                if (!processing) break;

                const isDir = (f.file_type === 0 || f.dir === true);
                const fileName = f.file_name;

                if (isDir) {
                    const sub = await scanDir(f.fid, fileName, fullPath, onDir, onFile);
                    children.push(sub);
                } else {
                    onFile(fileName);
                    children.push({ type: 'file', name: fileName });
                }
            }

            if (files.length < pageSize) break;
            page++;
        }

        return { type: 'dir', name: name, children: children };
    }

    // ============ 生成 HTML 文件树查看器 ============

    function buildHtml(tree, rootName, fileCount) {
        // 将 < 转义为 <，避免文件名中混入 </script> 破坏 HTML 结构
        const treeJson = JSON.stringify(tree).replace(/</g, '\\u003c');
        const metaJson = JSON.stringify({
            rootName: rootName,
            fileCount: fileCount,
            generatedAt: new Date().toLocaleString('zh-CN', { hour12: false })
        }).replace(/</g, '\\u003c');
        const titleSafe = escapeHtml(rootName);

        return `<!DOCTYPE html>
<html lang="zh-CN" data-theme="dark">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>${titleSafe}</title>
<style>
:root {
  --bg: #f4f6f9;
  --surface: #ffffff;
  --surface-hover: #f0f4f8;
  --text: #1e293b;
  --text-muted: #64748b;
  --border: #e2e8f0;
  --accent: #3b82f6;
  --accent-soft: rgba(59, 130, 246, 0.12);
  --highlight: #fef08a;
  --highlight-text: #854d0e;
  --shadow: 0 4px 20px rgba(0, 0, 0, 0.06);
}

[data-theme="dark"] {
  --bg: #0f172a;
  --surface: #1e293b;
  --surface-hover: #334155;
  --text: #f1f5f9;
  --text-muted: #94a3b8;
  --border: #334155;
  --accent: #60a5fa;
  --accent-soft: rgba(96, 165, 250, 0.15);
  --highlight: #854d0e;
  --highlight-text: #fef08a;
  --shadow: 0 4px 20px rgba(0, 0, 0, 0.35);
}

* { box-sizing: border-box; margin: 0; padding: 0; }

body {
  font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, "Helvetica Neue", Arial, "PingFang SC", "Microsoft YaHei", sans-serif;
  background: var(--bg);
  color: var(--text);
  line-height: 1.5;
  transition: background 0.2s, color 0.2s;
  padding: 24px 16px;
}

.container {
  max-width: 1000px;
  margin: 0 auto;
}

header {
  background: var(--surface);
  border: 1px solid var(--border);
  border-radius: 14px;
  padding: 20px 24px;
  box-shadow: var(--shadow);
  margin-bottom: 16px;
}

.header-top {
  display: flex;
  justify-content: space-between;
  align-items: center;
  flex-wrap: wrap;
  gap: 12px;
  margin-bottom: 16px;
}

h1 {
  font-size: 1.35rem;
  font-weight: 700;
  display: flex;
  align-items: center;
  gap: 10px;
  word-break: break-all;
}

.stats {
  display: flex;
  gap: 16px;
  flex-wrap: wrap;
  font-size: 0.875rem;
  color: var(--text-muted);
  margin-bottom: 16px;
}

.stat-pill {
  background: var(--bg);
  padding: 4px 12px;
  border-radius: 999px;
  border: 1px solid var(--border);
}

.controls {
  display: flex;
  gap: 10px;
  flex-wrap: wrap;
}

.search-box {
  flex: 1;
  min-width: 220px;
  position: relative;
}

.search-box input {
  width: 100%;
  padding: 10px 14px;
  border-radius: 10px;
  border: 1px solid var(--border);
  background: var(--bg);
  color: var(--text);
  font-size: 0.95rem;
  outline: none;
  transition: border-color 0.2s;
}

.search-box input:focus {
  border-color: var(--accent);
}

button {
  padding: 9px 14px;
  border-radius: 10px;
  border: 1px solid var(--border);
  background: var(--surface);
  color: var(--text);
  font-size: 0.875rem;
  font-weight: 500;
  cursor: pointer;
  transition: background 0.15s, border-color 0.15s;
}

button:hover {
  background: var(--surface-hover);
  border-color: var(--accent);
}

.tree-card {
  background: var(--surface);
  border: 1px solid var(--border);
  border-radius: 14px;
  padding: 16px;
  box-shadow: var(--shadow);
  overflow-x: auto;
}

.node-row {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 6px 10px;
  border-radius: 8px;
  cursor: pointer;
  user-select: none;
  gap: 12px;
}

.node-row:hover {
  background: var(--surface-hover);
}

.node-left {
  display: flex;
  align-items: center;
  gap: 8px;
  min-width: 0;
  flex: 1;
}

.toggle-icon {
  width: 18px;
  text-align: center;
  font-size: 0.75rem;
  color: var(--text-muted);
  flex-shrink: 0;
}

.node-name {
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
  font-size: 0.925rem;
}

.node-meta {
  font-size: 0.8rem;
  color: var(--text-muted);
  white-space: nowrap;
  flex-shrink: 0;
}

.children {
  margin-left: 20px;
  padding-left: 10px;
  border-left: 1px dashed var(--border);
}

.hidden {
  display: none !important;
}

mark {
  background: var(--highlight);
  color: var(--highlight-text);
  padding: 0 2px;
  border-radius: 3px;
}

.empty-state {
  text-align: center;
  padding: 40px 20px;
  color: var(--text-muted);
}
</style>
</head>
<body>
<div class="container">
  <header>
    <div class="header-top">
      <h1>📁 <span id="root-title"></span></h1>
      <button id="themeBtn">☀️ 浅色模式</button>
    </div>
    <div class="stats" id="stats"></div>
    <div class="controls">
      <div class="search-box">
        <input type="text" id="searchInput" placeholder="搜索文件或文件夹…" autocomplete="off">
      </div>
      <button id="expandAllBtn">展开全部</button>
      <button id="collapseAllBtn">折叠全部</button>
    </div>
  </header>

  <main class="tree-card">
    <div id="treeRoot"></div>
  </main>
</div>

<script type="application/json" id="tree-data">${treeJson}</script>
<script type="application/json" id="tree-meta">${metaJson}</script>
<script>
(function () {
  'use strict';

  var root = document.documentElement;
  var treeRootEl = document.getElementById('treeRoot');
  var statsEl = document.getElementById('stats');
  var searchInput = document.getElementById('searchInput');
  var themeBtn = document.getElementById('themeBtn');
  var expandAllBtn = document.getElementById('expandAllBtn');
  var collapseAllBtn = document.getElementById('collapseAllBtn');
  var rootTitleEl = document.getElementById('root-title');

  var meta = {};
  try { meta = JSON.parse(document.getElementById('tree-meta').textContent); } catch (e) {}
  var tree = { type: 'dir', name: '', children: [] };
  try { tree = JSON.parse(document.getElementById('tree-data').textContent); } catch (e) {}

  rootTitleEl.textContent = meta.rootName || '夸克网盘文件树';

  function applyTheme(t) {
    root.setAttribute('data-theme', t);
    themeBtn.textContent = t === 'dark' ? '☀️ 浅色模式' : '🌙 深色模式';
    try { localStorage.setItem('quark-filetree-theme', t); } catch (e) {}
  }
  var saved = null;
  try { saved = localStorage.getItem('quark-filetree-theme'); } catch (e) {}
  applyTheme(saved === 'light' ? 'light' : 'dark');
  themeBtn.addEventListener('click', function () {
    applyTheme(root.getAttribute('data-theme') === 'dark' ? 'light' : 'dark');
  });

  // 预计算每个文件夹下的文件总数（用于右侧展示）
  function computeCount(node) {
    if (node.type === 'file') return 1;
    var c = node.children || [];
    var n = 0;
    for (var i = 0; i < c.length; i++) n += computeCount(c[i]);
    node._count = n;
    return n;
  }
  computeCount(tree);

  // 统计（跳过多余的根层级）
  var totalFiles = 0, totalFolders = 0;
  (function countList(list) {
    for (var i = 0; i < list.length; i++) {
      var n = list[i];
      if (n.type === 'dir') {
        totalFolders++;
        if (n.children && n.children.length) countList(n.children);
      } else {
        totalFiles++;
      }
    }
  })(tree.children || []);

  statsEl.innerHTML =
    '<span class="stat-pill">📁 文件夹: <strong>' + totalFolders + '</strong></span>' +
    '<span class="stat-pill">📄 文件: <strong>' + totalFiles + '</strong></span>' +
    '<span class="stat-pill">🕒 生成于: <strong>' + escapeHtml(meta.generatedAt || '') + '</strong></span>';

  function escapeHtml(str) {
    return String(str).replace(/[&<>"']/g, function (m) {
      return ({ '&': '&amp;', '<': '&lt;', '>': '&gt;', '"': '&quot;', "'": '&#39;' })[m];
    });
  }

  function highlightText(text, query) {
    if (!query) return escapeHtml(text);
    var lower = text.toLowerCase();
    var q = query.toLowerCase();
    var out = '';
    var pos = 0;
    var idx = lower.indexOf(q);
    while (idx !== -1) {
      out += escapeHtml(text.slice(pos, idx));
      out += '<mark>' + escapeHtml(text.slice(idx, idx + q.length)) + '</mark>';
      pos = idx + q.length;
      idx = lower.indexOf(q, pos);
    }
    out += escapeHtml(text.slice(pos));
    return out;
  }

  function buildNode(node, query, depth) {
    var isDir = node.type === 'dir';
    var children = isDir ? (node.children || []) : [];
    var nameMatches = query ? node.name.toLowerCase().indexOf(query) !== -1 : true;

    var matchingChildren = [];
    if (isDir && children.length) {
      for (var i = 0; i < children.length; i++) {
        var el = buildNode(children[i], query, depth + 1);
        if (el) matchingChildren.push(el);
      }
    }

    if (query && !nameMatches && matchingChildren.length === 0) return null;

    var wrapper = document.createElement('div');
    wrapper.className = 'tree-node';

    var row = document.createElement('div');
    row.className = 'node-row';

    var hasChildren = isDir && children.length > 0;
    var isExpanded = !!query; // 默认折叠；搜索时才自动展开
    var icon = isDir ? (hasChildren ? (isExpanded ? '▼' : '▶') : '•') : '';
    var emoji = isDir ? '📁' : '📄';
    var metaText = isDir ? (node._count || 0) + ' 个文件' : '';

    row.innerHTML =
      '<div class="node-left">' +
        '<span class="toggle-icon">' + icon + '</span>' +
        '<span>' + emoji + '</span>' +
        '<span class="node-name" title="' + escapeHtml(node.name) + '">' + highlightText(node.name, query) + '</span>' +
      '</div>' +
      (metaText ? '<div class="node-meta">' + metaText + '</div>' : '');

    wrapper.appendChild(row);

    if (hasChildren) {
      var cc = document.createElement('div');
      cc.className = 'children' + (isExpanded ? '' : ' hidden');

      if (query && matchingChildren.length) {
        for (var j = 0; j < matchingChildren.length; j++) cc.appendChild(matchingChildren[j]);
      } else if (!query) {
        for (var k = 0; k < children.length; k++) {
          var cel = buildNode(children[k], '', depth + 1);
          if (cel) cc.appendChild(cel);
        }
      }

      row.addEventListener('click', function () {
        var isHidden = cc.classList.toggle('hidden');
        row.querySelector('.toggle-icon').textContent = isHidden ? '▶' : '▼';
      });
      wrapper.appendChild(cc);
    }

    return wrapper;
  }

  function render(query) {
    treeRootEl.innerHTML = '';
    var q = query ? query.trim().toLowerCase() : '';
    var frag = document.createDocumentFragment();
    var any = false;
    var list = tree.children || [];
    for (var i = 0; i < list.length; i++) {
      var el = buildNode(list[i], q, 0);
      if (el) { frag.appendChild(el); any = true; }
    }
    if (any) treeRootEl.appendChild(frag);
    else treeRootEl.innerHTML = '<div class="empty-state">没有匹配 “' + escapeHtml(q) + '” 的文件或文件夹</div>';
  }

  var timer = null;
  searchInput.addEventListener('input', function () {
    clearTimeout(timer);
    timer = setTimeout(function () { render(searchInput.value); }, 180);
  });
  searchInput.addEventListener('keydown', function (e) {
    if (e.key === 'Escape') { searchInput.value = ''; render(''); }
  });

  function setAllExpanded(expand) {
    var kids = treeRootEl.querySelectorAll('.children');
    for (var i = 0; i < kids.length; i++) kids[i].classList.toggle('hidden', !expand);
    var togs = treeRootEl.querySelectorAll('.toggle-icon');
    for (var j = 0; j < togs.length; j++) {
      if (togs[j].textContent === '▶' || togs[j].textContent === '▼') {
        togs[j].textContent = expand ? '▼' : '▶';
      }
    }
  }
  expandAllBtn.addEventListener('click', function () { setAllExpanded(true); });
  collapseAllBtn.addEventListener('click', function () { setAllExpanded(false); });

  render('');
})();
</script>
</body>
</html>`;
    }

    // ============ 下载与进度提示 ============

    function escapeHtml(s) {
        return String(s).replace(/&/g, '&amp;').replace(/</g, '&lt;').replace(/>/g, '&gt;').replace(/"/g, '&quot;');
    }

    function downloadHtml(html, filename) {
        const blob = new Blob([html], { type: 'text/html;charset=utf-8' });
        const url = URL.createObjectURL(blob);
        const a = document.createElement('a');
        a.href = url;
        a.download = filename;
        a.style.display = 'none';
        document.body.appendChild(a);
        a.click();
        document.body.removeChild(a);
        setTimeout(() => URL.revokeObjectURL(url), 10000);
    }

    function buildFilename(rootName) {
        // 只保留文件夹名本身，仅把 Windows 下非法的文件名字符替换掉
        const safe = String(rootName || '根目录').replace(/[\\/:*?"<>|]/g, '_');
        return safe + '.html';
    }

    function showProgress() {
        if (document.getElementById('ftd-progress')) return;
        const box = document.createElement('div');
        box.id = 'ftd-progress';
        box.innerHTML =
            '<div id="ftd-progress-dir">准备扫描…</div>' +
            '<div id="ftd-progress-file">已收集 0 个文件</div>' +
            '<div id="ftd-progress-stop">停止扫描</div>';
        document.body.appendChild(box);
        document.getElementById('ftd-progress-stop').addEventListener('click', () => {
            cancelled = true;
            processing = false;
        });
    }

    function hideProgress() {
        const box = document.getElementById('ftd-progress');
        if (box) box.remove();
    }

    function updateDirText(text) {
        const el = document.getElementById('ftd-progress-dir');
        if (el) el.textContent = '目录: ' + text;
    }

    function updateFileText() {
        const el = document.getElementById('ftd-progress-file');
        if (el) el.textContent = '已收集 ' + fileCount + ' 个文件';
    }

    // ============ 主流程 ============

    async function run() {
        if (processing) {
            alert('正在扫描中，请稍候…');
            return;
        }

        const pathInfo = getCurrentPathInfo();
        processing = true;
        cancelled = false;
        fileCount = 0;
        showProgress();

        let tree = null;
        try {
            tree = await scanDir(
                pathInfo.fid,
                pathInfo.name,
                '',
                (dir) => updateDirText(dir),
                () => { fileCount++; updateFileText(); }
            );
        } catch (e) {
            console.error('扫描出错:', e);
            processing = false;
            hideProgress();
            alert('扫描出错: ' + e.message);
            return;
        }

        processing = false;

        if (cancelled) {
            hideProgress();
            console.log('扫描已被用户取消');
            return;
        }

        if (!tree || !tree.children || tree.children.length === 0) {
            hideProgress();
            alert('当前目录为空或未获取到数据，请重试。');
            return;
        }

        updateDirText('正在生成 HTML 文件…');
        try {
            const html = buildHtml(tree, pathInfo.name, fileCount);
            downloadHtml(html, buildFilename(pathInfo.name));
            updateDirText('完成！共 ' + fileCount + ' 个文件');
        } catch (e) {
            console.error('生成 HTML 出错:', e);
            alert('生成 HTML 出错: ' + e.message);
        }

        setTimeout(hideProgress, 1500);
    }

    // ============ 按钮创建与插入 ============

    function createButton() {
        const btn = document.createElement('button');
        btn.id = 'ftd_button';
        btn.title = '扫描当前目录并下载文件树 HTML 查看器';

        // 复制“新建文件夹”按钮的样式类，使颜色/尺寸与之一致
        const ref = findCreateFolderButton();
        if (ref && ref.className && String(ref.className).trim()) {
            btn.className = ref.className;
        } else {
            btn.className = 'u-button nd-file-list-toolbar-action-item is-need-left-sep u-button--default u-button--small is-has-icon';
        }

        btn.innerHTML =
            '<svg class="ftd-icon" viewBox="0 0 24 24" width="14" height="14" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round" style="vertical-align:-2px;">' +
            '<path d="M21 15v4a2 2 0 0 1-2 2H5a2 2 0 0 1-2-2v-4"></path>' +
            '<polyline points="7 10 12 15 17 10"></polyline>' +
            '<line x1="12" y1="15" x2="12" y2="3"></line>' +
            '</svg>' +
            '<span>下载文件列表</span>';

        btn.addEventListener('click', function (e) {
            e.preventDefault();
            e.stopPropagation();
            run();
        });
        return btn;
    }

    function findCreateFolderButton() {
        const el = document.querySelector('.btn-create-folder');
        if (!el) return null;
        if (el.tagName === 'BUTTON') return el;
        return el.querySelector('button');
    }

    function insertButton(referenceElement) {
        const btn = createButton();
        referenceElement.parentNode.insertBefore(btn, referenceElement.nextSibling);
    }

    function appendButton(container) {
        const btn = createButton();
        container.appendChild(btn);
    }

    function addFloatingButton() {
        if (buttonAdded || document.getElementById('ftd_button')) return;
        const btn = createButton();
        btn.style.cssText = 'position:fixed;top:130px;right:20px;z-index:9999;box-shadow:0 2px 8px rgba(0,0,0,0.18);';
        document.body.appendChild(btn);
        buttonAdded = true;
    }

    function start() {
        if (buttonAdded || document.getElementById('ftd_button')) return true;

        const containers = [
            '.btn-create-folder',
            '.btn-file',
            '.header-toolbar',
            '.ant-space'
        ];

        for (let i = 0; i < containers.length; i++) {
            const el = document.querySelector(containers[i]);
            if (el) {
                try {
                    const action = i < 2 ? insertButton : appendButton;
                    action(el);
                    buttonAdded = true;
                    return true;
                } catch (e) {
                    console.error('插入按钮失败:', e);
                }
            }
        }

        return false;
    }

    // 监听 DOM 变化，在页面动态加载后插入按钮
    const observer = new MutationObserver(function () {
        if (!buttonAdded) start();
    });

    (async function init() {
        await sleep(2000);
        let success = start();
        if (!success) {
            observer.observe(document.body, { childList: true, subtree: true });
            await sleep(10000);
            if (!buttonAdded) {
                addFloatingButton();
                observer.disconnect();
            }
        } else {
            observer.disconnect();
        }
    })();

    // ============ 样式 ============

    GM_addStyle(`
    #ftd_button {
        display: inline-flex;
        align-items: center;
        gap: 4px;
        margin-left: 8px;
        cursor: pointer;
    }
    #ftd_button .ftd-icon {
        flex-shrink: 0;
    }

    #ftd-progress {
        position: fixed;
        right: 16px;
        bottom: 16px;
        z-index: 99999;
        width: 340px;
        background: #111827;
        color: #e5e7eb;
        border: 1px solid #374151;
        border-radius: 10px;
        box-shadow: 0 8px 30px rgba(0,0,0,0.35);
        font-size: 13px;
        overflow: hidden;
    }
    #ftd-progress-dir {
        padding: 10px 12px;
        word-break: break-all;
        background: #1f2937;
        font-weight: 600;
    }
    #ftd-progress-file {
        padding: 10px 12px;
        color: #93c5fd;
    }
    #ftd-progress-stop {
        padding: 10px 12px;
        text-align: center;
        cursor: pointer;
        background: #b45309;
        color: #fff;
        font-weight: 600;
        user-select: none;
    }
    #ftd-progress-stop:hover {
        background: #92400e;
    }
    `);
})();
