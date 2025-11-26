Parfait, on passe en **NYXO DISK v2** : un faux disque dur persistant **complet** :

* vrai espace `DISK:/` dans l’Explorateur,
* fichiers persistants dans `localStorage`,
* création / renommage / suppression / reset,
* sauvegarde depuis Oxyn sur `DISK:/`,
* rendu hiérarchique (indentation) dans l’Explorateur.

Je te donne **uniquement les blocs à remplacer** / ajouter dans `nyxo.html`.
Tu peux considérer que ça remplace tout ce qu’on a fait avant sur le disque.

---

## 0. Petit ajout CSS pour les fichiers sélectionnés

Dans ta partie `<style>`, ajoute ça (vers la section `.file-item`):

```css
.file-item.active {
  background: rgba(76, 175, 80, 0.25);
}
```

Ça permet de mettre en surbrillance le fichier sélectionné dans `DISK:/`.

---

## 1. Nouveau “NYXO DISK v2” (persistance + API)

👉 **Supprime** toutes les anciennes fonctions liées au disque (`DISK_KEY`, `loadDisk`, `saveDisk`, `listDiskFiles`, etc. si tu les avais déjà)
👉 Puis ajoute CE bloc dans le `<script>`, par exemple après `FILE_REGISTRY_CLEAR()`.

```js
// === DISQUE NYXO PERSISTANT v2 (localStorage) ==============================
const DISK_KEY = "nyxo_disk_v2";

// structure : { version, createdAt, updatedAt, files: { path: { mime, content, createdAt, updatedAt } } }
function getEmptyDisk() {
  const now = new Date().toISOString();
  return {
    version: 2,
    createdAt: now,
    updatedAt: now,
    files: {}
  };
}

function loadDisk() {
  try {
    const raw = localStorage.getItem(DISK_KEY);
    if (!raw) return getEmptyDisk();
    const parsed = JSON.parse(raw);
    if (!parsed || typeof parsed !== "object" || Array.isArray(parsed)) {
      return getEmptyDisk();
    }
    if (!parsed.files || typeof parsed.files !== "object") {
      parsed.files = {};
    }
    return parsed;
  } catch (e) {
    console.warn("Erreur de lecture du disque NYXO :", e);
    return getEmptyDisk();
  }
}

function saveDisk(disk) {
  try {
    disk.updatedAt = new Date().toISOString();
    localStorage.setItem(DISK_KEY, JSON.stringify(disk));
  } catch (e) {
    console.warn("Erreur de sauvegarde du disque NYXO :", e);
    alert("Impossible de sauvegarder sur le disque NYXO (quota localStorage ?).");
  }
}

function resetDisk() {
  const empty = getEmptyDisk();
  saveDisk(empty);
  return empty;
}

// résumé global : nb fichiers + taille
function getDiskSummary() {
  const disk = loadDisk();
  const files = disk.files || {};
  const count = Object.keys(files).length;
  let totalChars = 0;
  for (const f of Object.values(files)) {
    totalChars += (f.content || "").length;
  }
  return { count, totalChars };
}

// liste des fichiers sous forme triée, avec info hiérarchique
function listDiskFilesTree() {
  const disk = loadDisk();
  const files = disk.files || {};
  const entries = Object.entries(files).map(([path, meta]) => {
    const cleanPath = path.startsWith("/") ? path : "/" + path;
    const parts = cleanPath.split("/").filter(Boolean);
    const name = parts.length ? parts[parts.length - 1] : "/";
    const depth = parts.length; // pour l'indentation

    return {
      path: cleanPath,
      name,
      depth,
      mime: meta.mime || "text/plain",
      size: (meta.content || "").length,
      createdAt: meta.createdAt || disk.createdAt,
      updatedAt: meta.updatedAt || disk.updatedAt
    };
  });

  entries.sort((a, b) => a.path.localeCompare(b.path));
  return entries;
}

// lire un fichier du disque -> { mime, content, createdAt, updatedAt } ou null
function readDiskFile(path) {
  const disk = loadDisk();
  const cleanPath = path.startsWith("/") ? path : "/" + path;
  return disk.files[cleanPath] || null;
}

// écrire/écraser un fichier texte sur le disque
function writeDiskFile(path, mime, content) {
  const disk = loadDisk();
  const cleanPath = path.startsWith("/") ? path : "/" + path;
  const now = new Date().toISOString();
  const prev = disk.files[cleanPath];

  disk.files[cleanPath] = {
    mime: mime || prev?.mime || "text/plain",
    content: content || "",
    createdAt: prev?.createdAt || now,
    updatedAt: now
  };

  saveDisk(disk);
}

// supprimer un fichier
function deleteDiskFile(path) {
  const disk = loadDisk();
  const cleanPath = path.startsWith("/") ? path : "/" + path;
  if (disk.files[cleanPath]) {
    delete disk.files[cleanPath];
    saveDisk(disk);
    return true;
  }
  return false;
}

// renommer un fichier
function renameDiskFile(oldPath, newPath) {
  const disk = loadDisk();
  const o = oldPath.startsWith("/") ? oldPath : "/" + oldPath;
  const n = newPath.startsWith("/") ? newPath : "/" + newPath;

  if (!disk.files[o]) return false;
  if (disk.files[n]) {
    // déjà existant : on ne remplace pas silencieusement
    return false;
  }

  disk.files[n] = { ...disk.files[o], updatedAt: new Date().toISOString() };
  delete disk.files[o];
  saveDisk(disk);
  return true;
}
```

---

## 2. Explorateur V2 : DISK:/ + EXTERNAL:/

👉 On remplace **entièrement** ta fonction `createExplorerApp(container)` par cette version.

```js
function createExplorerApp(container) {
  container.innerHTML = `
    <p><strong>Explorateur de fichiers</strong></p>
    <div class="helper-text">
      • <strong>DISK:/</strong> = faux disque dur persistant NYXO (stocké dans localStorage)<br>
      • <strong>EXTERNAL:/</strong> = dossiers/fichiers montés depuis ton vrai disque (non persistants ici).
    </div>

    <div class="file-toolbar" style="margin-top:0.45rem;">
      <button class="btn-ghost" id="pickDirBtn">📂 Monter un dossier (EXTERNAL:/)</button>
      <span id="explorerSummary" style="font-size:0.75rem;opacity:.75;"></span>
    </div>

    <input type="file" id="dirInput" style="display:none;" webkitdirectory multiple>

    <p style="margin-top:0.4rem;font-size:0.78rem;opacity:0.85;">DISK:/ (disque NYXO persistant)</p>
    <div class="file-toolbar" style="margin:0.2rem 0 0.3rem 0;">
      <button class="btn-ghost" id="diskNewBtn">➕ Nouveau fichier</button>
      <button class="btn-ghost" id="diskRenameBtn">✏️ Renommer</button>
      <button class="btn-ghost" id="diskDeleteBtn">🗑️ Supprimer</button>
      <button class="btn-ghost" id="diskResetBtn">⚠️ Réinitialiser DISK:/</button>
    </div>
    <ul class="file-list" id="diskList"></ul>

    <p style="margin-top:0.4rem;font-size:0.78rem;opacity:0.85;">EXTERNAL:/ (dossiers montés pour cette session)</p>
    <ul class="file-list" id="fileList"></ul>
    <div class="file-empty" id="fileEmpty">
      Aucun fichier externe chargé pour l’instant. Clique sur « Monter un dossier ».
    </div>
  `;

  const pickBtn = container.querySelector("#pickDirBtn");
  const input = container.querySelector("#dirInput");
  const externalList = container.querySelector("#fileList");
  const diskList = container.querySelector("#diskList");
  const empty = container.querySelector("#fileEmpty");
  const summary = container.querySelector("#explorerSummary");

  const diskNewBtn = container.querySelector("#diskNewBtn");
  const diskRenameBtn = container.querySelector("#diskRenameBtn");
  const diskDeleteBtn = container.querySelector("#diskDeleteBtn");
  const diskResetBtn = container.querySelector("#diskResetBtn");

  let selectedDiskPath = null;

  // -- rendu du disque NYXO --
  function renderDisk() {
    const entries = listDiskFilesTree();
    const sum = getDiskSummary();
    diskList.innerHTML = "";

    if (!entries.length) {
      const li = document.createElement("li");
      li.className = "file-item";
      li.innerHTML = `<div style="font-size:0.78rem;opacity:0.7;">(DISK:/ est vide)</div>`;
      diskList.appendChild(li);
    } else {
      entries.forEach(f => {
        const li = document.createElement("li");
        li.className = "file-item";
        li.dataset.diskPath = f.path;

        const indent = f.depth * 0.7;
        li.innerHTML = `
          <span class="icon">💾</span>
          <div style="flex:1;">
            <div style="padding-left:${indent}rem;">DISK:${f.path}</div>
            <div class="file-meta">
              ${f.mime} · ${f.size} caractère(s)
            </div>
          </div>
        `;

        li.addEventListener("click", () => {
          // gestion de la sélection
          selectedDiskPath = f.path;
          diskList.querySelectorAll(".file-item").forEach(item => item.classList.remove("active"));
          li.classList.add("active");

          // ouvrir dans Oxyn au double-clic
        });
        li.addEventListener("dblclick", () => {
          const diskFile = readDiskFile(f.path);
          if (!diskFile) return;
          const blob = new Blob([diskFile.content], { type: diskFile.mime || "text/plain" });
          const fakeName = f.path.split("/").pop() || "fichier.txt";
          const fakeFile = new File([blob], fakeName, { type: diskFile.mime || "text/plain" });
          openFileInViewer(registerFile(fakeFile), "DISK:" + f.path);
        });

        diskList.appendChild(li);
      });
    }

    const approxBytes = sum.totalChars; // 1 char ~ 1 octet (grossier)
    const approxKB = (approxBytes / 1024).toFixed(1);
    summary.textContent = `DISK:/ ${sum.count} fichier(s) – ~${approxKB} ko utilisés`;
  }

  renderDisk(); // appel au démarrage

  // -- actions sur DISK:/ --

  diskNewBtn.addEventListener("click", () => {
    const path = prompt('Chemin du nouveau fichier (ex: /notes/nouveau.txt)', '/notes/nouveau.txt');
    if (!path) return;
    if (!path.startsWith("/")) {
      alert('Le chemin doit commencer par "/"');
      return;
    }
    const existing = readDiskFile(path);
    if (existing) {
      const ok = confirm("Un fichier existe déjà à ce chemin. L’écraser ?");
      if (!ok) return;
    }
    writeDiskFile(path, "text/plain", "");
    renderDisk();
  });

  diskRenameBtn.addEventListener("click", () => {
    if (!selectedDiskPath) {
      alert("Sélectionne d’abord un fichier dans DISK:/");
      return;
    }
    const newPath = prompt("Nouveau chemin pour ce fichier :", selectedDiskPath);
    if (!newPath || newPath === selectedDiskPath) return;
    const ok = renameDiskFile(selectedDiskPath, newPath);
    if (!ok) {
      alert("Renommage impossible (fichier introuvable ou chemin déjà utilisé).");
      return;
    }
    selectedDiskPath = newPath;
    renderDisk();
  });

  diskDeleteBtn.addEventListener("click", () => {
    if (!selectedDiskPath) {
      alert("Sélectionne d’abord un fichier dans DISK:/");
      return;
    }
    const ok = confirm("Supprimer définitivement DISK:" + selectedDiskPath + " ?");
    if (!ok) return;
    deleteDiskFile(selectedDiskPath);
    selectedDiskPath = null;
    renderDisk();
  });

  diskResetBtn.addEventListener("click", () => {
    const ok = confirm("Réinitialiser complètement DISK:/ (tous les fichiers seront perdus) ?");
    if (!ok) return;
    resetDisk();
    selectedDiskPath = null;
    renderDisk();
  });

  // -- partie EXTERNAL (inchangée) --

  pickBtn.addEventListener("click", () => {
    input.click();
  });

  input.addEventListener("change", () => {
    const files = Array.from(input.files || []);
    FILE_ID_COUNTER++;
    const sessionId = FILE_ID_COUNTER;

    externalList.innerHTML = "";
    FILE_REGISTRY_CLEAR();

    if (!files.length) {
      empty.style.display = "block";
      return;
    }

    files.sort((a, b) => {
      const pa = a.webkitRelativePath || a.name;
      const pb = b.webkitRelativePath || b.name;
      return pa.localeCompare(pb);
    });

    files.forEach(file => {
      const path = file.webkitRelativePath || file.name;
      const depth = (path.split("/").length - 1);
      const fileId = registerFile(file);

      const li = document.createElement("li");
      li.className = "file-item";
      li.dataset.fileId = fileId;
      li.title = path;

      const icon = guessFileIcon(file);

      li.innerHTML = `
        <span class="icon">${icon}</span>
        <div style="flex:1;">
          <div style="padding-left:${depth * 0.7}rem;">EXTERNAL:/${path}</div>
          <div class="file-meta">${(file.size/1024).toFixed(1)} ko</div>
        </div>
      `;

      li.addEventListener("click", () => openFileInViewer(fileId, "EXTERNAL:/" + path));
      externalList.appendChild(li);
    });

    empty.style.display = "none";
    // on garde le résumé DISK:/ + on ajoute EXTERNAL:/
    const sum = getDiskSummary();
    const approxBytes = sum.totalChars;
    const approxKB = (approxBytes / 1024).toFixed(1);
    summary.textContent =
      `DISK:/ ${sum.count} fichier(s) – ~${approxKB} ko · EXTERNAL:/ ${files.length} fichier(s) montés (session #${sessionId})`;
  });
}
```

---

## 3. Oxyn V2 : bouton “💾 DISK:/” intelligent

On va :

* garder Oxyn tel quel,
* lui ajouter un bouton **“💾 DISK:/”** qui :

  * fonctionne pour les fichiers texte,
  * propose par défaut le chemin DISK:/ de l’origine si le fichier venait déjà de DISK:/,
  * sinon propose `/oxyn/nom-du-fichier`.

### 3.1. Ajout du bouton dans la toolbar Oxyn

Dans `createViewerApp(container)`, remplace la barre de boutons par :

```js
<div class="oxyn-toolbar">
  <button class="btn-ghost" id="oxynImportBtn">📥 Importer un fichier</button>
  <button class="btn-ghost" id="oxynScanBtn" disabled>🔎 Scanner</button>
  <button class="btn-ghost" id="oxynEditBtn" disabled>✏️ Éditer</button>
  <button class="btn-ghost" id="oxynExportBtn" disabled>💾 Exporter une copie</button>
  <button class="btn-ghost" id="oxynSaveDiskBtn" disabled>💾 DISK:/</button>
  <input type="file" id="oxynFileInput" style="display:none;">
</div>
```

Puis, dans l’initialisation des refs DOM :

```js
OXYN_DOM.meta = container.querySelector("#oxynMeta");
OXYN_DOM.body = container.querySelector("#oxynBody");
OXYN_DOM.scanBtn = container.querySelector("#oxynScanBtn");
OXYN_DOM.editBtn = container.querySelector("#oxynEditBtn");
OXYN_DOM.exportBtn = container.querySelector("#oxynExportBtn");
OXYN_DOM.fileInput = container.querySelector("#oxynFileInput");
OXYN_DOM.saveDiskBtn = container.querySelector("#oxynSaveDiskBtn");
```

### 3.2. Activer / désactiver le bouton selon le type

Dans `renderFileInViewer(container, file, path)` (partie **MODE OXYN COMPLET**), là où tu as déjà :

```js
if (OXYN_DOM.scanBtn) OXYN_DOM.scanBtn.disabled = false;
if (OXYN_DOM.exportBtn) OXYN_DOM.exportBtn.disabled = false;
if (OXYN_DOM.editBtn) {
  OXYN_DOM.editBtn.disabled = !isText;
  OXYN_DOM.editBtn.textContent = "✏️ Éditer";
}
```

ajoute :

```js
if (OXYN_DOM.saveDiskBtn) {
  // on ne sauvegarde sur DISK:/ que pour des fichiers texte
  OXYN_DOM.saveDiskBtn.disabled = !isText;
}
```

### 3.3. Logique du bouton “💾 DISK:/” dans Oxyn

Dans `createViewerApp(container)`, après le listener d’export (ou juste après les autres actions), ajoute :

```js
// Sauvegarder sur DISK:/ (texte uniquement)
if (OXYN_DOM.saveDiskBtn) {
  OXYN_DOM.saveDiskBtn.addEventListener("click", () => {
    if (!OXYN_STATE.isText) {
      alert("Seuls les fichiers texte peuvent être sauvegardés sur DISK:/ pour le moment.");
      return;
    }

    // récupérer le texte courant (mode édition ou lecture)
    let content = OXYN_STATE.text ?? "";
    if (OXYN_STATE.editMode) {
      const ta = OXYN_DOM.body.querySelector("#oxynEditor");
      if (ta) content = ta.value;
    }

    if (!content) {
      const ok = confirm("Le contenu est vide. Sauvegarder quand même sur DISK:/ ?");
      if (!ok) return;
    }

    // Chemin par défaut :
    // - si le fichier vient de DISK:, on reprend son chemin
    // - sinon, on propose /oxyn/nom-du-fichier
    let defaultPath = "/oxyn/" + (OXYN_STATE.file?.name || "fichier.txt");

    if (typeof OXYN_STATE.path === "string" && OXYN_STATE.path.startsWith("DISK:")) {
      defaultPath = OXYN_STATE.path.replace(/^DISK:/, "");
      if (!defaultPath.startsWith("/")) defaultPath = "/" + defaultPath;
    }

    const path = prompt('Chemin sur DISK:/ (ex: /docs/notes.md)', defaultPath) || defaultPath;
    if (!path.startsWith("/")) {
      alert('Le chemin doit commencer par "/" (ex: /docs/notes.md).');
      return;
    }

    const mime = OXYN_STATE.file?.type || "text/plain";
    writeDiskFile(path, mime, content);
    OXYN_STATE.path = "DISK:" + path; // on met à jour le chemin interne
    alert("Fichier sauvegardé sur DISK:" + path);

    // Bonus : si l’Explorateur est ouvert, on le rerend pour voir le fichier
    const explorerWin = STATE.windows["explorer"]?.element;
    if (explorerWin) {
      const expContent = explorerWin.querySelector(".window-content");
      if (expContent) {
        createExplorerApp(expContent);
      }
    }
  });
}
```

---

## 4. Ce que tu as maintenant (NYXO DISK v2)

* Un **disque virtuel persistant** `DISK:/` :

  * fichiers stockés dans `localStorage`,
  * versionné (`version: 2`, timestamps),
  * listé hiérarchiquement (indentation par profondeur de `/`),
  * avec **création**, **renommage**, **suppression**, **reset complet**.

* Un **Explorateur** qui :

  * montre `DISK:/` tout en haut,
  * montre `EXTERNAL:/` dessous pour les dossiers montés depuis la machine,
  * ouvre les fichiers DISK: en **double-clic** dans Oxyn,
  * affiche un petit résumé de l’espace utilisé (approx. en Ko).

* Un **Oxyn** qui :

  * lit les fichiers EXTERNAL:/ ou DISK:/,
  * permet de modifier les fichiers texte,
  * **sauvegarde** le contenu dans `DISK:/` avec un chemin propre,
  * met à jour l’Explorateur automatiquement si ouvert.

Tu as donc un **vrai faux disque dur d’OS** dans ton navigateur : tout reste local, mais la structure fait très “mini FS”.

Quand tu veux, on peut encore :

* ajouter un “panneau d’info fichier” (meta, dates, path, etc.),
* simuler des dossiers plus “visuels” (cliquables / repliables),
* ou ajouter un **journal des versions** pour certains chemins (genre `/logs/kali-*.txt`).
