Très bonne question, là on touche au cœur du délire “vrai faux OS” 😈

On résume l’idée :

> Donner à NYXO **un faux disque dur persistant**, c’est-à-dire un “DISK:/” qui survit quand tu fermes l’onglet, en utilisant **localStorage** (ou IndexedDB, mais on va faire simple).

Je te propose :

* une **version 1 simple** :

  * disque NYXO = uniquement des **fichiers texte** (notes, markdown, JSON, etc.)
  * stockés dans `localStorage` sous forme de JSON
  * accessible dans l’Explorateur
  * sauvegarde depuis Oxyn via un bouton “💾 Sauver sur DISK:/…”
* tout en gardant **TES fichiers locaux** (via `webkitdirectory`) comme “disque externe non persistant”.

Je te donne les *morceaux de code à ajouter/modifier* par blocs, sans toucher au reste.

---

## 1. Modèle du “disque NYXO”

On va utiliser une structure ultra simple :

```js
// Exemple de DISK interne
{
  "/notes.md": { mime: "text/markdown", content: "# Hello" },
  "/docs/rapport.txt": { mime: "text/plain", content: "Bla bla bla" }
}
```

Tout est stocké sous la clé `nyxo_disk_v1` dans `localStorage`.

### 1.a. Ajouter les constantes + helpers “disque”

Dans ton `<script>`, quelque part en haut avec les autres constantes, ajoute :

```js
const DISK_KEY = "nyxo_disk_v1";   // clé localStorage pour le faux disque
```

Puis un peu plus bas (par exemple après `FILE_REGISTRY_CLEAR`), ajoute les fonctions suivantes :

```js
// === DISQUE NYXO PERSISTANT (localStorage) ================================
function loadDisk() {
  try {
    const raw = localStorage.getItem(DISK_KEY);
    if (!raw) return {};
    const parsed = JSON.parse(raw);
    // sécurité minimale : s'assurer que c'est bien un objet
    if (parsed && typeof parsed === "object" && !Array.isArray(parsed)) {
      return parsed;
    }
    return {};
  } catch (e) {
    console.warn("Erreur de lecture du disque NYXO :", e);
    return {};
  }
}

function saveDisk(disk) {
  try {
    localStorage.setItem(DISK_KEY, JSON.stringify(disk));
  } catch (e) {
    console.warn("Erreur de sauvegarde du disque NYXO :", e);
    alert("Impossible de sauvegarder sur le disque NYXO (quota localStorage ?)");
  }
}

// liste des fichiers du disque sous forme [{ path, mime, size }]
function listDiskFiles() {
  const disk = loadDisk();
  return Object.entries(disk).map(([path, file]) => ({
    path,
    mime: file.mime || "text/plain",
    size: (file.content || "").length
  }));
}

// lire un fichier du disque -> objet { mime, content } ou null
function readDiskFile(path) {
  const disk = loadDisk();
  return disk[path] || null;
}

// écrire/écraser un fichier texte sur le disque
function writeDiskFile(path, mime, content) {
  const disk = loadDisk();
  disk[path] = { mime: mime || "text/plain", content: content || "" };
  saveDisk(disk);
}
```

Avec ça, tu as déjà :

* `writeDiskFile("/notes.md", "text/markdown", "# Salut")`
* `readDiskFile("/notes.md")` → `{ mime, content }`

---

## 2. Intégrer le disque NYXO dans l’Explorateur

On va faire apparaître un **bloc “DISK:/”** en haut de l’Explorateur, avec la liste des fichiers internes.

### 2.a. Adapter le HTML de l’Explorateur

Dans `createExplorerApp(container)`, remplace le `innerHTML` actuel par quelque chose comme ceci (j’ajoute la section DISK, je garde le reste) :

```js
function createExplorerApp(container) {
  container.innerHTML = `
    <p><strong>Explorateur de fichiers</strong></p>
    <div class="helper-text">
      • <strong>DISK:/</strong> = faux disque dur persistant de NYXO (localStorage)<br>
      • <strong>EXTERNAL:/</strong> = dossiers/fichiers de ton vrai disque (non persistants ici).
    </div>

    <div class="file-toolbar">
      <button class="btn-ghost" id="pickDirBtn">📂 Monter un dossier (EXTERNAL:/)</button>
      <span id="explorerSummary" style="font-size:0.75rem;opacity:.75;"></span>
    </div>

    <input type="file" id="dirInput" style="display:none;" webkitdirectory multiple>

    <p style="margin-top:0.4rem;font-size:0.78rem;opacity:0.85;">DISK:/ (disque NYXO persistant)</p>
    <ul class="file-list" id="diskList"></ul>

    <p style="margin-top:0.4rem;font-size:0.78rem;opacity:0.85;">EXTERNAL:/ (dossiers montés pour cette session)</p>
    <ul class="file-list" id="fileList"></ul>
    <div class="file-empty" id="fileEmpty">
      Aucun fichier externe chargé pour l’instant. Clique sur « Monter un dossier ».
    </div>
  `;

  const pickBtn = container.querySelector("#pickDirBtn");
  const input = container.querySelector("#dirInput");
  const list = container.querySelector("#fileList");
  const diskList = container.querySelector("#diskList");
  const empty = container.querySelector("#fileEmpty");
  const summary = container.querySelector("#explorerSummary");

  // -- rendu du disque NYXO --
  function renderDisk() {
    const files = listDiskFiles();
    diskList.innerHTML = "";
    if (!files.length) {
      const li = document.createElement("li");
      li.className = "file-item";
      li.innerHTML = `<div style="font-size:0.78rem;opacity:0.7;">(disque vide)</div>`;
      diskList.appendChild(li);
      return;
    }

    files.forEach(f => {
      const li = document.createElement("li");
      li.className = "file-item";
      li.dataset.diskPath = f.path;
      li.innerHTML = `
        <span class="icon">💾</span>
        <div style="flex:1;">
          <div>${"DISK:" + f.path}</div>
          <div class="file-meta">${f.mime} · ${f.size} caractères</div>
        </div>
      `;
      li.addEventListener("click", () => {
        // on crée un faux File pour l’ouvrir dans Oxyn
        const diskFile = readDiskFile(f.path);
        if (!diskFile) return;
        const blob = new Blob([diskFile.content], { type: diskFile.mime || "text/plain" });
        const fakeFile = new File([blob], f.path.split("/").pop() || "fichier.txt", { type: diskFile.mime });
        openFileInViewer(registerFile(fakeFile), "DISK:" + f.path);
      });
      diskList.appendChild(li);
    });
  }

  renderDisk(); // appel au démarrage

  // -- partie EXTERNAL (inchangée, juste déplacée) --

  pickBtn.addEventListener("click", () => {
    input.click();
  });

  input.addEventListener("change", () => {
    const files = Array.from(input.files || []);
    FILE_ID_COUNTER++;
    const sessionId = FILE_ID_COUNTER;

    list.innerHTML = "";
    FILE_REGISTRY_CLEAR();

    if (!files.length) {
      empty.style.display = "block";
      summary.textContent = "";
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
      list.appendChild(li);
    });

    empty.style.display = "none";
    summary.textContent = `${files.length} fichier(s) montés – session EXTERNAL #${sessionId}`;
  });
}
```

👉 Résultat :

* En haut de l’Explorateur : la **liste persistante DISK:/** (faux disque).
* En dessous : les **fichiers externes** montés pour la session.

---

## 3. Sauvegarder depuis Oxyn sur le disque NYXO

Maintenant on veut :
Dans Oxyn, quand tu as un texte, pouvoir cliquer sur **“Sauver sur DISK:/…”** pour le stocker dans le faux disque.

### 3.a. Ajouter un bouton dans la toolbar Oxyn

Dans `createViewerApp(container)`, dans le `innerHTML` de la toolbar, ajoute un bouton :

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

Et juste après l’initialisation des éléments DOM :

```js
OXYN_DOM.meta = container.querySelector("#oxynMeta");
OXYN_DOM.body = container.querySelector("#oxynBody");
OXYN_DOM.scanBtn = container.querySelector("#oxynScanBtn");
OXYN_DOM.editBtn = container.querySelector("#oxynEditBtn");
OXYN_DOM.exportBtn = container.querySelector("#oxynExportBtn");
OXYN_DOM.fileInput = container.querySelector("#oxynFileInput");
OXYN_DOM.saveDiskBtn = container.querySelector("#oxynSaveDiskBtn");
```

### 3.b. Activer/désactiver le bouton en fonction du type

Dans `renderFileInViewer(container, file, path)` (partie “MODE OXYN COMPLET”), après :

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

### 3.c. Logique du bouton “💾 DISK:/”

Toujours dans `createViewerApp(container)`, après l’event listener d’export (ou juste avant, comme tu veux), ajoute :

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

    // proposition de chemin par défaut
    const baseName = (OXYN_STATE.file?.name || "fichier.txt");
    const defPath = "/oxyn/" + baseName;
    const path = prompt('Chemin sur DISK:/ (ex: /docs/notes.md)', defPath) || defPath;
    if (!path.startsWith("/")) {
      alert('Le chemin doit commencer par "/" (ex: /docs/notes.md).');
      return;
    }

    const mime = OXYN_STATE.file?.type || "text/plain";
    writeDiskFile(path, mime, content);
    alert("Fichier sauvegardé sur DISK:" + path);

    // Bonus : si l’Explorateur est ouvert, on peut forcer son refresh
    const explorerWin = STATE.windows["explorer"]?.element;
    if (explorerWin) {
      const expContent = explorerWin.querySelector(".window-content");
      if (expContent) {
        const diskList = expContent.querySelector("#diskList");
        if (diskList) {
          // petite astuce : on relance juste createExplorerApp sur la même fenêtre
          createExplorerApp(expContent);
        }
      }
    }
  });
}
```

👉 Résultat :

* Tu ouvres n’importe quel fichier texte (ou en import via Oxyn),
* tu modifies si tu veux,
* tu cliques sur **“💾 DISK:/”**,
* tu choisis un chemin `/quelque-chose/fichier.md`,
* c’est **écrit dans `localStorage`** et visible dans `DISK:/` via l’Explorateur.

---

## 4. Ce que tu as maintenant

* **DISK:/** = faux disque dur persistant dans le navigateur (localStorage)

  * géré par `loadDisk/saveDisk/readDiskFile/writeDiskFile`
  * visible dans l’Explorateur en haut
  * ouvert dans Oxyn comme n’importe quel fichier
* **EXTERNAL:/** = dossiers montés via `<input type="file" webkitdirectory>`

  * non persistants, mais utilisables comme “clé USB” le temps de la session
* Oxyn peut :

  * lire EXTERNAL:/,
  * lire DISK:/,
  * **sauvegarder du texte sur DISK:/**.

---

Si tu veux une **V2 plus hardcore** plus tard :

* permettre aussi des fichiers binaires (stockés en base64),
* simuler un arborescence avec véritable notion de dossiers,
* afficher l’espace utilisé (ex “DISK : 120 Ko / 5 Mo”).

Mais avec ce qu’on vient de faire, tu as déjà un **vrai faux disque dur NYXO** qui survit aux fermetures d’onglet 🎛️💾
