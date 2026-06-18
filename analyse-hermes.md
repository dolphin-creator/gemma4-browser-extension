# Analyse technique — gemma4-browser-extension

> Analyse réalisée par Hermes Agent — 18 juin 2026

---

## Vue d'ensemble

**~3 900 lignes TypeScript/React** — extension Chrome MV3 qui fait tourner un LLM (Gemma 4) entièrement en local via **Transformers.js + WebGPU**. Agent conversationnel capable de contrôler le navigateur.

**Repo :** [dolphin-creator/gemma4-browser-extension](https://github.com/dolphin-creator/gemma4-browser-extension) (fork de `nico-martin/gemma4-browser-extension`)
**Version :** 0.2.1

---

## Architecture — 3 couches

| Couche | Rôle | Lignes |
|--------|------|--------|
| **Background** (`service_worker`) | Cerveau : models, inference, agent loop, outils | ~1 400 |
| **Side Panel** (React) | UI chat + streaming + settings | ~1 200 |
| **Content Script** | Extraction DOM + highlighting | ~150 |

Architecture classique MV3, bien séparée. Le background est le seul à charger les models — bonne décision pour éviter le reload à chaque tab.

---

## Moteur d'Agent (`Agent.ts`)

Le cœur du système. **Boucle agent classique** avec tool-calling :

### `generateText()`
Inference via Transformers.js avec :
- `DynamicCache` pour le KV caching (contexte persistant entre turns)
- `TextStreamer` pour le streaming token par token vers l'UI
- Metrics complètes (tok/s, prefill/decode times)

### `runAgent()`
Boucle principale :
1. Génère le texte → parse les tool calls
2. Exécute les outils en parallèle (`Promise.all`)
3. Ré-injecte les réponses → boucle jusqu'à réponse finale
4. **Streaming en temps réel** vers l'UI pendant l'inference

### Tool calling — supporte 3 formats
1. Standard JSON (`` ` ``)
2. Gemma natif (`<|tool_call|>call:...<tool_call|>`)
3. Gemma "bare" (fallback sans delimiters — parsing manuel de `{}`)

**Points forts :**
- KV caching = contexte conversationnel persistant
- Streaming = feedback immédiat
- Metrics exposées = debug possible
- Robuste sur les formats de tool call (3 fallbacks)

**Points faibles :**
- `strictNullChecks: false` dans tsconfig — risque de bugs silencieux
- Pas de limite de boucle sur `runAgent` — boucle infinie possible si le modèle loop sur les tool calls
- `while (prompt !== null)` sans guard de profondeur max

---

## Outils disponibles (7 tools)

| Tool | Rôle | Qualité |
|------|------|---------|
| `get_open_tabs` | Liste tous les tabs avec meta | ✓ Simple, propre |
| `go_to_tab` | Change de tab | ✓ |
| `open_url` | Ouvre une URL | ✓ |
| `close_tab` | Ferme un tab | ✓ |
| `find_history` | **Recherche sémantique** dans l'historique IndexedDB | ⭐ Impressionnant |
| `ask_website` | **RAG sur la page actuelle** via embeddings + cosine similarity | ⭐ Feature phat |
| `highlight_website_element` | Survole + highlight un élément | ✓ |

`google_search` existe dans le code mais est **commenté** — ouvrirait juste un tab Google.

---

## RAG / Embeddings

### FeatureExtractor
Wrapper autour de `all-MiniLM-L6-v2` (384 dims) :
- `extractFeatures()` → embeddings normalisés, mean pooling
- Utilisé pour `ask_website` (RAG page) et `find_history` (recherche historique)

### VectorHistory
Base de données vectorielle IndexedDB :
- Stocke embeddings de title + description + URL pour chaque page visitée
- Recherche par similarité cosinus max sur les 3 champs
- Filtre temporel via index `time` (ISO 8601)
- **Auto-indexage** : chaque `chrome.tabs.onUpdated` ajoute la page

### WebsiteContentManager (pour `ask_website`)
- Extrait `h1-h6` + `p` de la page via content script
- Génère embeddings par phrase
- Recherche top-K par max cosine similarity
- ID structuré (`section-paragraph`) pour le highlighting

---

## Communication (IPC)

Tout passe par `chrome.runtime.sendMessage` avec enums typés :

```
Side Panel → Background : CHECK_MODELS, INITIALIZE_MODELS, AGENT_*
Background → Side Panel : DOWNLOAD_PROGRESS, MESSAGES_UPDATE
Background → Content    : EXTRACT_PAGE_DATA, HIGHLIGHT_ELEMENTS
```

Design contract bien défini dans `shared/types.ts`.

---

## UI (Side Panel)

React 19 + TailwindCSS 4 + Vite 7 :
- **App** — gestion du cycle de vie : IDLE → CHECKING → DOWNLOAD → READY
- **Chat** — affichage des messages avec streaming, tool calls, metrics
- **SettingsHeader** — config des outils + model selection
- **Theme** — composants réutilisables (Button, Card, Modal, Input, etc.)

Propre, responsive, dark/light aware.

---

## Build

- **Vite** avec plugins custom pour :
  - Inline du content script (pas d'import dynamique autorisé en MV3)
  - Déplacement de `sidebar.html` au root de `dist/`
- `vite-plugin-web-extension` **désactivé** (commenté) — build manuel
- Models exclus de l'optimisation (`optimizeDeps.exclude`)

---

## Modèles supportés

| ID | Modèle | Taille | Dtype |
|----|--------|--------|-------|
| `gemma4E2B` | Gemma 4 E2B-it | ~2B params | q4f16 |
| `gemma4E4B` | Gemma 4 E4B-it | ~4B params | q4f16 |
| `granite350m` | Granite 4.0 350M | 350M | fp16 |
| `granite1B` | Granite 4.0 1B | 1B | q4 |
| `granite3B` | Granite 4.0 3B | 3B | q4f16 |
| `allMiniLM` | all-MiniLM-L6-v2 | 22M | fp32 |

Par défaut : **Gemma 4 E2B** (léger, rapide, q4f16). Switchable via l'UI.

---

## Points forts

1. **100% local** — zéro data envoyée, inference WebGPU
2. **RAG intégré** — recherche sémantique sur page ET historique
3. **Streaming** — feedback en temps réel
4. **KV caching** — contexte persistant
5. **Architecture propre** — séparation claire background/UI/content
6. **Outils bien conçus** — schema JSON, validation, error handling
7. **Multi-modèle** — 5 LLMs disponibles, switchable
8. **Metrics exposées** — perf d'inference visibles

## Points faibles

1. **Pas de limite de boucle agent** — risque de boucle infinie
2. **`strictNullChecks: false`** — risque TS
3. **Pas de `google_search` actif** — implémenté mais désactivé
4. **Content script inline hack** — regex fragile sur le build output
5. **Pas de persistance du chat** — lost on refresh
6. **Pas de rate limiting** sur `extractFeatures` — peut spam le GPU
7. **CSP `wasm-unsafe-eval`** — nécessaire pour WebGPU mais sécurité réduite

---

## Verdict

C'est une **demo impressionnante** — un vrai agent LLM qui tourne en local dans le navigateur. Le code est propre et bien structuré. Les features RAG (embeddings + cosine similarity) sont particulièrement bien implémentées.

### Axes d'amélioration prioritaires

1. Limite de boucle sur `runAgent` (sécurité)
2. Activer `google_search` + l'améliorer (parse les résultats)
3. Persistance du chat (localStorage)
4. `strictNullChecks: true` + fix des warnings
5. Plus d'outils : cliquer, remplir formulaires, scroll
6. Rate limiting sur les embeddings

---

*Document généré par Hermes Agent — forking et analyse automatisés*
