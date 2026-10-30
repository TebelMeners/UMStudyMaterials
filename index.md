---
layout: default
title: UM Study Materials
---

<script src="https://cdnjs.cloudflare.com/ajax/libs/jszip/3.10.1/jszip.min.js" integrity="sha512-XMVd28F1ACc/Xgy5/GQNLUB36215iX226WpQQH11C/xJ6zYNf/vEaLz410F03M/Ld++Yn2/0m==" crossorigin="anonymous" referrerpolicy="no-referrer"></script>
<script src="https://cdnjs.cloudflare.com/ajax/libs/FileSaver.js/2.0.5/FileSaver.min.js" crossorigin="anonymous" referrerpolicy="no-referrer"></script>

<div id="app" style="margin-top: 20px;">
  <p>Loading course materials...</p>
</div>

<script>
  const OWNER = 'TebelMeners';
  const REPO = 'UMStudyMaterials';
  const BRANCH = 'main';

  const IGNORE_FILES = ['readme.md', 'index.md', '_config.yml', 'desktop.ini', 'thumbs.db', '.ds_store'];

  // Natural human sorting (e.g. "Lecture 2" before "Lecture 10")
  function naturalSort(a, b) {
    return a.localeCompare(b, undefined, { numeric: true, sensitivity: 'base' });
  }

  function isIgnored(path) {
    const filename = path.split('/').pop().toLowerCase();
    return filename.startsWith('.') || IGNORE_FILES.includes(filename);
  }

  // Generates absolute, segment-encoded same-origin URLs
  function getEncodedUrl(path) {
    const baseUrl = window.location.origin + window.location.pathname.replace(/\/+$/, '');
    const encodedSegments = path.split('/').map(segment => encodeURIComponent(segment)).join('/');
    return `${baseUrl}/${encodedSegments}`;
  }

  async function init() {
    const app = document.getElementById('app');
    const urlParams = new URLSearchParams(window.location.search);
    const currentPath = (urlParams.get('path') || urlParams.get('folder') || '').trim().replace(/\/+$|^\/+/g, '');

    try {
      const res = await fetch(`https://api.github.com/repos/${OWNER}/${REPO}/git/trees/${BRANCH}?recursive=1`);
      
      if (!res.ok) {
        throw new Error(`GitHub API error (HTTP ${res.status})`);
      }

      const data = await res.json();

      if (!data.tree) {
        app.innerHTML = '<p>Error loading repository data.</p>';
        return;
      }

      renderPathView(app, data.tree, currentPath);
    } catch (e) {
      app.innerHTML = `<p>Error loading repository contents: ${e.message}</p>`;
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
      html += `
        <div style="display: flex; justify-content: space-between; align-items: center; border-bottom: 2px solid #eaecef; padding-bottom: 8px; margin-bottom: 20px; flex-wrap: wrap; gap: 10px;">
          <h2 style="margin: 0;">📁 ${folderName}</h2>
          <button id="downloadBtn" onclick="downloadFolderZip('${currentPath}')" style="background-color: #2ea44f; color: white; border: none; padding: 8px 16px; border-radius: 6px; font-weight: bold; cursor: pointer;">
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
        const fileUrl = getEncodedUrl(file.path);
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
      const res = await fetch(`https://api.github.com/repos/${OWNER}/${REPO}/git/trees/${BRANCH}?recursive=1`);
      
      if (!res.ok) {
        throw new Error(`GitHub Tree API limit reached or unreachable (HTTP ${res.status})`);
      }

      const data = await res.json();
      
      const prefix = folderPath + '/';
      const folderFiles = data.tree.filter(item => item.type === 'blob' && item.path.startsWith(prefix) && !isIgnored(item.path));

      if (folderFiles.length === 0) {
        alert('No downloadable files found in this folder.');
        btn.innerText = '📦 Download Folder (.zip)';
        btn.disabled = false;
        return;
      }

      const zip = new JSZip();

      for (const file of folderFiles) {
        const fileUrl = getEncodedUrl(file.path);
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