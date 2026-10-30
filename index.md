---
layout: default
title: UM Study Materials
---

<script src="https://cdn.jsdelivr.net/npm/jszip@3.10.1/dist/jszip.min.js"></script>
<script src="https://cdn.jsdelivr.net/npm/file-saver@2.0.5/dist/FileSaver.min.js"></script>

<div id="app" style="margin-top: 20px;">
  <p>Loading course materials...</p>
</div>

{% raw %}
<script>
  const OWNER = 'TebelMeners';
  const REPO = 'UMStudyMaterials';
  const BRANCH = 'main';

  const IGNORE_FILES = ['readme.md', 'index.md', '_config.yml', 'desktop.ini', 'thumbs.db', '.ds_store'];

  function loadScript(src) {
    return new Promise((resolve, reject) => {
      const script = document.createElement('script');
      script.src = src;
      script.onload = resolve;
      script.onerror = () => reject(new Error(`Failed to load script from ${src}`));
      document.head.appendChild(script);
    });
  }

  async function ensureLibrariesLoaded() {
    if (typeof JSZip === 'undefined') {
      await loadScript('https://cdn.jsdelivr.net/npm/jszip@3.10.1/dist/jszip.min.js');
    }
    if (typeof saveAs === 'undefined') {
      await loadScript('https://cdn.jsdelivr.net/npm/file-saver@2.0.5/dist/FileSaver.min.js');
    }
  }

  function naturalSort(a, b) {
    return a.localeCompare(b, undefined, { numeric: true, sensitivity: 'base' });
  }

  function isIgnored(path) {
    const filename = path.split('/').pop().toLowerCase();
    return filename.startsWith('.') || IGNORE_FILES.includes(filename);
  }

  // Resolves both standard Git files and Git LFS binary pointers automatically
  function getFileUrl(path) {
    const encodedSegments = path.split('/').map(segment => encodeURIComponent(segment)).join('/');
    return `https://github.com/${OWNER}/${REPO}/raw/${BRANCH}/${encodedSegments}`;
  }

  async function fetchTreeData() {
    const cacheKey = `gh_tree_${OWNER}_${REPO}_${BRANCH}`;
    const cached = sessionStorage.getItem(cacheKey);
    if (cached) {
      try { return JSON.parse(cached); } catch (e) { sessionStorage.removeItem(cacheKey); }
    }

    const res = await fetch(`https://api.github.com/repos/${OWNER}/${REPO}/git/trees/${BRANCH}?recursive=1`);
    if (!res.ok) {
      throw new Error(`GitHub Tree API error (HTTP ${res.status})`);
    }

    const data = await res.json();
    if (data && data.tree) {
      try { sessionStorage.setItem(cacheKey, JSON.stringify(data.tree)); } catch (e) {}
      return data.tree;
    }
    throw new Error('Invalid repository data received.');
  }

  async function init() {
    const app = document.getElementById('app');
    const urlParams = new URLSearchParams(window.location.search);
    const currentPath = (urlParams.get('path') || urlParams.get('folder') || '').trim().replace(/\/+$|^\/+/g, '');

    try {
      const tree = await fetchTreeData();
      renderPathView(app, tree, currentPath);
    } catch (e) {
      app.innerHTML = `<p style="color: #cb2431; font-weight: bold;">Error loading directory: ${e.message}</p>`;
    }
  }

  function renderPathView(container, items, currentPath) {
    const subfolders = new Set();
    const directFiles = [];
    const prefix = currentPath ? currentPath + '/' : '';

    items.forEach(item => {
      if (isIgnored(item.path)) return;

      if (!currentPath) {
        const parts = item.path.split('/');
        if (parts.length > 1 || item.type === 'tree') {
          subfolders.add(parts[0]);
        } else if (parts.length === 1 && item.type === 'blob') {
          directFiles.push({ name: item.path, path: item.path });
        }
      } else {
        if (item.path.startsWith(prefix)) {
          const relPath = item.path.slice(prefix.length);
          const parts = relPath.split('/');

          if (parts.length > 1 || item.type === 'tree') {
            subfolders.add(parts[0]);
          } else if (parts.length === 1 && item.type === 'blob') {
            directFiles.push({ name: parts[0], path: item.path });
          }
        }
      }
    });

    let html = '';
    html += renderBreadcrumbs(currentPath);

    if (currentPath) {
      const folderName = currentPath.split('/').pop();
      const safePath = encodeURIComponent(currentPath);
      html += `
        <div style="display: flex; justify-content: space-between; align-items: center; border-bottom: 2px solid #eaecef; padding-bottom: 8px; margin-bottom: 20px; flex-wrap: wrap; gap: 10px;">
          <h2 style="margin: 0;">📁 ${folderName}</h2>
          <button id="downloadBtn" data-path="${safePath}" onclick="downloadFolderZip(decodeURIComponent(this.dataset.path))" style="background-color: #2ea44f; color: white; border: none; padding: 8px 16px; border-radius: 6px; font-weight: bold; cursor: pointer;">
            📦 Download Folder (.zip)
          </button>
        </div>
      `;
    } else {
      html += `<h2 style="border-bottom: 2px solid #eaecef; padding-bottom: 8px; margin-bottom: 20px;">Course Modules</h2>`;
    }

    if (subfolders.size > 0) {
      html += `<div style="display: grid; grid-template-columns: repeat(auto-fill, minmax(200px, 1fr)); gap: 14px; margin-bottom: 24px;">`;
      
      Array.from(subfolders).sort(naturalSort).forEach(folder => {
        const nextPath = currentPath ? `${currentPath}/${folder}` : folder;
        html += `
          <a href="?path=${encodeURIComponent(nextPath)}" style="display: flex; align-items: center; padding: 14px 18px; border: 1px solid #e1e4e8; border-radius: 8px; text-decoration: none; color: #0366d6; background: #f6f8fa; font-weight: 600; transition: background 0.2s ease;">
            <span style="font-size: 1.2rem; margin-right: 8px;">📁</span> ${folder}
          </a>
        `;
      });
      html += `</div>`;
    }

    if (directFiles.length > 0) {
      html += `<h3 style="margin-top: 10px; margin-bottom: 12px; font-size: 1.1rem; color: #57606a;">Files</h3>`;
      html += `<ul style="list-style: none; padding-left: 0;">`;
      
      directFiles.sort((a, b) => naturalSort(a.name, b.name)).forEach(file => {
        const fileUrl = getFileUrl(file.path);
        html += `
          <li style="padding: 10px 14px; border-bottom: 1px solid #f0f0f0; display: flex; align-items: center; justify-content: space-between; flex-wrap: wrap; gap: 8px;">
            <div style="display: flex; align-items: center; flex: 1; min-width: 200px;">
              <span style="margin-right: 10px;">📄</span>
              <a href="${fileUrl}" target="_blank" style="color: #0366d6; text-decoration: none; font-weight: 500; word-break: break-word;">${file.name}</a>
            </div>
            <a href="${fileUrl}" download="${file.name}" style="font-size: 0.85rem; color: #0366d6; text-decoration: none; border: 1px solid #d1d5da; padding: 4px 10px; border-radius: 6px; background-color: #f6f8fa; font-weight: 500; white-space: nowrap;">
              📥 Direct Download
            </a>
          </li>
        `;
      });
      html += `</ul>`;
    }

    if (subfolders.size === 0 && directFiles.length === 0) {
      html += `<p style="color: #6a737d;">This folder is empty.</p>`;
    }

    container.innerHTML = html;
  }

  function renderBreadcrumbs(currentPath) {
    let breadcrumbHtml = `<div style="margin-bottom: 20px; font-size: 1rem; color: #586069;">`;
    breadcrumbHtml += `<a href="./" style="color: #0366d6; text-decoration: none; font-weight: bold;">🏠 Home</a>`;

    if (currentPath) {
      const parts = currentPath.split('/');
      let accumulatedPath = '';
      
      parts.forEach((part, index) => {
        accumulatedPath += (index === 0 ? '' : '/') + part;
        breadcrumbHtml += ` <span style="color: #a3aab1; margin: 0 6px;">/</span> `;
        
        if (index === parts.length - 1) {
          breadcrumbHtml += `<span style="font-weight: bold; color: #24292e;">${part}</span>`;
        } else {
          breadcrumbHtml += `<a href="?path=${encodeURIComponent(accumulatedPath)}" style="color: #0366d6; text-decoration: none;">${part}</a>`;
        }
      });
    }

    breadcrumbHtml += `</div>`;
    return breadcrumbHtml;
  }

  async function downloadFolderZip(folderPath) {
    const btn = document.getElementById('downloadBtn');
    btn.innerText = '⏳ Zipping files...';
    btn.disabled = true;

    try {
      await ensureLibrariesLoaded();

      const tree = await fetchTreeData();
      const prefix = folderPath + '/';
      const folderFiles = tree.filter(item => item.type === 'blob' && item.path.startsWith(prefix) && !isIgnored(item.path));

      if (folderFiles.length === 0) {
        alert('No downloadable files found in this folder.');
        btn.innerText = '📦 Download Folder (.zip)';
        btn.disabled = false;
        return;
      }

      const zip = new JSZip();

      for (const file of folderFiles) {
        const fileUrl = getFileUrl(file.path);
        const fileRes = await fetch(fileUrl);
        
        if (!fileRes.ok) {
          throw new Error(`Failed to download ${file.path.split('/').pop()} (HTTP ${fileRes.status})`);
        }

        const blob = await fileRes.blob();
        const relativePath = file.path.slice(prefix.length);
        zip.file(relativePath, blob);
      }

      const zipBlob = await zip.generateAsync({ type: 'blob', mimeType: 'application/zip' });
      const folderName = folderPath.split('/').pop();
      
      saveAs(zipBlob, `${folderName}.zip`);

      btn.innerText = '📦 Download Folder (.zip)';
      btn.disabled = false;
    } catch (e) {
      alert(`Download failed: ${e.message}`);
      btn.innerText = '📦 Download Folder (.zip)';
      btn.disabled = false;
    }
  }

  init();
</script>
{% endraw %}