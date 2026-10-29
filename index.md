---
layout: default
title: UM Study Materials
---

<div id="app" style="margin-top: 20px;">
  <p>Loading course modules...</p>
</div>

<script>
  const OWNER = 'TebelMeners';
  const REPO = 'UMStudyMaterials';

  async function init() {
    const app = document.getElementById('app');
    const urlParams = new URLSearchParams(window.location.search);
    const selectedFolder = urlParams.get('folder');

    try {
      const res = await fetch(`https://api.github.com/repos/${OWNER}/${REPO}/git/trees/main?recursive=1`);
      const data = await res.json();

      if (!data.tree) {
        app.innerHTML = '<p>Error loading repository data.</p>';
        return;
      }

      if (selectedFolder) {
        renderFolderView(app, data.tree, selectedFolder);
      } else {
        renderTopLevelView(app, data.tree);
      }
    } catch (e) {
      app.innerHTML = '<p>Error loading repository contents.</p>';
    }
  }

  function renderTopLevelView(container, items) {
    const topFolders = new Set();
    items.forEach(item => {
      if (item.path.startsWith('.') || item.path === 'README.md' || item.path === '_config.yml' || item.path === 'index.md') return;
      const parts = item.path.split('/');
      if (parts.length > 1 || item.type === 'tree') {
        topFolders.add(parts[0]);
      }
    });

    if (topFolders.size === 0) {
      container.innerHTML = '<p>No course folders found in repository.</p>';
      return;
    }

    let html = '<h2>Course Modules</h2><div style="display: grid; grid-template-columns: repeat(auto-fill, minmax(220px, 1fr)); gap: 16px; margin-top: 20px;">';
    
    Array.from(topFolders).sort().forEach(folder => {
      html += `
        <a href="?folder=${encodeURIComponent(folder)}" style="display: block; padding: 18px; border: 1px solid #e1e4e8; border-radius: 8px; text-decoration: none; color: #0366d6; background: #f6f8fa; font-weight: bold; text-align: center; transition: all 0.2s ease;">
          📁 ${folder}
        </a>
      `;
    });
    
    html += '</div>';
    container.innerHTML = html;
  }

  function renderFolderView(container, items, folderName) {
    const prefix = folderName + '/';
    const subItems = items.filter(item => item.path === folderName || item.path.startsWith(prefix));

    if (subItems.length === 0) {
      container.innerHTML = `<p><a href="./">← Back to All Courses</a></p><h2>${folderName}</h2><p>Folder is empty.</p>`;
      return;
    }

    const tree = {};
    subItems.forEach(item => {
      if (item.path === folderName) return;
      const relPath = item.path.slice(prefix.length);
      const parts = relPath.split('/');
      let curr = tree;
      parts.forEach((part, idx) => {
        if (!curr[part]) {
          curr[part] = (idx === parts.length - 1 && item.type === 'blob') ? { _file: true, path: item.path } : {};
        }
        curr = curr[part];
      });
    });

    const breadcrumbs = `<div style="margin-bottom: 20px;"><a href="./" style="font-weight: bold; color: #0366d6; text-decoration: none;">← Back to All Courses</a></div>`;
    const title = `<h2 style="border-bottom: 2px solid #eaecef; padding-bottom: 8px;">📁 ${folderName}</h2>`;
    
    container.innerHTML = breadcrumbs + title;
    container.appendChild(buildHtmlTree(tree));
  }

  function buildHtmlTree(node) {
    const ul = document.createElement('ul');
    ul.style.listStyleType = 'none';
    ul.style.paddingLeft = '20px';

    for (const key in node) {
      if (key === '_file' || key === 'path') continue;
      const li = document.createElement('li');
      li.style.margin = '8px 0';

      if (node[key]._file) {
        li.innerHTML = `📄 <a href="${encodeURI(node[key].path)}" target="_blank" style="color: #0366d6;">${key}</a>`;
      } else {
        li.innerHTML = `📁 <strong style="color: #24292e;">${key}</strong>`;
        li.appendChild(buildHtmlTree(node[key]));
      }
      ul.appendChild(li);
    }
    return ul;
  }

  init();
</script>