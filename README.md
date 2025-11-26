# NYXO – Faux système d’exploitation en une page HTML

`nyxo.html` est un mini “système d’exploitation” entièrement écrit en **HTML/CSS/JavaScript**, sans backend, qui tourne directement dans ton navigateur.

Il simule :

- un **bureau** avec icônes,
- une **barre des tâches**,
- un **menu démarrer**,
- des **fenêtres** déplaçables (sur desktop) et plein écran (sur mobile),
- plusieurs **applications** internes (bloc-notes, explorateur de fichiers, visionneuse, console, mode Kali, etc.).

Tout fonctionne **localement** : rien n’est envoyé à un serveur, tout reste dans ton onglet.

---

## 1. Objectifs du projet

NYXO sert à :

1. **Jouer** avec l’idée d’un OS dans le navigateur.
2. **Prototyper** des modules pédagogiques (cybersécurité, sensibilisation aux données, etc.).
3. **Servir de base** pour rajouter facilement d’autres applis (lecteurs, quiz, mini-jeux, simulateurs…).

Tu peux :

- l’ouvrir directement dans ton navigateur (`Ctrl+O` > choisir `nyxo.html`),  
- l’héberger sur un serveur statique (GitHub Pages, Vercel, etc.),  
- le modifier comme un simple fichier HTML.

---

## 2. Structure générale de `nyxo.html`

Le fichier est organisé en trois grandes parties :

1. `<head>`  
   - Définition des **variables CSS** (`--accent`, `--bg-desktop`, etc.).
   - Styles pour :
     - le bureau,
     - la barre des tâches,
     - les fenêtres,
     - les applications (notes, explorateur, Oxyn, Mode Kali, Alerte Cyber…).

2. `<body>`  
   - `<div id="desktop">` : le “bureau”.
   - `<div id="startMenu">` : menu démarrer.
   - `<div id="taskbar">` : barre des tâches (bouton Menu, liste de fenêtres, horloge).
   - `<template id="windowTemplate">` : modèle de fenêtre (titlebar + contenu).

3. `<script>`  
   - **État global** (liste des fenêtres ouvertes, registre de fichiers, etc.).
   - Tableau `APPS` qui décrit toutes les applications.
   - Gestion de :
     - la création/fermeture/minimisation des fenêtres,
     - la barre des tâches,
     - le menu démarrer,
     - les applis (une fonction par application).

---

## 3. Les applications incluses

### 3.1. À propos (`id: "about"`)

- Fonction : **présenter** NYXO, ses limites, son fonctionnement.
- Implémentation : `createAboutApp(container)` remplit la fenêtre avec du texte explicatif.

### 3.2. Bloc-notes (`id: "notes"`)

- Fonction : prendre des notes **persistantes** dans `localStorage`.
- Clés :
  - Constante `NOTES_KEY = "nyxo_notes"`.
  - `createNotesApp(container)` :
    - crée un `<textarea>`,
    - charge le contenu depuis `localStorage`,
    - sauvegarde sur clic sur “💾 Sauvegarder” ou `Ctrl+S`.

### 3.3. Explorateur (`id: "explorer"`)

- Fonction : **scanner un dossier local** via `<input type="file" webkitdirectory>`.
- Affiche :
  - chemin relatif,
  - taille,
  - icône par type (image, audio, texte, etc.).
- Chaque fichier est enregistré dans le **registre** `FILE_REGISTRY` avec un ID unique via `registerFile(file)`.
- Un clic sur une entrée appelle `openFileInViewer(fileId, path)`.

### 3.4. Oxyn – Visionneuse & éditeur (`id: "viewer"`)

- Fonction :
  - afficher des fichiers ouverts depuis l’Explorateur,
  - **importer** directement un fichier (bouton `📥 Importer un fichier`),
  - **scanner** le fichier (taille, type, stats texte),
  - **éditer** les fichiers texte (`txt`, `md`, `json`, `html`, etc.),
  - **exporter** une copie (texte ou binaire).

- État :
  - `OXYN_STATE` contient :
    - `file` : le `File` courant,
    - `path` : chemin ou nom,
    - `text` : contenu texte si applicable,
    - `isText` : booléen,
    - `editMode` : lecture / édition.

- Flux :
  1. `openFileInViewer(fileId, path)` récupère le `File` dans `FILE_REGISTRY` et ouvre la fenêtre Oxyn.
  2. `renderFileInViewer(container, file, path)` :
     - détecte le type (image/audio/vidéo/PDF/texte/autre),
     - rend le contenu dans Oxyn,
     - active/désactive les boutons Scan / Éditer / Exporter.

### 3.5. Mode Kali (`id: "kali"`)

- Fonction : générer un **rapport technique** de la session
  (ce que le navigateur expose : user-agent, langue, résolution, stockage estimé, etc.).
- Fonctions clés :
  - `generateSessionReportBase()` : récupère les infos via `navigator`, `screen`, etc.
  - `enrichReportWithStorage(report, cb)` : ajoute l’estimation de stockage si possible.
  - `formatReportHuman/JSON/MD/TXT/CSV/HTML` : formats d’export.
  - `triggerDownload(filename, mime, content)` : télécharge le rapport dans différents formats.
- UI :
  - Scan manuel,
  - auto-scan à intervalle régulier (minutes),
  - boutons d’export (JSON / TXT / MD / CSV / HTML).

### 3.6. Alerte Cyber (`id: "cyber"`)

- Fonction : module de **sensibilisation par la peur** (mais pédagogique).
- Utilise les mêmes infos de base que le Mode Kali pour générer un texte
  expliquant ce qu’un site peut déduire de ta config.
- Inclut :
  - cartes “risques” (suivi publicitaire, empreinte numérique, phishing, etc.),
  - mini quiz avec boutons et feedback.

### 3.7. Paramètres (`id: "settings"`)

- Fonction : changer le **fond d’écran** via des “swatches”.
- Données :
  - tableau `wallpapers` (id, label, css).
  - `applyWallpaper(id)` applique le CSS sur `--bg-desktop` + sauvegarde dans `localStorage`.

### 3.8. Console (`id: "terminal"`)

- Fonction : mini **shell jouet**.
- Commandes :
  - `help`, `whoami`, `time`, `clear`.
- Permet d’illustrer comment gérer **une petite logique interne** dans un module.

---

## 4. Comment modifier NYXO

### 4.1. Modifier les couleurs et le thème

Dans le `<style>` :

- Variables principales :

```css
:root {
  --accent: #4caf50;
  --accent-dark: #2e7d32;
  --bg-desktop: radial-gradient(circle at top left, #202840, #050813);
  --text: #f5f5f5;
  --window-bg: #121826;
  --taskbar-bg: rgba(8, 10, 20, 0.95);
}
````

Tu peux changer :

* `--accent` / `--accent-dark` : couleurs des boutons principaux.
* `--bg-desktop` : fond du bureau (ou via les Paramètres).
* `--window-bg` : fond des fenêtres.

### 4.2. Changer le fond d’écran par défaut

Dans le script :

* `wallpapers` : tableau des thèmes.
* `applyWallpaper(id)` : applique + sauvegarde.
* Au démarrage :

```js
const savedWallpaper = localStorage.getItem(WALLPAPER_KEY);
if (savedWallpaper) applyWallpaper(savedWallpaper);
else applyWallpaper("dark-neon");
```

Change `"dark-neon"` par un autre ID de `wallpapers` pour changer le thème par défaut.

### 4.3. Ajouter ou supprimer des applis du bureau

Le tableau `APPS` définit toutes les applis :

```js
const APPS = [
  { id: "about",    name: "À propos",    icon: "💻", open: createAboutApp },
  { id: "notes",    name: "Bloc-notes", icon: "📝", open: createNotesApp },
  ...
];
```

* Pour **retirer** une appli : supprime l’objet correspondant.
* Pour **ajouter** une appli :

  1. Crée une fonction `createNomApp(container)`.
  2. Ajoute une entrée dans `APPS` avec :

     * `id`: identifiant unique (string),
     * `name`: nom affiché,
     * `icon`: emoji affiché,
     * `open`: référence à la fonction.

Le bureau et le menu démarrer sont alimentés automatiquement par `APPS`.

---

## 5. Comment ajouter un nouveau module (pattern général)

### Étape 1 – Créer la fonction d’appli

Dans le `<script>`, ajoute :

```js
function createMaNouvelleApp(container) {
  container.innerHTML = `
    <p><strong>Ma nouvelle appli</strong></p>
    <p class="helper-text">Ici je fais ce que je veux…</p>
  `;
}
```

### Étape 2 – Ajouter l’entrée dans `APPS`

Dans le tableau `APPS`, ajoute par exemple :

```js
{
  id: "nouvelle",
  name: "Nouvelle app",
  icon: "✨",
  open: createMaNouvelleApp
},
```

> Important : ne pas dupliquer les `id` existants.

C’est tout :

* l’appli apparaîtra sur le bureau (icône),
* dans le menu démarrer,
* et aura sa fenêtre gérée automatiquement.

---

## 6. Tutoriel : ajouter un module “Lecteur Markdown”

### 6.1. Objectif du module Markdown

Créer une appli :

* qui permet de **coller du texte Markdown** ou **d’importer un fichier `.md`**,
* de voir une **prévisualisation HTML** dans la fenêtre,
* de **exporter** le texte markdown.

Ce module sert d’exemple pour :

* une appli avec **éditeur + preview**,
* un **import/export** de fichier,
* une mini “logique métier” (rendu Markdown).

---

### 6.2. Étape 1 – Ajouter un peu de CSS

Dans le `<style>`, ajoute (par exemple sous la section OXYN) :

```css
/* APPLI : Lecteur Markdown */

.md-layout {
  margin-top: 0.4rem;
  display: grid;
  grid-template-columns: 1fr;
  gap: 0.4rem;
}

@media (min-width: 800px) {
  .md-layout {
    grid-template-columns: 1fr 1fr;
  }
}

.md-editor,
.md-preview {
  border-radius: 0.75rem;
  border: 1px solid rgba(255,255,255,0.2);
  background: rgba(0,0,0,0.6);
  padding: 0.4rem;
  min-height: 8rem;
  max-height: 16rem;
  overflow: auto;
}

.md-editor textarea {
  width: 100%;
  height: 100%;
  min-height: 8rem;
  border: none;
  background: transparent;
  color: #f5f5f5;
  font-family: ui-monospace, SFMono-Regular, Menlo, Monaco, Consolas, "Liberation Mono", "Courier New", monospace;
  font-size: 0.8rem;
  resize: none;
}

.md-editor textarea:focus {
  outline: none;
}

.md-preview-content {
  font-size: 0.8rem;
  line-height: 1.4;
}

.md-preview-content h1,
.md-preview-content h2,
.md-preview-content h3 {
  margin: 0.2rem 0;
}

.md-preview-content p {
  margin: 0.2rem 0;
}

.md-preview-content code {
  font-family: ui-monospace, SFMono-Regular, Menlo, Monaco, Consolas, "Liberation Mono", "Courier New", monospace;
  font-size: 0.78rem;
  background: rgba(255,255,255,0.08);
  padding: 0.05rem 0.25rem;
  border-radius: 0.3rem;
}

.md-preview-content pre {
  background: rgba(0,0,0,0.8);
  padding: 0.3rem;
  border-radius: 0.4rem;
  overflow: auto;
}
```

---

### 6.3. Étape 2 – Ajouter l’entrée dans `APPS`

Dans le tableau `APPS`, ajoute une nouvelle entrée (par exemple juste après Oxyn) :

```js
{
  id: "markdown",
  name: "Markdown",
  icon: "📚",
  open: createMarkdownApp
},
```

> Attention : veille à ce que la fonction `createMarkdownApp` existe (prochaine étape).

---

### 6.4. Étape 3 – Fonction de rendu Markdown simplifiée

Toujours dans le `<script>`, ajoute quelque part (par exemple près des autres utilitaires) cette petite fonction de rendu **très simple** :

```js
// Rendu Markdown ultra simplifié (pour l'exemple)
function simpleMarkdownToHtml(md) {
  if (!md) return "";

  let html = md;

  // Échappement très léger de base (pour éviter d'injecter du HTML brut)
  html = html
    .replace(/&/g, "&amp;")
    .replace(/</g, "&lt;")
    .replace(/>/g, "&gt;");

  // Titres
  html = html.replace(/^### (.*)$/gm, "<h3>$1</h3>");
  html = html.replace(/^## (.*)$/gm, "<h2>$1</h2>");
  html = html.replace(/^# (.*)$/gm, "<h1>$1</h1>");

  // Gras & italique
  html = html.replace(/\*\*(.+?)\*\*/g, "<strong>$1</strong>");
  html = html.replace(/\*(.+?)\*/g, "<em>$1</em>");

  // Code inline
  html = html.replace(/`([^`]+)`/g, "<code>$1</code>");

  // Liens [texte](url)
  html = html.replace(/\[([^\]]+)\]\(([^)]+)\)/g, '<a href="$2" target="_blank" rel="noreferrer">$1</a>');

  // Paragraphes (simple) : on découpe par double saut de ligne
  const blocks = html.split(/\n\s*\n/);
  html = blocks.map(b => `<p>${b.replace(/\n/g, "<br>")}</p>`).join("");

  return html;
}
```

> C’est volontairement minimal (et sûr) : pas de parsing complet, juste de quoi faire un rendu agréable pour la démo.

---

### 6.5. Étape 4 – Créer l’appli Markdown

Ajoute ensuite la fonction `createMarkdownApp(container)` :

```js
function createMarkdownApp(container) {
  container.innerHTML = `
    <p><strong>Lecteur Markdown NYXO</strong></p>
    <p class="helper-text">
      Colle ton texte Markdown, importe un fichier <code>.md</code> ou exporte ton contenu.
    </p>

    <div class="oxyn-toolbar" style="margin-top:0.3rem;">
      <button class="btn-ghost" id="mdImportBtn">📥 Importer .md</button>
      <button class="btn-ghost" id="mdExportBtn">💾 Exporter .md</button>
      <input type="file" id="mdFileInput" accept=".md,text/markdown" style="display:none;">
    </div>

    <div class="md-layout" style="margin-top:0.4rem;">
      <div class="md-editor">
        <textarea id="mdEditor" placeholder="# Titre

Un peu de **Markdown** ici.
- Liste
- De
- Test

[Un lien](https://example.org)
"></textarea>
      </div>
      <div class="md-preview">
        <div class="md-preview-content" id="mdPreview"></div>
      </div>
    </div>
  `;

  const editor = container.querySelector("#mdEditor");
  const preview = container.querySelector("#mdPreview");
  const importBtn = container.querySelector("#mdImportBtn");
  const exportBtn = container.querySelector("#mdExportBtn");
  const fileInput = container.querySelector("#mdFileInput");

  function refreshPreview() {
    const md = editor.value;
    const html = simpleMarkdownToHtml(md);
    preview.innerHTML = html;
  }

  // Mise à jour en temps réel
  editor.addEventListener("input", refreshPreview);

  // Import de fichier .md
  importBtn.addEventListener("click", () => {
    fileInput.click();
  });

  fileInput.addEventListener("change", () => {
    const file = fileInput.files?.[0];
    if (!file) return;
    const reader = new FileReader();
    reader.onload = () => {
      editor.value = reader.result || "";
      refreshPreview();
    };
    reader.onerror = () => {
      alert("Erreur de lecture du fichier Markdown.");
    };
    reader.readAsText(file, "utf-8");
  });

  // Export du contenu
  exportBtn.addEventListener("click", () => {
    const md = editor.value || "";
    const defName = "nyxo-notes.md";
    const filename = prompt("Nom du fichier à exporter :", defName) || defName;
    triggerDownload(filename, "text/markdown", md);
  });

  // Première preview
  refreshPreview();
}
```

---

### 6.6. Résultat

Après avoir :

1. Ajouté le CSS Markdown,
2. Ajouté l’entrée dans `APPS`,
3. Ajouté `simpleMarkdownToHtml`,
4. Ajouté `createMarkdownApp`,

tu obtiens :

* une **nouvelle icône** sur le bureau : `📚 Markdown`,
* un **nouvel item** dans le menu démarrer,
* une appli avec :

  * zone d’édition Markdown (gauche),
  * prévisualisation HTML (droite sur écrans larges, en-dessous sur mobile),
  * import `.md`,
  * export `.md`.

C’est un exemple concret de **nouveau module** pour NYXO que tu peux ensuite cloner/adapter pour :

* un lecteur JSON graphique,
* un éditeur de cartes de jeu,
* une interface pour questionnaires,
* etc.

---

## 7. Idées pour aller plus loin

* Ajouter un module “Journal” connecté au bloc-notes et au Mode Kali (journal technique).
* Intégrer une mini API “plugins” (format d’appli standardisé).
* Ajouter un module “Tableau de bord” qui réunit :

  * notes,
  * derniers rapports Kali,
  * dernières sessions d’Alerte Cyber,
  * fichiers récemment ouverts dans Oxyn.

NYXO est pensé comme une **boîte à outils** : un seul fichier, facile à trimballer (clé USB, intranet, atelier), que tu peux enrichir petit à petit.

---

