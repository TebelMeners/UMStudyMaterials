---
layout: default
title: UM Study Materials
---

<div id="app" style="margin-top: 20px;">
  <p>Loading course materials...</p>
</div>

<script>
  const OWNER = 'TebelMeners';
  const REPO = 'UMStudyMaterials';

  // Files to exclude from site listings
  const IGNORE_FILES = ['readme.md', 'index.md', '_config.yml', 'desktop.ini', 'thumbs.db', '.ds_store'];

  function isIgnored(path) {
    const filename = path.split('/').pop().toLowerCase();
    return filename.startsWith('.') || IGNORE_FILES.includes(filename);
  }

  async function init() {
    const app = document.getElementById('app');
    const urlParams = new URLSearchParams(window.location.search);
    const currentPath = (urlParams.get('path') || urlParams.get('folder') || '').trim().replace(/\/+$|^\/+/g, '');

    try {
      const res = await fetch(`https://api.github.com/repos/${OWNER}/${REPO}/git/trees/main?recursive=1`);
      const data = await res.json();

      if (!data.tree) {
        app.innerHTML = '<p>Error loading repository data.</p>';
        return;
      }

      renderPathView(app, data.tree, currentPath);
    } catch (e) {
      app.innerHTML = '<p>Error loading repository contents.</p>';
    }
  }

  function renderPathView(container, items, currentPath) {
    const subfolders = new Set();
    const directFiles = [];
    const prefix = currentPath ? currentPath + '/' : '';

    items.forEach(item => {
      if (isIgnored(item.path)) return;

      if (!currentPath) {
        // Root directory level
        const parts = item.path.split('/');
        if (parts.length > 1 || item.type === 'tree') {
          subfolders.add(parts[0]);
        } else if (parts.length === 1 && item.type === 'blob') {
          directFiles.push({ name: item.path, path: item.path });
        }
      } else {
        // Inner directory level
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

    // Render Breadcrumb Navigation
    html += renderBreadcrumbs(currentPath);

    // Title Header
    if (currentPath) {
      const folderName = currentPath.split('/').pop();
      html += `<h2 style="border-bottom: 2px solid #eaecef; padding-bottom: 8px; margin-bottom: 20px;">📁 ${folderName}</h2>`;
    } else {
      html += `<h2 style="border-bottom: 2px solid #eaecef; padding-bottom: 8px; margin-bottom: 20px;">Course Modules</h2>`;
    }

    // Subfolders Section
    if (subfolders.size > 0) {
      html += `<div style="display: grid; grid-template-columns: repeat(auto-fill, minmax(200px, 1fr)); gap: 14px; margin-bottom: 24px;">`;
      Array.from(subfolders).sort().forEach(folder => {
        const nextPath = currentPath ? `${currentPath}/${folder}` : folder;
        html += `
          <a href="?path=${encodeURIComponent(nextPath)}" style="display: flex; align-items: center; padding: 14px 18px; border: 1px solid #e1e4e8; border-radius: 8px; text-decoration: none; color: #0366d6; background: #f6f8fa; font-weight: 600; transition: background 0.2s ease;">
            <span style="font-size: 1.2rem; margin-right: 8px;">📁</span> ${folder}
          </a>
        `;
      });
      html += `</div>`;
    }

    // Direct Files Section
    if (directFiles.length > 0) {
      html += `<h3 style="margin-top: 10px; margin-bottom: 12px; font-size: 1.1rem; color: #57606a;">Files</h3>`;
      html += `<ul style="list-style: none; padding-left: 0;">`;
      directFiles.sort((a, b) => a.name.localeCompare(b.name)).forEach(file => {
        html += `
          <li style="padding: 10px 14px; border-bottom: 1px solid #f0f0f0; display: flex; align-items: center;">
            <span style="margin-right: 10px;">📄</span>
            <a href="${encodeURI(file.path)}" target="_blank" style="color: #0366d6; text-decoration: none; font-weight: 500;">${file.name}</a>
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

  init();
</script>