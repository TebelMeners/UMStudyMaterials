# UMStudyMaterials
Compiled Lecture Notes and Tutorial Problems from my Undergrad Study
# UM Study Materials

Welcome to the study materials archive. Below is the live structure of all course folders, subfolders, and files.

<div id="explorer" style="margin-top: 20px; font-size: 1.05rem;">Loading repository contents...</div>

<script>
  const OWNER = 'TebelMeners';
  const REPO = 'UMStudyMaterials';

  async function loadTree() {
    try {
      const res = await fetch(`https://api.github.com/repos/${OWNER}/${REPO}/git/trees/main?recursive=1`);
      const data = await res.json();
      const explorer = document.getElementById('explorer');
      explorer.innerHTML = '';
      
      const tree = {};
      data.tree.forEach(item => {
        if (item.path.startsWith('.') || item.path === 'README.md' || item.path === '_config.yml') return;
        const parts = item.path.split('/');
        let curr = tree;
        parts.forEach((part, idx) => {
          if (!curr[part]) {
            curr[part] = (idx === parts.length - 1 && item.type === 'blob') ? { _file: true, path: item.path } : {};
          }
          curr = curr[part];
        });
      });

      explorer.appendChild(renderTree(tree));
    } catch (e) {
      document.getElementById('explorer').innerText = 'Error loading directory contents.';
    }
  }

  function renderTree(node) {
    const ul = document.createElement('ul');
    ul.style.listStyleType = 'none';
    ul.style.paddingLeft = '20px';
    
    for (const key in node) {
      if (key === '_file' || key === 'path') continue;
      const li = document.createElement('li');
      li.style.margin = '6px 0';
      
      if (node[key]._file) {
        li.innerHTML = `📄 <a href="${encodeURI(node[key].path)}" target="_blank">${key}</a>`;
      } else {
        li.innerHTML = `📁 <strong>${key}</strong>`;
        li.appendChild(renderTree(node[key]));
      }
      ul.appendChild(li);
    }
    return ul;
  }

  loadTree();
</script>