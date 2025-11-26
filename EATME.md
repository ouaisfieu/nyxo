Ça marche, on le fait “à la main” ici, pas en mode README formel 😊
Je pars de **ton `nyxo.html` actuel**.

---

## Objectif

Ajouter une nouvelle appli **“Navigateur”** dans NYXO :

* Icône 🌐 sur le bureau + menu.
* Barre d’adresse + boutons (Précédent / Suivant / Accueil / Reload).
* Affichage des sites dans un `<iframe>` (avec les limites habituelles : certains sites refusent les iframes).

On fait ça en **4 étapes**.

---

## 1. Ajouter un peu de CSS pour le navigateur

Dans la partie `<style>` de `nyxo.html`, ajoute ce bloc quelque part dans les applis (par exemple juste après Oxyn ou avant “APPLI : MODE KALI”).

```css
/* APPLI : Navigateur Internet */

.browser-toolbar {
  margin-top: 0.4rem;
  display: flex;
  flex-wrap: wrap;
  gap: 0.35rem;
  align-items: center;
}

.browser-toolbar button {
  font-size: 0.78rem;
}

.browser-addr {
  flex: 1;
  border-radius: 999px;
  border: 1px solid rgba(255,255,255,0.2);
  background: rgba(0,0,0,0.6);
  color: #f5f5f5;
  font-size: 0.78rem;
  padding: 0.2rem 0.6rem;
}

.browser-addr::placeholder {
  color: rgba(255,255,255,0.4);
}

.browser-addr:focus {
  outline: none;
  box-shadow: 0 0 0 1px var(--accent);
}

.browser-status {
  margin-top: 0.25rem;
  font-size: 0.72rem;
  opacity: 0.8;
}

.browser-frame-wrap {
  margin-top: 0.4rem;
  border-radius: 0.75rem;
  border: 1px solid rgba(255,255,255,0.2);
  background: #000;
  overflow: hidden;
}

.browser-iframe {
  border: none;
  width: 100%;
  height: 18rem;
}

@media (min-height: 700px) {
  .browser-iframe {
    height: 22rem;
  }
}
```

---

## 2. Ajouter l’appli dans le tableau `APPS`

Dans le `<script>`, repère :

```js
const APPS = [
  {
    id: "about",
    name: "À propos",
    icon: "💻",
    open: createAboutApp
  },
  ...
];
```

Et ajoute une entrée pour le navigateur, par exemple après Oxyn ou la console :

```js
{
  id: "browser",
  name: "Navigateur",
  icon: "🌐",
  open: createBrowserApp
},
```

> Important : l’`id` doit être **unique** (ne pas réutiliser `about`, `notes`, `explorer`, etc.).

À partir de là, NYXO générera automatiquement :

* une icône 🌐 sur le bureau,
* un item dans le menu démarrer.

---

## 3. Petite fonction utilitaire pour nettoyer l’URL

Toujours dans le `<script>`, ajoute cette fonction (par exemple près de `triggerDownload`, ou avec les autres utilitaires globaux) :

```js
function normalizeUrl(input) {
  let url = (input || "").trim();
  if (!url) return "";

  // Bloquer les schémas dangereux
  if (/^javascript:/i.test(url)) {
    alert("Schéma javascript: bloqué pour des raisons de sécurité.");
    return "";
  }

  // Si aucun schéma, on préfixe par https://
  if (!/^https?:\/\//i.test(url)) {
    url = "https://" + url;
  }

  return url;
}
```

But :
– si tu tapes `duckduckgo.com`, ça devient `https://duckduckgo.com`
– si quelqu’un essaie `javascript:alert(1)`, c’est bloqué.

---

## 4. Implémenter l’appli `createBrowserApp`

Toujours dans le `<script>`, ajoute cette fonction (juste après les autres `createXXXApp`, par exemple après le terminal ou Alerte Cyber) :

```js
function createBrowserApp(container) {
  container.innerHTML = `
    <p><strong>Navigateur NYXO</strong></p>
    <p class="helper-text">
      Mini navigateur pédagogique. Certains sites peuvent refuser d’être affichés
      en iframe (politique de sécurité).
    </p>

    <div class="browser-toolbar">
      <button class="btn-ghost" id="navBackBtn" disabled>⬅️</button>
      <button class="btn-ghost" id="navForwardBtn" disabled>➡️</button>
      <button class="btn-ghost" id="navHomeBtn">🏠</button>
      <button class="btn-ghost" id="navReloadBtn">🔄</button>
      <input type="text" id="navAddress" class="browser-addr"
             placeholder="https://… ou domaine (ex: duckduckgo.com)">
      <button class="btn-ghost" id="navGoBtn">GO</button>
    </div>

    <div class="browser-status" id="navStatus">
      Prêt. Tape une URL (par ex. <code>duckduckgo.com</code>) puis appuie sur GO ou Entrée.
    </div>

    <div class="browser-frame-wrap">
      <iframe id="navFrame" class="browser-iframe"
              sandbox="allow-same-origin allow-scripts allow-forms allow-popups"
              referrerpolicy="strict-origin-when-cross-origin">
      </iframe>
    </div>
  `;

  const backBtn    = container.querySelector("#navBackBtn");
  const fwdBtn     = container.querySelector("#navForwardBtn");
  const homeBtn    = container.querySelector("#navHomeBtn");
  const reloadBtn  = container.querySelector("#navReloadBtn");
  const goBtn      = container.querySelector("#navGoBtn");
  const addrInput  = container.querySelector("#navAddress");
  const frame      = container.querySelector("#navFrame");
  const statusEl   = container.querySelector("#navStatus");

  const HOME_URL = "https://duckduckgo.com";

  const history = [];
  let historyIndex = -1;
  let currentUrl = "";

  function updateHistoryButtons() {
    backBtn.disabled = historyIndex <= 0;
    fwdBtn.disabled  = historyIndex < 0 || historyIndex >= history.length - 1;
  }

  function setStatus(msg) {
    statusEl.textContent = msg;
  }

  function load(url, push = true) {
    const normalized = normalizeUrl(url);
    if (!normalized) return;

    currentUrl = normalized;
    frame.src = normalized;
    addrInput.value = normalized;
    setStatus(`Chargement : ${normalized}`);

    if (push) {
      // On tronque la suite de l'historique puis on pousse la nouvelle URL
      history.splice(historyIndex + 1);
      history.push(normalized);
      historyIndex = history.length - 1;
      updateHistoryButtons();
    }
  }

  backBtn.addEventListener("click", () => {
    if (historyIndex > 0) {
      historyIndex--;
      const url = history[historyIndex];
      load(url, false);
    }
  });

  fwdBtn.addEventListener("click", () => {
    if (historyIndex < history.length - 1) {
      historyIndex++;
      const url = history[historyIndex];
      load(url, false);
    }
  });

  homeBtn.addEventListener("click", () => {
    load(HOME_URL, true);
  });

  reloadBtn.addEventListener("click", () => {
    if (currentUrl) {
      load(currentUrl, false);
    }
  });

  goBtn.addEventListener("click", () => {
    if (addrInput.value.trim()) {
      load(addrInput.value, true);
    }
  });

  addrInput.addEventListener("keydown", (e) => {
    if (e.key === "Enter") {
      e.preventDefault();
      if (addrInput.value.trim()) {
        load(addrInput.value, true);
      }
    }
  });

  frame.addEventListener("load", () => {
    if (currentUrl) {
      setStatus(`Page chargée : ${currentUrl}`);
    } else {
      setStatus("Prêt.");
    }
  });

  // On charge la page d’accueil au démarrage
  load(HOME_URL, true);
}
```

---

## 5. Ce que tu obtiens

Une fois ces 4 étapes faites :

* Tu vois une icône **🌐 Navigateur** sur le bureau NYXO.
* Quand tu l’ouvres :

  * tu as une barre d’adresse,
  * des boutons de navigation,
  * et la page qui s’affiche en dessous.

Si tu tombes sur une page blanche, c’est souvent parce que le site refuse l’iframe (X-Frame-Options, CSP). Teste avec des sites “cool” genre :

* `duckduckgo.com`
* ton propre site,
* un site statique perso.

Si tu veux ensuite **restreindre** le navigateur à ton écosystème (par ex. autoriser seulement `ouaisfi.eu` et tes sous-domaines), je peux te montrer comment filtrer les URL dans `load()` pour en faire un “kiosque web NYXO”.
