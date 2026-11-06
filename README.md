# 📱 WatchMate TV — Mobile Web-Companion (Implementierungs-Spezifikation & Vollständiger Quellcode)

> **🤖 ANWEISUNG FÜR DEN AI-CODING-AGENTEN:**
> Bitte integriere die mobile **WatchMate TV Web-Companion App (PWA)** in dieses Repository.
> 1. Erstelle im Hauptverzeichnis des Projekts den Ordner `web-companion/`.
> 2. Lege darin **alle 7 unten aufgeführten Dateien** (`index.html`, `styles.css`, `app.js`, `manifest.json`, `sw.js`, `firestore.rules`, `firebase.json`) mit dem **exakt angegebenen, vollständigen Quellcode** an (keine Platzhalter, keine Kürzungen).
> 3. Verändere dabei keinen bestehenden Android-TV-Code, sofern nicht explizit angefordert.

---

## 🏗️ Architektur & Schnittstellen-Vertrag (TV ↔ Mobile Web)

Die Web-App ist eine eigenständige, ohne Build-Step lauffähige **Progressive Web App (PWA)** auf Basis von nativen ES-Modulen und dem modularen **Firebase JS SDK v10**.

### 1. QR-Code Deep-Link / Kopplungs-URL
Wenn die Android-TV-App einen QR-Code zur Kopplung anzeigt, kann sie folgende URL-Parameter übergeben, die von der Web-App beim Öffnen automatisch ausgelesen, im `localStorage` gespeichert und danach aus der Browser-Adresszeile entfernt werden:
```text
https://<deine-domain>/?tmdb_key=<TMDB_API_KEY>&region=DE
```

### 2. Cloud Firestore Datenmodell
- **Pfad:** `/users/{userId}/watchlist/{itemId}`
- **Authentifizierung:** Google Sign-In (`Firebase Auth`) mit demselben Google-Konto wie auf dem Android TV.
- **Dokument-Schema (`WatchlistItem`):**
```json
{
  "id": 693134,
  "mediaType": "movie",
  "title": "Dune: Part Two",
  "posterPath": "/8b8R8l88Qje9dn9OE8PY05Nxl1X.jpg",
  "backdropPath": "/xOMo8BRK7PfcJv9JCnx7s5hj0x2.jpg",
  "overview": "Paul Atreides vereinigt sich mit Chani...",
  "releaseDate": "2024-02-27",
  "voteAverage": 8.2,
  "status": "PLAN_TO_WATCH",
  "currentSeason": 1,
  "currentEpisode": 1,
  "totalSeasons": 0,
  "seasonEpisodeCount": "{}",
  "addedAt": 1730847600000,
  "lastWatchedAt": 0,
  "streamingProviders": "[\"wow\",\"prime\"]",
  "source": "mobile_web",
  "createdAt": "SERVER_TIMESTAMP",
  "updatedAt": "SERVER_TIMESTAMP"
}
```

---

## 📂 Ziel-Verzeichnisstruktur

```text
web-companion/
├── index.html          # Haupt-UI (Touch-optimiert, Dark-Mode #060A18)
├── styles.css          # WatchMate TV Design-System, Animationen & Glassmorphism
├── app.js              # Logik: TMDb Live-Suche (300ms Debounce), Provider-Mapping, Firestore Sync
├── manifest.json       # PWA-Manifest für Installation auf dem Smartphone-Homescreen
├── sw.js               # Service Worker für Offline-Caching & schnellen Start
├── firestore.rules     # Zero-Trust Firestore Sicherheitsregeln
└── firebase.json       # Firebase Hosting & Firestore Konfiguration
```

---

## 📄 Datei 1: `web-companion/index.html`

```html
<!DOCTYPE html>
<html lang="de" class="dark">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no, viewport-fit=cover" />
  <meta name="theme-color" content="#060A18" />
  <meta name="apple-mobile-web-app-capable" content="yes" />
  <meta name="apple-mobile-web-app-status-bar-style" content="black-translucent" />
  <title>WatchMate TV — Mobile Companion</title>

  <link rel="manifest" href="./manifest.json" />
  <link rel="stylesheet" href="./styles.css" />
  <link rel="preconnect" href="https://fonts.googleapis.com">
  <link rel="preconnect" href="https://fonts.gstatic.com" crossorigin>
  <link href="https://fonts.googleapis.com/css2?family=Inter:wght@400;500;600;700;800&display=swap" rel="stylesheet">
</head>
<body>
  <!-- Hintergrund-Glow-Effekte passend zum WatchMate TV-Design -->
  <div class="ambient-glow glow-cyan"></div>
  <div class="ambient-glow glow-red"></div>

  <!-- App Container -->
  <div class="app-shell">

    <!-- Top Header -->
    <header class="app-header">
      <div class="brand">
        <div class="brand-logo" aria-hidden="true">
          <svg width="22" height="22" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.2" stroke-linecap="round" stroke-linejoin="round">
            <rect x="2" y="7" width="20" height="15" rx="2" ry="2"></rect>
            <polyline points="17 2 12 7 7 2"></polyline>
          </svg>
        </div>
        <div>
          <h1 class="brand-title">WatchMate <span class="badge-tv">TV</span></h1>
          <div class="sync-status" id="syncStatusIndicator">
            <span class="status-dot" id="syncStatusDot"></span>
            <span id="syncStatusText">Lokaler Modus</span>
          </div>
        </div>
      </div>

      <div class="header-actions">
        <!-- Region Badge -->
        <button id="regionToggleBtn" class="icon-chip" title="Streaming-Region wechseln">
          <span id="currentRegionFlag">🇩🇪</span>
          <span id="currentRegionCode">DE</span>
        </button>

        <!-- Auth / Profil Button -->
        <button id="authBtn" class="icon-btn" title="Google-Konto für TV-Sync">
          <img id="userAvatar" class="user-avatar hidden" alt="Profilbild" />
          <svg id="userIconDefault" width="20" height="20" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
            <path d="M20 21v-2a4 4 0 0 0-4-4H8a4 4 0 0 0-4 4v2"></path>
            <circle cx="12" cy="7" r="4"></circle>
          </svg>
        </button>

        <!-- Einstellungen Button -->
        <button id="settingsBtn" class="icon-btn" title="API & Abos konfigurieren">
          <svg width="20" height="20" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
            <circle cx="12" cy="12" r="3"></circle>
            <path d="M19.4 15a1.65 1.65 0 0 0 .33 1.82l.06.06a2 2 0 0 1 0 2.83 2 2 0 0 1-2.83 0l-.06-.06a1.65 1.65 0 0 0-1.82-.33 1.65 1.65 0 0 0-1 1.51V21a2 2 0 0 1-2 2 2 2 0 0 1-2-2v-.09A1.65 1.65 0 0 0 9 19.4a1.65 1.65 0 0 0-1.82.33l-.06.06a2 2 0 0 1-2.83 0 2 2 0 0 1 0-2.83l.06-.06a1.65 1.65 0 0 0 .33-1.82 1.65 1.65 0 0 0-1.51-1H3a2 2 0 0 1-2-2 2 2 0 0 1 2-2h.09A1.65 1.65 0 0 0 4.6 9a1.65 1.65 0 0 0-.33-1.82l-.06-.06a2 2 0 0 1 0-2.83 2 2 0 0 1 2.83 0l.06.06a1.65 1.65 0 0 0 1.82.33H9a1.65 1.65 0 0 0 1-1.51V3a2 2 0 0 1 2-2 2 2 0 0 1 2 2v.09a1.65 1.65 0 0 0 1 1.51 1.65 1.65 0 0 0 1.82-.33l.06-.06a2 2 0 0 1 2.83 0 2 2 0 0 1 0 2.83l-.06.06a1.65 1.65 0 0 0-.33 1.82V9a1.65 1.65 0 0 0 1.51 1H21a2 2 0 0 1 2 2 2 2 0 0 1-2 2h-.09a1.65 1.65 0 0 0-1.51 1z"></path>
          </svg>
        </button>
      </div>
    </header>

    <!-- Suchleiste (Sticky) -->
    <section class="search-section">
      <div class="search-box">
        <svg class="search-icon" width="20" height="20" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.2">
          <circle cx="11" cy="11" r="8"></circle>
          <line x1="21" y1="21" x2="16.65" y2="16.65"></line>
        </svg>
        <input
          type="search"
          id="searchInput"
          placeholder="Film oder Serie für TV suchen..."
          autocomplete="off"
          autocorrect="off"
          spellcheck="false"
        />
        <button id="clearSearchBtn" class="clear-btn hidden" aria-label="Suche löschen">✕</button>
      </div>

      <!-- Hauptnavigation: Entdecken / Meine TV-Watchlist -->
      <div class="nav-tabs" role="tablist">
        <button class="nav-tab active" data-tab="search" id="tabSearchBtn">
          <span>🔍 Suche & Trends</span>
        </button>
        <button class="nav-tab" data-tab="watchlist" id="tabWatchlistBtn">
          <span>📺 TV-Watchlist</span>
          <span class="tab-counter" id="watchlistCounter">0</span>
        </button>
      </div>
    </section>

    <!-- TAB 1: SUCHE & ENTDECKEN -->
    <main id="searchView" class="view-container">
      <div class="section-header">
        <h2 id="searchSectionTitle">Beliebte Empfehlungen</h2>
        <span id="searchMetaInfo" class="meta-subtitle">Sofort zum TV senden</span>
      </div>

      <!-- Lade-Spinner -->
      <div id="searchLoading" class="loading-state hidden">
        <div class="spinner"></div>
        <p>Durchsuche TMDb & Streaming-Anbieter...</p>
      </div>

      <!-- Ergebnisliste -->
      <div id="searchResultsGrid" class="media-list"></div>
    </main>

    <!-- TAB 2: MEINE TV-WATCHLIST (Echtzeit-Sync) -->
    <main id="watchlistView" class="view-container hidden">
      <div class="filter-bar">
        <button class="filter-pill active" data-filter="ALL">Alle</button>
        <button class="filter-pill" data-filter="PLAN_TO_WATCH">Geplant</button>
        <button class="filter-pill" data-filter="WATCHING">Schaue ich</button>
        <button class="filter-pill" data-filter="movie">Filme</button>
        <button class="filter-pill" data-filter="tv">Serien</button>
      </div>

      <div id="watchlistEmptyState" class="empty-state hidden">
        <div class="empty-icon">🍿</div>
        <h3>Deine TV-Watchlist ist noch leer</h3>
        <p>Suche oben nach einem Film oder einer Serie und tippe auf „Auf die Watchlist“.</p>
      </div>

      <div id="watchlistGrid" class="media-list"></div>
    </main>

  </div>

  <!-- Toast-Benachrichtigung (Unten) -->
  <div id="toastContainer" class="toast-container" aria-live="polite"></div>

  <!-- MODAL: EINSTELLUNGEN (Firebase Auth, TMDb API, Eigene Streaming-Abos) -->
  <div id="settingsModal" class="modal-backdrop hidden" role="dialog" aria-modal="true">
    <div class="modal-sheet">
      <div class="modal-header">
        <h2>⚙️ Kopplung & Einstellungen</h2>
        <button id="closeSettingsBtn" class="icon-btn">✕</button>
      </div>

      <div class="modal-body">
        <!-- 1. Google Cloud Sync Status -->
        <div class="settings-card">
          <div class="card-title-row">
            <h3>☁️ Google TV Cloud-Sync</h3>
            <span id="authBadge" class="status-pill offline">Nicht angemeldet</span>
          </div>
          <p class="settings-help">
            Melde dich mit demselben Google-Konto an wie auf deinem Android TV.
          </p>
          <div id="authUserInfo" class="auth-user-box hidden"></div>
          <div class="btn-row">
            <button id="googleSignInBtn" class="btn btn-primary full-width">
              Mit Google anmelden
            </button>
            <button id="googleSignOutBtn" class="btn btn-outline full-width hidden">
              Abmelden
            </button>
          </div>
        </div>

        <!-- 2. Eigene Streaming-Abonnements -->
        <div class="settings-card">
          <h3>🍿 Meine Streaming-Abos</h3>
          <p class="settings-help">
            Aktivierte Anbieter werden bei den Suchergebnissen farblich hervorgehoben.
          </p>
          <div id="subscriptionsGrid" class="subs-grid"></div>
        </div>

        <!-- 3. TMDb API-Konfiguration -->
        <div class="settings-card">
          <h3>🎬 TMDb API-Schlüssel</h3>
          <p class="settings-help">
            Für weltweite Echtzeit-Suche, FSK-Freigaben und Streaming-Verfügbarkeit in DE/AT/CH.
          </p>
          <label class="input-label" for="tmdbApiKeyInput">TMDb API Key (v3) oder Read Access Token (v4)</label>
          <input
            type="password"
            id="tmdbApiKeyInput"
            class="text-input"
            placeholder="z.B. 8a7b9c... oder eyJhbGci..."
          />
        </div>

        <!-- 4. Firebase Projekt-Konfiguration -->
        <div class="settings-card">
          <h3>🔥 Firebase Projekt-Konfiguration</h3>
          <p class="settings-help">
            Füge dein Firebase Web-Config JSON-Objekt ein (oder hinterlege es direkt in <code>app.js</code>).
          </p>
          <textarea
            id="firebaseConfigInput"
            class="text-input code-textarea"
            rows="5"
            placeholder='{"apiKey": "...", "authDomain": "...", "projectId": "..."}'
          ></textarea>
        </div>

        <button id="saveSettingsBtn" class="btn btn-primary full-width">
          Einstellungen speichern & neu verbinden
        </button>
      </div>
    </div>
  </div>

  <!-- Haupt-Skript -->
  <script type="module" src="./app.js"></script>
</body>
</html>
```

---

## 🎨 Datei 2: `web-companion/styles.css`

```css
:root {
  --bg-deep: #060A18;
  --bg-surface: #0E162B;
  --bg-surface-elevated: #16213E;
  --bg-glass: rgba(14, 22, 43, 0.82);

  --accent-cyan: #38BDF8;
  --accent-cyan-glow: rgba(56, 189, 248, 0.25);
  --accent-red: #E11D48;
  --accent-red-glow: rgba(225, 29, 72, 0.28);
  --accent-gold: #F59E0B;
  --accent-green: #10B981;

  --text-primary: #F8FAFC;
  --text-secondary: #94A3B8;
  --text-muted: #64748B;

  --border-subtle: rgba(148, 163, 184, 0.14);
  --border-cyan: rgba(56, 189, 248, 0.5);

  --radius-sm: 8px;
  --radius-md: 14px;
  --radius-lg: 20px;
}

* {
  box-sizing: border-box;
  margin: 0;
  padding: 0;
  -webkit-tap-highlight-color: transparent;
}

body {
  font-family: 'Inter', -apple-system, BlinkMacSystemFont, sans-serif;
  background-color: var(--bg-deep);
  color: var(--text-primary);
  min-height: 100dvh;
  overflow-x: hidden;
  line-height: 1.45;
}

.hidden {
  display: none !important;
}

/* Ambient Background Glows */
.ambient-glow {
  position: fixed;
  width: 340px;
  height: 340px;
  border-radius: 50%;
  filter: blur(110px);
  pointer-events: none;
  z-index: 0;
  opacity: 0.45;
}

.glow-cyan {
  top: -120px;
  left: -80px;
  background: var(--accent-cyan);
}

.glow-red {
  bottom: -120px;
  right: -80px;
  background: var(--accent-red);
}

/* App Shell */
.app-shell {
  position: relative;
  z-index: 1;
  max-width: 600px;
  margin: 0 auto;
  padding: 0 16px 96px;
}

/* Header */
.app-header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 16px 0 12px;
}

.brand {
  display: flex;
  align-items: center;
  gap: 12px;
}

.brand-logo {
  width: 42px;
  height: 42px;
  border-radius: 12px;
  background: linear-gradient(135deg, rgba(56, 189, 248, 0.2), rgba(225, 29, 72, 0.25));
  border: 1px solid var(--border-cyan);
  display: flex;
  align-items: center;
  justify-content: center;
  color: var(--accent-cyan);
  box-shadow: 0 4px 16px var(--accent-cyan-glow);
}

.brand-title {
  font-size: 1.15rem;
  font-weight: 800;
  letter-spacing: -0.02em;
  display: flex;
  align-items: center;
  gap: 6px;
}

.badge-tv {
  font-size: 0.7rem;
  font-weight: 700;
  background: var(--accent-red);
  color: #fff;
  padding: 2px 6px;
  border-radius: 5px;
  letter-spacing: 0.04em;
}

.sync-status {
  display: flex;
  align-items: center;
  gap: 6px;
  font-size: 0.75rem;
  color: var(--text-secondary);
  margin-top: 2px;
}

.status-dot {
  width: 7px;
  height: 7px;
  border-radius: 50%;
  background: var(--accent-gold);
}

.status-dot.online {
  background: var(--accent-green);
  box-shadow: 0 0 8px var(--accent-green);
}

.header-actions {
  display: flex;
  align-items: center;
  gap: 8px;
}

.icon-btn, .icon-chip {
  background: var(--bg-surface);
  border: 1px solid var(--border-subtle);
  color: var(--text-primary);
  height: 40px;
  border-radius: 12px;
  display: flex;
  align-items: center;
  justify-content: center;
  cursor: pointer;
  transition: transform 0.15s ease, border-color 0.2s ease;
}

.icon-btn {
  width: 40px;
}

.icon-chip {
  padding: 0 10px;
  gap: 5px;
  font-size: 0.8rem;
  font-weight: 600;
}

.icon-btn:active, .icon-chip:active {
  transform: scale(0.93);
}

.user-avatar {
  width: 100%;
  height: 100%;
  border-radius: 11px;
  object-fit: cover;
}

/* Sticky Search Section */
.search-section {
  position: sticky;
  top: 0;
  z-index: 20;
  background: rgba(6, 10, 24, 0.88);
  backdrop-filter: blur(16px);
  -webkit-backdrop-filter: blur(16px);
  padding: 10px 0 12px;
}

.search-box {
  position: relative;
  display: flex;
  align-items: center;
}

.search-icon {
  position: absolute;
  left: 15px;
  color: var(--accent-cyan);
  pointer-events: none;
}

#searchInput {
  width: 100%;
  height: 52px;
  padding: 0 44px 0 46px;
  border-radius: var(--radius-md);
  background: var(--bg-surface);
  border: 1.5px solid var(--border-subtle);
  color: var(--text-primary);
  font-size: 1rem;
  font-weight: 500;
  outline: none;
  transition: border-color 0.2s, box-shadow 0.2s;
}

#searchInput:focus {
  border-color: var(--accent-cyan);
  box-shadow: 0 0 0 3px var(--accent-cyan-glow);
}

#searchInput::placeholder {
  color: var(--text-muted);
}

.clear-btn {
  position: absolute;
  right: 12px;
  width: 28px;
  height: 28px;
  border-radius: 50%;
  border: none;
  background: var(--bg-surface-elevated);
  color: var(--text-secondary);
  cursor: pointer;
  font-size: 0.85rem;
}

/* Navigation Tabs */
.nav-tabs {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 8px;
  margin-top: 12px;
  background: var(--bg-surface);
  padding: 4px;
  border-radius: 12px;
  border: 1px solid var(--border-subtle);
}

.nav-tab {
  background: transparent;
  border: none;
  color: var(--text-secondary);
  font-size: 0.88rem;
  font-weight: 600;
  padding: 9px 12px;
  border-radius: 9px;
  cursor: pointer;
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 6px;
  transition: all 0.2s ease;
}

.nav-tab.active {
  background: var(--bg-surface-elevated);
  color: var(--accent-cyan);
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.3);
}

.tab-counter {
  background: rgba(56, 189, 248, 0.18);
  color: var(--accent-cyan);
  font-size: 0.72rem;
  padding: 1px 7px;
  border-radius: 99px;
}

/* Section Headers */
.section-header {
  display: flex;
  align-items: baseline;
  justify-content: space-between;
  margin: 8px 0 14px;
}

.section-header h2 {
  font-size: 1.05rem;
  font-weight: 700;
}

.meta-subtitle {
  font-size: 0.78rem;
  color: var(--text-muted);
}

/* Filter Bar */
.filter-bar {
  display: flex;
  gap: 8px;
  overflow-x: auto;
  padding: 4px 0 14px;
  scrollbar-width: none;
}

.filter-bar::-webkit-scrollbar {
  display: none;
}

.filter-pill {
  white-space: nowrap;
  padding: 7px 14px;
  border-radius: 99px;
  font-size: 0.82rem;
  font-weight: 600;
  background: var(--bg-surface);
  color: var(--text-secondary);
  border: 1px solid var(--border-subtle);
  cursor: pointer;
}

.filter-pill.active {
  background: var(--accent-cyan);
  color: #04101E;
  border-color: var(--accent-cyan);
}

/* Media Card List */
.media-list {
  display: flex;
  flex-direction: column;
  gap: 14px;
}

.media-card {
  background: var(--bg-surface);
  border: 1px solid var(--border-subtle);
  border-radius: var(--radius-md);
  overflow: hidden;
  transition: border-color 0.2s ease, transform 0.2s ease;
}

.media-card.in-watchlist {
  border-color: rgba(56, 189, 248, 0.45);
}

.media-card-main {
  display: flex;
  gap: 14px;
  padding: 12px;
}

.poster-wrap {
  position: relative;
  width: 92px;
  min-width: 92px;
  height: 138px;
  border-radius: 10px;
  overflow: hidden;
  background: var(--bg-surface-elevated);
}

.poster-img {
  width: 100%;
  height: 100%;
  object-fit: cover;
  display: block;
}

.media-type-badge {
  position: absolute;
  top: 6px;
  left: 6px;
  font-size: 0.65rem;
  font-weight: 700;
  text-transform: uppercase;
  padding: 2px 6px;
  border-radius: 5px;
  background: rgba(6, 10, 24, 0.85);
  color: var(--accent-cyan);
  backdrop-filter: blur(4px);
}

.media-info {
  flex: 1;
  min-width: 0;
  display: flex;
  flex-direction: column;
}

.media-title {
  font-size: 1rem;
  font-weight: 700;
  color: var(--text-primary);
  margin-bottom: 4px;
  display: -webkit-box;
  -webkit-line-clamp: 2;
  -webkit-box-orient: vertical;
  overflow: hidden;
}

.media-meta-row {
  display: flex;
  flex-wrap: wrap;
  align-items: center;
  gap: 6px;
  font-size: 0.76rem;
  color: var(--text-secondary);
  margin-bottom: 6px;
}

.rating-badge {
  color: var(--accent-gold);
  font-weight: 700;
}

.age-badge {
  border: 1px solid var(--border-subtle);
  padding: 1px 5px;
  border-radius: 4px;
  font-size: 0.68rem;
  font-weight: 600;
}

.media-overview {
  font-size: 0.8rem;
  color: var(--text-secondary);
  display: -webkit-box;
  -webkit-line-clamp: 2;
  -webkit-box-orient: vertical;
  overflow: hidden;
  margin-bottom: 8px;
}

/* Streaming Providers Row */
.providers-row {
  display: flex;
  flex-wrap: wrap;
  gap: 6px;
  margin-top: auto;
}

.provider-pill {
  font-size: 0.7rem;
  font-weight: 600;
  padding: 3px 8px;
  border-radius: 6px;
  background: var(--bg-surface-elevated);
  color: var(--text-secondary);
  border: 1px solid var(--border-subtle);
  display: inline-flex;
  align-items: center;
  gap: 4px;
}

.provider-pill.subscribed {
  background: rgba(56, 189, 248, 0.16);
  color: var(--accent-cyan);
  border-color: var(--accent-cyan);
}

/* Action Buttons Bar inside Card */
.card-actions {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 8px;
  padding: 10px 12px;
  background: rgba(6, 10, 24, 0.45);
  border-top: 1px solid var(--border-subtle);
}

.action-btn {
  border: none;
  border-radius: 10px;
  padding: 10px 12px;
  font-size: 0.82rem;
  font-weight: 700;
  cursor: pointer;
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 6px;
  transition: transform 0.12s ease, opacity 0.2s ease;
}

.action-btn:active {
  transform: scale(0.96);
}

.btn-watchlist {
  background: rgba(56, 189, 248, 0.15);
  color: var(--accent-cyan);
  border: 1px solid rgba(56, 189, 248, 0.4);
}

.btn-watchlist.active {
  background: var(--accent-cyan);
  color: #05101F;
}

.btn-watching {
  background: var(--accent-red);
  color: #fff;
  box-shadow: 0 3px 12px var(--accent-red-glow);
}

.btn-watching.active {
  background: var(--accent-green);
  box-shadow: none;
}

.btn-danger-Ghost {
  background: rgba(225, 29, 72, 0.12);
  color: #FDA4AF;
  border: 1px solid rgba(225, 29, 72, 0.3);
}

/* Episode Tracker Bar (for TV Shows in Watchlist) */
.episode-tracker {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 8px 12px;
  background: rgba(56, 189, 248, 0.07);
  border-top: 1px solid var(--border-subtle);
  font-size: 0.82rem;
}

.ep-badge {
  font-weight: 700;
  color: var(--accent-cyan);
}

.ep-next-btn {
  background: var(--bg-surface-elevated);
  color: var(--text-primary);
  border: 1px solid var(--border-cyan);
  padding: 5px 10px;
  border-radius: 8px;
  font-size: 0.76rem;
  font-weight: 600;
  cursor: pointer;
}

/* Toast Notification */
.toast-container {
  position: fixed;
  bottom: 20px;
  left: 16px;
  right: 16px;
  max-width: 560px;
  margin: 0 auto;
  z-index: 100;
  display: flex;
  flex-direction: column;
  gap: 8px;
  pointer-events: none;
}

.toast {
  background: rgba(14, 22, 43, 0.96);
  border: 1.5px solid var(--accent-cyan);
  color: var(--text-primary);
  padding: 13px 16px;
  border-radius: 14px;
  box-shadow: 0 10px 30px rgba(0, 0, 0, 0.65);
  display: flex;
  align-items: center;
  gap: 10px;
  font-size: 0.86rem;
  font-weight: 600;
  animation: toastSlideUp 0.25s cubic-bezier(0.16, 1, 0.3, 1);
}

@keyframes toastSlideUp {
  from {
    opacity: 0;
    transform: translateY(20px) scale(0.95);
  }
  to {
    opacity: 1;
    transform: translateY(0) scale(1);
  }
}

/* Loading & Empty States */
.loading-state, .empty-state {
  text-align: center;
  padding: 48px 20px;
  color: var(--text-secondary);
}

.spinner {
  width: 34px;
  height: 34px;
  border: 3px solid var(--border-subtle);
  border-top-color: var(--accent-cyan);
  border-radius: 50%;
  margin: 0 auto 12px;
  animation: spin 0.8s linear infinite;
}

@keyframes spin {
  to { transform: rotate(360deg); }
}

.empty-icon {
  font-size: 2.5rem;
  margin-bottom: 10px;
}

/* Settings Modal */
.modal-backdrop {
  position: fixed;
  inset: 0;
  background: rgba(3, 6, 15, 0.78);
  backdrop-filter: blur(8px);
  z-index: 50;
  display: flex;
  align-items: flex-end;
  justify-content: center;
}

.modal-sheet {
  width: 100%;
  max-width: 600px;
  max-height: 90dvh;
  background: var(--bg-surface);
  border-top: 1px solid var(--border-cyan);
  border-radius: 22px 22px 0 0;
  display: flex;
  flex-direction: column;
  overflow: hidden;
}

.modal-header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 16px 20px;
  border-bottom: 1px solid var(--border-subtle);
}

.modal-header h2 {
  font-size: 1.05rem;
}

.modal-body {
  padding: 16px 20px 32px;
  overflow-y: auto;
  display: flex;
  flex-direction: column;
  gap: 16px;
}

.settings-card {
  background: var(--bg-deep);
  border: 1px solid var(--border-subtle);
  padding: 14px;
  border-radius: 14px;
}

.card-title-row {
  display: flex;
  align-items: center;
  justify-content: space-between;
  margin-bottom: 6px;
}

.settings-card h3 {
  font-size: 0.92rem;
  margin-bottom: 4px;
}

.settings-help {
  font-size: 0.78rem;
  color: var(--text-secondary);
  margin-bottom: 12px;
}

.status-pill {
  font-size: 0.7rem;
  font-weight: 700;
  padding: 3px 8px;
  border-radius: 99px;
}

.status-pill.online {
  background: rgba(16, 185, 129, 0.2);
  color: var(--accent-green);
}

.status-pill.offline {
  background: rgba(245, 158, 11, 0.2);
  color: var(--accent-gold);
}

.subs-grid {
  display: flex;
  flex-wrap: wrap;
  gap: 8px;
}

.sub-chip {
  padding: 7px 11px;
  border-radius: 9px;
  font-size: 0.78rem;
  font-weight: 600;
  background: var(--bg-surface);
  color: var(--text-secondary);
  border: 1px solid var(--border-subtle);
  cursor: pointer;
}

.sub-chip.active {
  background: rgba(56, 189, 248, 0.18);
  color: var(--accent-cyan);
  border-color: var(--accent-cyan);
}

.input-label {
  display: block;
  font-size: 0.75rem;
  color: var(--text-secondary);
  margin-bottom: 6px;
}

.text-input {
  width: 100%;
  padding: 10px 12px;
  border-radius: 10px;
  background: var(--bg-surface);
  border: 1px solid var(--border-subtle);
  color: var(--text-primary);
  font-size: 0.85rem;
  font-family: inherit;
}

.code-textarea {
  font-family: monospace;
  font-size: 0.76rem;
}

.btn {
  padding: 12px 16px;
  border-radius: 12px;
  font-size: 0.9rem;
  font-weight: 700;
  cursor: pointer;
  border: none;
}

.btn-primary {
  background: var(--accent-cyan);
  color: #041221;
}

.btn-outline {
  background: transparent;
  color: var(--text-primary);
  border: 1px solid var(--border-subtle);
}

.full-width {
  width: 100%;
}

.auth-user-box {
  font-size: 0.82rem;
  color: var(--accent-cyan);
  margin-bottom: 10px;
  padding: 8px 10px;
  background: rgba(56, 189, 248, 0.08);
  border-radius: 8px;
}
```

---

## ⚙️ Datei 3: `web-companion/app.js`

```javascript
import { initializeApp, getApps } from "https://www.gstatic.com/firebasejs/10.13.1/firebase-app.js";
import {
  getAuth,
  GoogleAuthProvider,
  signInWithPopup,
  signOut,
  onAuthStateChanged
} from "https://www.gstatic.com/firebasejs/10.13.1/firebase-auth.js";
import {
  getFirestore,
  collection,
  doc,
  setDoc,
  deleteDoc,
  onSnapshot,
  serverTimestamp
} from "https://www.gstatic.com/firebasejs/10.13.1/firebase-firestore.js";

/* ==========================================================================
   1. KONSTANTEN & STREAMING-ANBIETER MAPPING (Kompatibel mit WatchMate TV)
   ========================================================================== */

const SUPPORTED_PROVIDERS = [
  { id: "netflix", label: "Netflix", tmdbIds: [8, 1796] },
  { id: "prime", label: "Prime Video", tmdbIds: [9, 119, 10] },
  { id: "disney", label: "Disney+", tmdbIds: [337] },
  { id: "wow", label: "WOW / Sky", tmdbIds: [30, 130, 29] },
  { id: "apple", label: "Apple TV+", tmdbIds: [350, 2] },
  { id: "paramount", label: "Paramount+", tmdbIds: [531] },
  { id: "rtl", label: "RTL+", tmdbIds: [298] },
  { id: "joyn", label: "Joyn", tmdbIds: [304, 421] },
  { id: "magenta", label: "MagentaTV", tmdbIds: [178] },
  { id: "crunchyroll", label: "Crunchyroll", tmdbIds: [283] }
];

const REGIONS = [
  { code: "DE", flag: "🇩🇪" },
  { code: "AT", flag: "🇦🇹" },
  { code: "CH", flag: "🇨🇭" }
];

// Kuratierte Beispieldaten (identisch zum WatchMate TV Starter-Katalog)
const CURATED_STARTER_CATALOG = [
  {
    id: 693134,
    mediaType: "movie",
    title: "Dune: Part Two",
    posterPath: "/8b8R8l88Qje9dn9OE8PY05Nxl1X.jpg",
    backdropPath: "/xOMo8BRK7PfcJv9JCnx7s5hj0x2.jpg",
    overview: "Paul Atreides vereinigt sich mit Chani und den Fremen, während er auf einem Rachefeldzug gegen die Verschwörer ist, die seine Familie zerstört haben.",
    releaseDate: "2024-02-27",
    voteAverage: 8.2,
    ageRating: "FSK 12",
    totalSeasons: 0,
    seasonEpisodeCount: "{}",
    streamingProviders: ["wow", "prime"]
  },
  {
    id: 95396,
    mediaType: "tv",
    title: "Severance",
    posterPath: "/pPHpeI2X1qEd1CS1SeyrdhZ4qnT.jpg",
    backdropPath: "/ixgFmf1X59PUZam2qbAfskx2gQr.jpg",
    overview: "Mark leitet ein Team von Büroangestellten, deren Erinnerungen zwischen Arbeit und Privatleben chirurgisch getrennt wurden.",
    releaseDate: "2022-02-17",
    voteAverage: 8.4,
    ageRating: "FSK 12",
    totalSeasons: 2,
    seasonEpisodeCount: JSON.stringify({ "1": 9, "2": 10 }),
    streamingProviders: ["apple"]
  },
  {
    id: 66732,
    mediaType: "tv",
    title: "Stranger Things",
    posterPath: "/49WJfeN0moxb9IPfGn8AIqMGskD.jpg",
    backdropPath: "/56v2KjBlU4XaOv9rVYEQypROD7P.jpg",
    overview: "Als ein kleiner Junge verschwindet, deckt eine Kleinstadt ein Geheimnis um geheime Experimente, übernatürliche Kräfte und ein seltsames Mädchen auf.",
    releaseDate: "2016-07-15",
    voteAverage: 8.6,
    ageRating: "FSK 16",
    totalSeasons: 4,
    seasonEpisodeCount: JSON.stringify({ "1": 8, "2": 9, "3": 8, "4": 9 }),
    streamingProviders: ["netflix"]
  },
  {
    id: 94605,
    mediaType: "tv",
    title: "Arcane",
    posterPath: "/fqldf2t8ztc9aiwn3k6mlX3tvRT.jpg",
    backdropPath: "/rkB4LyZHo1NHXFEDHl9vSD9r1lI.jpg",
    overview: "Inmitten des Konflikts zwischen den Zwillingsstädten Piltover und Zhaun kämpfen zwei Schwestern auf gegnerischen Seiten eines Krieges.",
    releaseDate: "2021-11-06",
    voteAverage: 8.8,
    ageRating: "FSK 16",
    totalSeasons: 2,
    seasonEpisodeCount: JSON.stringify({ "1": 9, "2": 9 }),
    streamingProviders: ["netflix"]
  },
  {
    id: 27205,
    mediaType: "movie",
    title: "Inception",
    posterPath: "/ljsZTbVsrQSqZgWeep2B1QiDKuh.jpg",
    backdropPath: "/8ZTVqvKDQ8emSGUEMjsS4yHAwrp.jpg",
    overview: "Dom Cobb ist ein begnadeter Dieb, der wertvolle Geheimnisse aus den Tiefen des Unterbewusstseins während der Traumphase stiehlt.",
    releaseDate: "2010-07-15",
    voteAverage: 8.4,
    ageRating: "FSK 12",
    totalSeasons: 0,
    seasonEpisodeCount: "{}",
    streamingProviders: ["prime", "netflix"]
  },
  {
    id: 1396,
    mediaType: "tv",
    title: "Breaking Bad",
    posterPath: "/ztkUQFLlC19CCMYHW9o1zWhJRNq.jpg",
    backdropPath: "/tsRy63Mu5cu8etL1X7ZLyf7UP1M.jpg",
    overview: "Ein an Lungenkrebs erkrankter Chemielehrer beginnt zusammen mit einem ehemaligen Schüler Methamphetamin herzustellen, um seine Familie abzusichern.",
    releaseDate: "2008-01-20",
    voteAverage: 8.9,
    ageRating: "FSK 16",
    totalSeasons: 5,
    seasonEpisodeCount: JSON.stringify({ "1": 7, "2": 13, "3": 13, "4": 13, "5": 16 }),
    streamingProviders: ["netflix"]
  }
];

/* ==========================================================================
   2. APP STATE & KONFIGURATION
   ========================================================================== */

const state = {
  region: localStorage.getItem("wm_region") || "DE",
  tmdbKey: localStorage.getItem("wm_tmdb_key") || "",
  subscriptions: JSON.parse(localStorage.getItem("wm_subs") || '["netflix","prime","disney"]'),
  firebaseConfig: JSON.parse(localStorage.getItem("wm_firebase_config") || "null"),
  currentUser: null,
  firestoreDb: null,
  authInstance: null,
  unsubscribeWatchlist: null,
  watchlistMap: new Map(), // Key: String(id), Value: WatchlistItem
  activeTab: "search",
  activeFilter: "ALL",
  searchResults: [...CURATED_STARTER_CATALOG]
};

// URL-Parameter aus QR-Code vom Fernseher auslesen (z. B. ?tmdb_key=...&region=DE)
function applyUrlPairingParams() {
  const params = new URLSearchParams(window.location.search);
  let updated = false;

  if (params.get("tmdb_key")) {
    state.tmdbKey = params.get("tmdb_key").trim();
    localStorage.setItem("wm_tmdb_key", state.tmdbKey);
    updated = true;
  }
  if (params.get("region") && ["DE", "AT", "CH"].includes(params.get("region").toUpperCase())) {
    state.region = params.get("region").toUpperCase();
    localStorage.setItem("wm_region", state.region);
    updated = true;
  }
  if (updated) {
    window.history.replaceState({}, document.title, window.location.pathname);
    showToast("🔗 TV-Parameter über QR-Code übernommen!");
  }
}

/* ==========================================================================
   3. FIREBASE AUTH & FIRESTORE REAL-TIME SYNC
   ========================================================================== */

function initFirebaseIfConfigured() {
  if (!state.firebaseConfig || !state.firebaseConfig.apiKey) {
    loadLocalWatchlistFallback();
    updateSyncStatusUI(false);
    return;
  }

  try {
    const app = getApps().length === 0 ? initializeApp(state.firebaseConfig) : getApps()[0];
    state.authInstance = getAuth(app);
    state.firestoreDb = getFirestore(app);

    onAuthStateChanged(state.authInstance, (user) => {
      state.currentUser = user;
      updateAuthModalUI(user);

      if (user) {
        updateSyncStatusUI(true, user.email || user.displayName);
        subscribeToUserWatchlist(user.uid);
      } else {
        if (state.unsubscribeWatchlist) state.unsubscribeWatchlist();
        loadLocalWatchlistFallback();
        updateSyncStatusUI(false);
      }
    });
  } catch (err) {
    console.error("Firebase Init Error:", err);
    showToast("⚠️ Firebase-Konfiguration ungültig. Nutze lokalen Speicher.");
    loadLocalWatchlistFallback();
  }
}

function subscribeToUserWatchlist(userId) {
  if (state.unsubscribeWatchlist) state.unsubscribeWatchlist();

  const watchlistRef = collection(state.firestoreDb, "users", userId, "watchlist");
  state.unsubscribeWatchlist = onSnapshot(
    watchlistRef,
    (snapshot) => {
      state.watchlistMap.clear();
      snapshot.forEach((docSnap) => {
        const data = docSnap.data();
        state.watchlistMap.set(String(data.id), data);
      });
      saveLocalWatchlistBackup();
      renderAll();
    },
    (error) => {
      console.error("Firestore Listener Error:", error);
      showToast("⚠️ Synchronisationsfehler mit Firestore");
    }
  );
}

function loadLocalWatchlistFallback() {
  const raw = localStorage.getItem("wm_local_watchlist");
  state.watchlistMap.clear();
  if (raw) {
    try {
      const arr = JSON.parse(raw);
      arr.forEach((item) => state.watchlistMap.set(String(item.id), item));
    } catch (e) {
      console.error(e);
    }
  }
  renderAll();
}

function saveLocalWatchlistBackup() {
  const arr = Array.from(state.watchlistMap.values());
  localStorage.setItem("wm_local_watchlist", JSON.stringify(arr));
}

/* ==========================================================================
   4. WATCHLIST SCHREIBOPERATIONEN (Exaktes Schema gemäß Architekturplan)
   ========================================================================== */

async function addOrUpdateWatchlistItem(mediaItem, targetStatus = "PLAN_TO_WATCH") {
  // Haptisches Feedback auf Smartphones
  if (navigator.vibrate) {
    navigator.vibrate([35, 50, 35]);
  }

  // Bei Bedarf vollständige TV-Staffeldaten & Streaming-Anbieter von TMDb nachladen
  const enriched = await enrichMediaDetailsIfNeeded(mediaItem);
  const existing = state.watchlistMap.get(String(enriched.id));
  const now = Date.now();

  const payload = {
    id: Number(enriched.id),
    mediaType: enriched.mediaType === "tv" ? "tv" : "movie",
    title: enriched.title || "Unbekannter Titel",
    posterPath: enriched.posterPath || "",
    backdropPath: enriched.backdropPath || "",
    overview: enriched.overview || "",
    releaseDate: enriched.releaseDate || "",
    voteAverage: Number(enriched.voteAverage || 0),
    status: targetStatus,
    currentSeason: existing?.currentSeason || 1,
    currentEpisode: existing?.currentEpisode || 1,
    totalSeasons: Number(enriched.totalSeasons || 0),
    seasonEpisodeCount:
      typeof enriched.seasonEpisodeCount === "string"
        ? enriched.seasonEpisodeCount
        : JSON.stringify(enriched.seasonEpisodeCount || {}),
    addedAt: existing?.addedAt || now,
    lastWatchedAt: targetStatus === "WATCHING" ? now : existing?.lastWatchedAt || 0,
    streamingProviders:
      typeof enriched.streamingProviders === "string"
        ? enriched.streamingProviders
        : JSON.stringify(enriched.streamingProviders || []),
    source: "mobile_web"
  };

  // Sofort lokal aktualisieren (Optimistic UI)
  state.watchlistMap.set(String(payload.id), payload);
  saveLocalWatchlistBackup();
  renderAll();

  // In Cloud Firestore speichern (/users/{userId}/watchlist/{itemId})
  if (state.currentUser && state.firestoreDb) {
    try {
      const itemRef = doc(
        state.firestoreDb,
        "users",
        state.currentUser.uid,
        "watchlist",
        String(payload.id)
      );
      await setDoc(
        itemRef,
        {
          ...payload,
          createdAt: existing?.createdAt || serverTimestamp(),
          updatedAt: serverTimestamp()
        },
        { merge: true }
      );
      showToast(`✅ „${payload.title}“ gespeichert! Sofort auf deinem TV verfügbar.`);
    } catch (err) {
      console.error("Firestore Write Error:", err);
      showToast("⚠️ Lokal gespeichert (Cloud-Schreibfehler)");
    }
  } else {
    showToast(`📌 „${payload.title}“ gespeichert! (Konto verbinden für Live-TV-Sync)`);
  }
}

async function incrementEpisode(itemId) {
  const item = state.watchlistMap.get(String(itemId));
  if (!item || item.mediaType !== "tv") return;

  if (navigator.vibrate) navigator.vibrate(25);

  let seasonMap = {};
  try {
    seasonMap = JSON.parse(item.seasonEpisodeCount || "{}");
  } catch (_) {}

  let nextSeason = item.currentSeason || 1;
  let nextEp = (item.currentEpisode || 1) + 1;
  const maxEpInCurrentSeason = seasonMap[String(nextSeason)] || 99;

  if (nextEp > maxEpInCurrentSeason) {
    if (item.totalSeasons > 0 && nextSeason < item.totalSeasons) {
      nextSeason += 1;
      nextEp = 1;
    } else if (item.totalSeasons > 0 && nextSeason >= item.totalSeasons) {
      item.status = "COMPLETED";
      nextEp = maxEpInCurrentSeason;
    }
  }

  item.currentSeason = nextSeason;
  item.currentEpisode = nextEp;
  item.lastWatchedAt = Date.now();
  if (item.status === "PLAN_TO_WATCH") item.status = "WATCHING";

  state.watchlistMap.set(String(item.id), item);
  saveLocalWatchlistBackup();
  renderAll();

  if (state.currentUser && state.firestoreDb) {
    const itemRef = doc(state.firestoreDb, "users", state.currentUser.uid, "watchlist", String(item.id));
    await setDoc(itemRef, { ...item, updatedAt: serverTimestamp() }, { merge: true });
  }
  showToast(`📺 ${item.title}: S${item.currentSeason} • E${item.currentEpisode}`);
}

async function removeWatchlistItem(itemId) {
  const item = state.watchlistMap.get(String(itemId));
  if (!item) return;

  state.watchlistMap.delete(String(itemId));
  saveLocalWatchlistBackup();
  renderAll();

  if (state.currentUser && state.firestoreDb) {
    const itemRef = doc(state.firestoreDb, "users", state.currentUser.uid, "watchlist", String(itemId));
    await deleteDoc(itemRef);
  }
  showToast(`🗑️ „${item.title}“ von der Watchlist entfernt.`);
}

/* ==========================================================================
   5. TMDB LIVE-SUCHE & METADATEN-ANREICHERUNG (DE / AT / CH)
   ========================================================================== */

function getTmdbFetchOptions() {
  const isBearer = state.tmdbKey.length > 45;
  return isBearer
    ? { headers: { Authorization: `Bearer ${state.tmdbKey}`, accept: "application/json" } }
    : {};
}

function buildTmdbUrl(endpoint, queryParams = {}) {
  const url = new URL(`https://api.themoviedb.org/3${endpoint}`);
  url.searchParams.set("language", "de-DE");
  if (state.tmdbKey && state.tmdbKey.length <= 45) {
    url.searchParams.set("api_key", state.tmdbKey);
  }
  Object.entries(queryParams).forEach(([k, v]) => url.searchParams.set(k, v));
  return url.toString();
}

async function performTmdbSearch(query) {
  const trimmed = query.trim();
  const titleEl = document.getElementById("searchSectionTitle");
  const loadingEl = document.getElementById("searchLoading");

  if (!trimmed) {
    titleEl.textContent = "Beliebte Empfehlungen";
    state.searchResults = [...CURATED_STARTER_CATALOG];
    renderSearchResults();
    return;
  }

  titleEl.textContent = `Suchergebnisse für „${trimmed}“`;

  // Falls noch kein TMDb-Key hinterlegt ist: Lokalen Katalog durchsuchen
  if (!state.tmdbKey) {
    const q = trimmed.toLowerCase();
    state.searchResults = CURATED_STARTER_CATALOG.filter(
      (item) => item.title.toLowerCase().includes(q) || item.overview.toLowerCase().includes(q)
    );
    renderSearchResults();
    return;
  }

  loadingEl.classList.remove("hidden");
  try {
    const url = buildTmdbUrl("/search/multi", {
      query: trimmed,
      include_adult: "false",
      region: state.region
    });
    const res = await fetch(url, getTmdbFetchOptions());
    if (!res.ok) throw new Error(`TMDb HTTP ${res.status}`);
    const data = await res.json();

    const filtered = (data.results || [])
      .filter((r) => r.media_type === "movie" || r.media_type === "tv")
      .slice(0, 12)
      .map((raw) => ({
        id: raw.id,
        mediaType: raw.media_type,
        title: raw.title || raw.name || "Unbekannt",
        posterPath: raw.poster_path || "",
        backdropPath: raw.backdrop_path || "",
        overview: raw.overview || "Keine deutsche Beschreibung verfügbar.",
        releaseDate: raw.release_date || raw.first_air_date || "",
        voteAverage: raw.vote_average ? Number(raw.vote_average.toFixed(1)) : 0,
        ageRating: "",
        totalSeasons: 0,
        seasonEpisodeCount: "{}",
        streamingProviders: [],
        _detailsLoaded: false
      }));

    state.searchResults = filtered;
    renderSearchResults();

    // Streaming-Anbieter & FSK für die Top-6-Treffer im Hintergrund nachladen
    filtered.slice(0, 6).forEach(async (item) => {
      await enrichMediaDetailsIfNeeded(item);
      renderSearchResults();
    });
  } catch (err) {
    console.error("TMDb Search Error:", err);
    showToast("⚠️ TMDb-Abfrage fehlgeschlagen. Prüfe deinen API-Key.");
  } finally {
    loadingEl.classList.add("hidden");
  }
}

async function enrichMediaDetailsIfNeeded(item) {
  if (item._detailsLoaded || !state.tmdbKey) return item;

  try {
    const append =
      item.mediaType === "movie" ? "watch/providers,release_dates" : "watch/providers,content_ratings";
    const url = buildTmdbUrl(`/${item.mediaType}/${item.id}`, { append_to_response: append });
    const res = await fetch(url, getTmdbFetchOptions());
    if (!res.ok) return item;

    const details = await res.json();

    // 1. Streaming-Provider in der gewählten Region (DE / AT / CH)
    const regionProviders = details["watch/providers"]?.results?.[state.region];
    const flatrateList = [
      ...(regionProviders?.flatrate || []),
      ...(regionProviders?.free || [])
    ];
    const matchedSlugs = new Set();
    flatrateList.forEach((p) => {
      const match = SUPPORTED_PROVIDERS.find((sp) => sp.tmdbIds.includes(p.provider_id));
      if (match) matchedSlugs.add(match.id);
    });
    item.streamingProviders = Array.from(matchedSlugs);

    // 2. Serien-Staffeln & Episodenanzahl
    if (item.mediaType === "tv") {
      item.totalSeasons = details.number_of_seasons || 0;
      const epMap = {};
      (details.seasons || []).forEach((s) => {
        if (s.season_number > 0) {
          epMap[String(s.season_number)] = s.episode_count || 1;
        }
      });
      item.seasonEpisodeCount = JSON.stringify(epMap);

      const ratingEntry = (details.content_ratings?.results || []).find(
        (r) => r.iso_3166_1 === state.region || r.iso_3166_1 === "DE"
      );
      if (ratingEntry?.rating) item.ageRating = `FSK ${ratingEntry.rating}`;
    } else {
      // Film FSK-Altersfreigabe
      const relEntry = (details.release_dates?.results || []).find(
        (r) => r.iso_3166_1 === state.region || r.iso_3166_1 === "DE"
      );
      const cert = relEntry?.release_dates?.find((d) => d.certification)?.certification;
      if (cert) item.ageRating = `FSK ${cert}`;
    }

    item._detailsLoaded = true;
  } catch (e) {
    console.warn("Detail Enrichment Failed:", e);
  }
  return item;
}

/* ==========================================================================
   6. UI RENDERING
   ========================================================================== */

function renderAll() {
  document.getElementById("watchlistCounter").textContent = state.watchlistMap.size;
  renderSearchResults();
  renderWatchlist();
}

function parseProviders(providersField) {
  if (Array.isArray(providersField)) return providersField;
  try {
    return JSON.parse(providersField || "[]");
  } catch (_) {
    return [];
  }
}

function createMediaCardHTML(item, isWatchlistTab = false) {
  const savedItem = state.watchlistMap.get(String(item.id));
  const isSaved = Boolean(savedItem);
  const currentStatus = savedItem?.status || null;

  const year = (item.releaseDate || "").slice(0, 4) || "—";
  const typeLabel = item.mediaType === "tv" ? "Serie" : "Film";
  const rating = item.voteAverage ? `★ ${Number(item.voteAverage).toFixed(1)}` : "";
  const posterUrl = item.posterPath
    ? `https://image.tmdb.org/t/p/w342${item.posterPath}`
    : "https://via.placeholder.com/342x513/0E162B/38BDF8?text=WatchMate+TV";

  const providers = parseProviders(savedItem?.streamingProviders || item.streamingProviders);
  const providerPillsHTML =
    providers.length > 0
      ? providers
          .map((slug) => {
            const pObj = SUPPORTED_PROVIDERS.find((p) => p.id === slug);
            const label = pObj ? pObj.label : slug;
            const isSubscribed = state.subscriptions.includes(slug);
            return `<span class="provider-pill ${isSubscribed ? "subscribed" : ""}">
              ${isSubscribed ? "✓ " : ""}${label}
            </span>`;
          })
          .join("")
      : `<span class="provider-pill">Verfügbarkeit prüfen</span>`;

  // Episoden-Tracker für Serien in der Watchlist
  const episodeTrackerHTML =
    isWatchlistTab && item.mediaType === "tv"
      ? `<div class="episode-tracker">
          <span>Fortschritt: <strong class="ep-badge">Staffel ${savedItem.currentSeason || 1} • Episode ${savedItem.currentEpisode || 1}</strong></span>
          <button class="ep-next-btn" data-action="next-ep" data-id="${item.id}">+1 Episode ▶</button>
        </div>`
      : "";

  const actionsHTML = isWatchlistTab
    ? `<div class="card-actions">
        <button class="action-btn ${currentStatus === "WATCHING" ? "btn-watching active" : "btn-watchlist"}"
                data-action="toggle-watching" data-id="${item.id}">
          ${currentStatus === "WATCHING" ? "▶ Schaue ich gerade" : "▶ Jetzt anschauen"}
        </button>
        <button class="action-btn btn-danger-Ghost" data-action="remove" data-id="${item.id}">
          🗑️ Entfernen
        </button>
      </div>`
    : `<div class="card-actions">
        <button class="action-btn btn-watchlist ${currentStatus === "PLAN_TO_WATCH" ? "active" : ""}"
                data-action="plan" data-id="${item.id}">
          ${currentStatus === "PLAN_TO_WATCH" ? "✓ Auf Watchlist" : "+ Auf die Watchlist"}
        </button>
        <button class="action-btn btn-watching ${currentStatus === "WATCHING" ? "active" : ""}"
                data-action="watch-now" data-id="${item.id}">
          ${currentStatus === "WATCHING" ? "▶ Läuft auf TV" : "🍿 Jetzt anschauen"}
        </button>
      </div>`;

  return `
    <article class="media-card ${isSaved ? "in-watchlist" : ""}" data-card-id="${item.id}">
      <div class="media-card-main">
        <div class="poster-wrap">
          <img class="poster-img" src="${posterUrl}" alt="${escapeHtml(item.title)}" loading="lazy" />
          <span class="media-type-badge">${typeLabel}</span>
        </div>
        <div class="media-info">
          <h3 class="media-title">${escapeHtml(item.title)}</h3>
          <div class="media-meta-row">
            <span>${year}</span>
            ${rating ? `<span class="rating-badge">${rating}</span>` : ""}
            ${item.ageRating ? `<span class="age-badge">${escapeHtml(item.ageRating)}</span>` : ""}
            ${item.totalSeasons > 0 ? `<span>• ${item.totalSeasons} Staffeln</span>` : ""}
          </div>
          <p class="media-overview">${escapeHtml(item.overview || "")}</p>
          <div class="providers-row">${providerPillsHTML}</div>
        </div>
      </div>
      ${episodeTrackerHTML}
      ${actionsHTML}
    </article>
  `;
}

function renderSearchResults() {
  const container = document.getElementById("searchResultsGrid");
  if (!state.searchResults.length) {
    container.innerHTML = `
      <div class="empty-state">
        <div class="empty-icon">🔍</div>
        <h3>Keine Treffer gefunden</h3>
        <p>Prüfe den Titel oder hinterlege in den Einstellungen ⚙️ deinen TMDb API-Key für Millionen weitere Titel.</p>
      </div>`;
    return;
  }
  container.innerHTML = state.searchResults.map((item) => createMediaCardHTML(item, false)).join("");
}

function renderWatchlist() {
  const container = document.getElementById("watchlistGrid");
  const emptyEl = document.getElementById("watchlistEmptyState");

  let items = Array.from(state.watchlistMap.values()).sort((a, b) => (b.addedAt || 0) - (a.addedAt || 0));

  if (state.activeFilter !== "ALL") {
    items = items.filter((item) => {
      if (state.activeFilter === "movie" || state.activeFilter === "tv") {
        return item.mediaType === state.activeFilter;
      }
      return item.status === state.activeFilter;
    });
  }

  if (items.length === 0) {
    container.innerHTML = "";
    emptyEl.classList.remove("hidden");
    return;
  }

  emptyEl.classList.add("hidden");
  container.innerHTML = items.map((item) => createMediaCardHTML(item, true)).join("");
}

/* ==========================================================================
   7. HILFSFUNKTIONEN & EVENT-HANDLER
   ========================================================================== */

function updateSyncStatusUI(isOnline, label = "") {
  const dot = document.getElementById("syncStatusDot");
  const text = document.getElementById("syncStatusText");
  if (isOnline) {
    dot.classList.add("online");
    text.textContent = `Mit TV synchronisiert (${label.split("@")[0]})`;
  } else {
    dot.classList.remove("online");
    text.textContent = "Lokaler Modus (Tippe auf 👤 für Cloud-Sync)";
  }
}

function updateAuthModalUI(user) {
  const badge = document.getElementById("authBadge");
  const infoBox = document.getElementById("authUserInfo");
  const signInBtn = document.getElementById("googleSignInBtn");
  const signOutBtn = document.getElementById("googleSignOutBtn");
  const avatar = document.getElementById("userAvatar");
  const defaultIcon = document.getElementById("userIconDefault");

  if (user) {
    badge.textContent = "Verbunden";
    badge.className = "status-pill online";
    infoBox.textContent = `Angemeldet als: ${user.displayName || ""} (${user.email})`;
    infoBox.classList.remove("hidden");
    signInBtn.classList.add("hidden");
    signOutBtn.classList.remove("hidden");

    if (user.photoURL) {
      avatar.src = user.photoURL;
      avatar.classList.remove("hidden");
      defaultIcon.classList.add("hidden");
    }
  } else {
    badge.textContent = "Nicht angemeldet";
    badge.className = "status-pill offline";
    infoBox.classList.add("hidden");
    signInBtn.classList.remove("hidden");
    signOutBtn.classList.add("hidden");
    avatar.classList.add("hidden");
    defaultIcon.classList.remove("hidden");
  }
}

function showToast(message) {
  const container = document.getElementById("toastContainer");
  const toast = document.createElement("div");
  toast.className = "toast";
  toast.textContent = message;
  container.appendChild(toast);

  setTimeout(() => {
    toast.style.opacity = "0";
    toast.style.transition = "opacity 0.25s ease";
    setTimeout(() => toast.remove(), 250);
  }, 2800);
}

function escapeHtml(str) {
  return String(str)
    .replace(/&/g, "&amp;")
    .replace(/</g, "&lt;")
    .replace(/>/g, "&gt;")
    .replace(/"/g, "&quot;");
}

function renderSubscriptionsModalGrid() {
  const grid = document.getElementById("subscriptionsGrid");
  grid.innerHTML = SUPPORTED_PROVIDERS.map((p) => {
    const active = state.subscriptions.includes(p.id);
    return `<button type="button" class="sub-chip ${active ? "active" : ""}" data-sub-id="${p.id}">
      ${active ? "✓ " : ""}${p.label}
    </button>`;
  }).join("");
}

function updateRegionButtonUI() {
  const r = REGIONS.find((reg) => reg.code === state.region) || REGIONS[0];
  document.getElementById("currentRegionFlag").textContent = r.flag;
  document.getElementById("currentRegionCode").textContent = r.code;
}

/* ==========================================================================
   8. INITIALISIERUNG & DOM-EVENTS
   ========================================================================== */

document.addEventListener("DOMContentLoaded", () => {
  applyUrlPairingParams();
  updateRegionButtonUI();
  renderSubscriptionsModalGrid();
  initFirebaseIfConfigured();
  renderAll();

  // 1. Debounced Search (300ms wie im Architekturplan spezifiziert)
  const searchInput = document.getElementById("searchInput");
  const clearBtn = document.getElementById("clearSearchBtn");
  let debounceTimer = null;

  searchInput.addEventListener("input", (e) => {
    const val = e.target.value;
    clearBtn.classList.toggle("hidden", !val);

    if (state.activeTab !== "search") switchTab("search");

    clearTimeout(debounceTimer);
    debounceTimer = setTimeout(() => {
      performTmdbSearch(val);
    }, 300);
  });

  clearBtn.addEventListener("click", () => {
    searchInput.value = "";
    clearBtn.classList.add("hidden");
    performTmdbSearch("");
    searchInput.focus();
  });

  // 2. Tab-Umschaltung
  document.querySelectorAll(".nav-tab").forEach((btn) => {
    btn.addEventListener("click", () => switchTab(btn.dataset.tab));
  });

  function switchTab(tabName) {
    state.activeTab = tabName;
    document.getElementById("tabSearchBtn").classList.toggle("active", tabName === "search");
    document.getElementById("tabWatchlistBtn").classList.toggle("active", tabName === "watchlist");
    document.getElementById("searchView").classList.toggle("hidden", tabName !== "search");
    document.getElementById("watchlistView").classList.toggle("hidden", tabName !== "watchlist");
  }

  // 3. Watchlist-Filter
  document.querySelectorAll(".filter-pill").forEach((pill) => {
    pill.addEventListener("click", () => {
      document.querySelectorAll(".filter-pill").forEach((p) => p.classList.remove("active"));
      pill.classList.add("active");
      state.activeFilter = pill.dataset.filter;
      renderWatchlist();
    });
  });

  // 4. Event-Delegation für Karten-Aktionen (1-Tap Add)
  document.body.addEventListener("click", async (e) => {
    const btn = e.target.closest("[data-action]");
    if (!btn) return;

    const action = btn.dataset.action;
    const id = btn.dataset.id;
    const item =
      state.searchResults.find((x) => String(x.id) === String(id)) ||
      state.watchlistMap.get(String(id));

    if (!item) return;

    if (action === "plan") {
      await addOrUpdateWatchlistItem(item, "PLAN_TO_WATCH");
    } else if (action === "watch-now") {
      await addOrUpdateWatchlistItem(item, "WATCHING");
    } else if (action === "toggle-watching") {
      const nextStatus = item.status === "WATCHING" ? "PLAN_TO_WATCH" : "WATCHING";
      await addOrUpdateWatchlistItem(item, nextStatus);
    } else if (action === "next-ep") {
      await incrementEpisode(id);
    } else if (action === "remove") {
      await removeWatchlistItem(id);
    }
  });

  // 5. Region-Wechsel (DE -> AT -> CH)
  document.getElementById("regionToggleBtn").addEventListener("click", () => {
    const idx = REGIONS.findIndex((r) => r.code === state.region);
    const next = REGIONS[(idx + 1) % REGIONS.length];
    state.region = next.code;
    localStorage.setItem("wm_region", state.region);
    updateRegionButtonUI();
    showToast(`${next.flag} Streaming-Region auf ${next.code} geändert`);
    if (searchInput.value.trim()) performTmdbSearch(searchInput.value);
  });

  // 6. Einstellungs-Modal öffnen / schließen
  const settingsModal = document.getElementById("settingsModal");
  const openSettings = () => {
    document.getElementById("tmdbApiKeyInput").value = state.tmdbKey;
    document.getElementById("firebaseConfigInput").value = state.firebaseConfig
      ? JSON.stringify(state.firebaseConfig, null, 2)
      : "";
    settingsModal.classList.remove("hidden");
  };

  document.getElementById("settingsBtn").addEventListener("click", openSettings);
  document.getElementById("authBtn").addEventListener("click", openSettings);
  document.getElementById("closeSettingsBtn").addEventListener("click", () => {
    settingsModal.classList.add("hidden");
  });

  // Abo-Chips im Modal umschalten
  document.getElementById("subscriptionsGrid").addEventListener("click", (e) => {
    const chip = e.target.closest("[data-sub-id]");
    if (!chip) return;
    const subId = chip.dataset.subId;
    if (state.subscriptions.includes(subId)) {
      state.subscriptions = state.subscriptions.filter((s) => s !== subId);
    } else {
      state.subscriptions.push(subId);
    }
    localStorage.setItem("wm_subs", JSON.stringify(state.subscriptions));
    renderSubscriptionsModalGrid();
    renderAll();
  });

  // Einstellungen speichern
  document.getElementById("saveSettingsBtn").addEventListener("click", () => {
    state.tmdbKey = document.getElementById("tmdbApiKeyInput").value.trim();
    localStorage.setItem("wm_tmdb_key", state.tmdbKey);

    const fbRaw = document.getElementById("firebaseConfigInput").value.trim();
    if (fbRaw) {
      try {
        state.firebaseConfig = JSON.parse(fbRaw);
        localStorage.setItem("wm_firebase_config", JSON.stringify(state.firebaseConfig));
      } catch (err) {
        showToast("⚠️ Firebase Config ist kein gültiges JSON.");
        return;
      }
    }

    settingsModal.classList.add("hidden");
    initFirebaseIfConfigured();
    if (searchInput.value.trim()) performTmdbSearch(searchInput.value);
    showToast("✅ Einstellungen gespeichert!");
  });

  // Google Sign-In / Sign-Out
  document.getElementById("googleSignInBtn").addEventListener("click", async () => {
    if (!state.authInstance) {
      showToast("⚠️ Bitte zuerst unten die Firebase Projekt-Konfiguration eintragen.");
      return;
    }
    try {
      const provider = new GoogleAuthProvider();
      await signInWithPopup(state.authInstance, provider);
      showToast("🎉 Erfolgreich mit Google-Konto verbunden!");
    } catch (err) {
      console.error("Google Sign-In Error:", err);
      showToast(`⚠️ Anmeldung fehlgeschlagen: ${err.message}`);
    }
  });

  document.getElementById("googleSignOutBtn").addEventListener("click", async () => {
    if (state.authInstance) {
      await signOut(state.authInstance);
      showToast("👋 Abgemeldet.");
    }
  });
});

// PWA Service Worker registrieren
if ("serviceWorker" in navigator) {
  window.addEventListener("load", () => {
    navigator.serviceWorker.register("./sw.js").catch(() => {});
  });
}
```

---

## 📱 Datei 4: `web-companion/manifest.json`

```json
{
  "name": "WatchMate TV Companion",
  "short_name": "WatchMate",
  "description": "Filme & Serien von unterwegs direkt auf deinen Android TV senden.",
  "start_url": "./index.html",
  "display": "standalone",
  "background_color": "#060A18",
  "theme_color": "#060A18",
  "orientation": "portrait",
  "icons": [
    {
      "src": "https://cdn-icons-png.flaticon.com/512/4221/4221419.png",
      "sizes": "512x512",
      "type": "image/png",
      "purpose": "any maskable"
    }
  ]
}
```

---

## ⚡ Datei 5: `web-companion/sw.js`

```javascript
const CACHE_NAME = "watchmate-companion-v1";
const ASSETS = ["./", "./index.html", "./styles.css", "./app.js", "./manifest.json"];

self.addEventListener("install", (event) => {
  event.waitUntil(caches.open(CACHE_NAME).then((cache) => cache.addAll(ASSETS)));
  self.skipWaiting();
});

self.addEventListener("activate", (event) => {
  event.waitUntil(
    caches.keys().then((keys) =>
      Promise.all(keys.filter((k) => k !== CACHE_NAME).map((k) => caches.delete(k)))
    )
  );
  self.clients.claim();
});

self.addEventListener("fetch", (event) => {
  if (event.request.method !== "GET" || !event.request.url.startsWith(self.location.origin)) {
    return;
  }
  event.respondWith(
    fetch(event.request).catch(() => caches.match(event.request))
  );
});
```

---

## 🔒 Datei 6: `web-companion/firestore.rules`

```javascript
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {

    // Zero-Trust: Jeder Nutzer darf ausschließlich seine eigene Watchlist lesen & schreiben
    match /users/{userId}/watchlist/{itemId} {
      allow read, delete: if request.auth != null && request.auth.uid == userId;

      allow create, update: if request.auth != null
        && request.auth.uid == userId
        && request.resource.data.keys().hasAll(['id', 'mediaType', 'title', 'status'])
        && request.resource.data.id is int
        && request.resource.data.mediaType in ['movie', 'tv']
        && request.resource.data.status in ['PLAN_TO_WATCH', 'WATCHING', 'COMPLETED']
        && request.resource.data.title is string
        && request.resource.data.title.size() > 0;
    }
  }
}
```

---

## 🔥 Datei 7: `web-companion/firebase.json`

```json
{
  "firestore": {
    "rules": "firestore.rules"
  },
  "hosting": {
    "public": ".",
    "ignore": ["firebase.json", "**/.*", "**/node_modules/**"]
  }
}
```

---