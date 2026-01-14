# Plan: Blade Runner MCP Server

## Vision

Transformer Blade Runner en serveur MCP pour exposer la knowledge base YouTube à Kiro CLI.

**Objectif**: Permettre à Kiro CLI de chercher dans vos transcripts YouTube, accéder aux métadonnées des vidéos, et potentiellement générer des synthèses - le tout via le protocole MCP.

## Architecture Proposée

```
┌─────────────────────────────────────────────────────────────┐
│                        Kiro CLI                              │
│                    (MCP Client)                              │
└────────────────────┬────────────────────────────────────────┘
                     │ stdio (MCP Protocol)
                     │
┌────────────────────▼────────────────────────────────────────┐
│              Blade Runner MCP Server                         │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐  │
│  │  MCP Tools (Exposed Functions)                       │  │
│  │  - search_transcripts(query, max_results)            │  │
│  │  - get_video_info(video_id)                          │  │
│  │  - list_channels()                                   │  │
│  │  - get_channel_videos(channel_id)                    │  │
│  │  - generate_summary(video_ids, format)               │  │
│  └──────────────────────────────────────────────────────┘  │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐  │
│  │  MCP Resources (Data Access)                         │  │
│  │  - transcript://{video_id}                           │  │
│  │  - video://{video_id}                                │  │
│  │  - channel://{channel_id}                            │  │
│  └──────────────────────────────────────────────────────┘  │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐  │
│  │  Existing Blade Runner Services                      │  │
│  │  - knowledge_base.py (SQLite + FTS5)                 │  │
│  │  - youtube_service.py (API + metadata)               │  │
│  │  - ai_service.py (synthèses)                         │  │
│  └──────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

## Phase 1: MCP Server Foundation

### 1.1 Setup MCP Server Structure
- [ ] Créer `mcp_server/` dans Blade Runner
- [ ] Installer `mcp` Python package
- [ ] Créer `server.py` avec structure MCP de base
- [ ] Tester communication stdio avec un client MCP simple

**Fichiers:**
```
blade-runner/
├── mcp_server/
│   ├── __init__.py
│   ├── server.py          # Point d'entrée MCP
│   ├── tools.py           # Définitions des tools
│   ├── resources.py       # Définitions des resources
│   └── config.py          # Configuration MCP
```

### 1.2 Intégration avec Services Existants
- [ ] Importer `knowledge_base.py` dans MCP server
- [ ] Importer `youtube_service.py` pour métadonnées
- [ ] Créer adaptateurs si nécessaire (wrapper léger)

## Phase 2: MCP Tools Implementation

### 2.1 Tool: search_transcripts
**Description**: Recherche full-text dans tous les transcripts

**Paramètres:**
- `query` (string, required): Requête de recherche
- `max_results` (int, optional): Nombre max de résultats (défaut: 10)
- `channel_filter` (string, optional): Filtrer par channel_id

**Retour:**
```json
{
  "results": [
    {
      "video_id": "abc123",
      "title": "Python Tips",
      "channel": "Tech Channel",
      "snippet": "...contexte autour du match...",
      "timestamp": "00:05:23",
      "relevance_score": 0.95
    }
  ],
  "total_found": 42
}
```

### 2.2 Tool: get_video_info
**Description**: Récupère métadonnées complètes d'une vidéo

**Paramètres:**
- `video_id` (string, required): ID YouTube de la vidéo

**Retour:**
```json
{
  "video_id": "abc123",
  "title": "Python Tips",
  "channel_id": "UC...",
  "channel_name": "Tech Channel",
  "published_at": "2025-01-10T12:00:00Z",
  "duration": "PT15M30S",
  "view_count": 10000,
  "has_transcript": true,
  "transcript_language": "en"
}
```

### 2.3 Tool: list_channels
**Description**: Liste toutes les chaînes dans la knowledge base

**Paramètres:**
- `sort_by` (string, optional): "name" | "video_count" | "recent"

**Retour:**
```json
{
  "channels": [
    {
      "channel_id": "UC...",
      "name": "Tech Channel",
      "video_count": 150,
      "last_updated": "2025-01-14T10:00:00Z"
    }
  ]
}
```

### 2.4 Tool: get_channel_videos
**Description**: Liste les vidéos d'une chaîne spécifique

**Paramètres:**
- `channel_id` (string, required): ID de la chaîne
- `limit` (int, optional): Nombre max de vidéos (défaut: 50)
- `has_transcript_only` (bool, optional): Filtrer vidéos avec transcript

**Retour:**
```json
{
  "channel_id": "UC...",
  "channel_name": "Tech Channel",
  "videos": [
    {
      "video_id": "abc123",
      "title": "Python Tips",
      "published_at": "2025-01-10T12:00:00Z",
      "has_transcript": true
    }
  ]
}
```

### 2.5 Tool: generate_summary (Optionnel Phase 3)
**Description**: Génère une synthèse AI de vidéos sélectionnées

**Paramètres:**
- `video_ids` (array[string], required): Liste d'IDs vidéo
- `format` (string, optional): "markdown" | "bullet_points" | "faq"
- `focus` (string, optional): Thème spécifique à extraire

**Retour:**
```json
{
  "summary": "# Synthèse\n\n...",
  "videos_processed": 3,
  "generated_at": "2025-01-14T11:00:00Z"
}
```

## Phase 3: MCP Resources Implementation

### 3.1 Resource: transcript://
**URI Pattern**: `transcript://{video_id}`

**Description**: Accès direct au transcript complet d'une vidéo

**Exemple:**
```
transcript://abc123
```

**Retour:**
```json
{
  "uri": "transcript://abc123",
  "mimeType": "text/plain",
  "text": "Full transcript text with timestamps..."
}
```

### 3.2 Resource: video://
**URI Pattern**: `video://{video_id}`

**Description**: Métadonnées complètes + transcript d'une vidéo

**Exemple:**
```
video://abc123
```

**Retour:**
```json
{
  "uri": "video://abc123",
  "mimeType": "application/json",
  "metadata": { ... },
  "transcript": "..."
}
```

### 3.3 Resource: channel://
**URI Pattern**: `channel://{channel_id}`

**Description**: Info chaîne + liste de vidéos

**Exemple:**
```
channel://UC...
```

## Phase 4: Configuration & Deployment

### 4.1 Configuration Kiro CLI
Créer `.kiro/settings/mcp.json` dans le projet utilisateur:

```json
{
  "mcpServers": {
    "blade-runner": {
      "command": "python",
      "args": [
        "D:/kiro/mcp_server/server.py"
      ],
      "env": {
        "BLADE_RUNNER_DB": "D:/kiro/data/knowledge.db"
      }
    }
  }
}
```

### 4.2 Variables d'Environnement
Le serveur MCP doit pouvoir accéder à:
- `BLADE_RUNNER_DB`: Chemin vers knowledge.db
- `YOUTUBE_CLIENT_ID`: Pour métadonnées (optionnel)
- `YOUTUBE_CLIENT_SECRET`: Pour métadonnées (optionnel)

### 4.3 Documentation Utilisateur
- [ ] README avec instructions d'installation
- [ ] Exemples d'utilisation dans Kiro CLI
- [ ] Troubleshooting guide

## Phase 5: Testing & Polish

### 5.1 Tests Unitaires
- [ ] Tests pour chaque tool
- [ ] Tests pour chaque resource
- [ ] Mock de la DB pour tests rapides

### 5.2 Tests d'Intégration
- [ ] Test communication MCP stdio
- [ ] Test avec Kiro CLI réel
- [ ] Test performance (recherche sur grosse DB)

### 5.3 Error Handling
- [ ] Gestion DB non trouvée
- [ ] Gestion vidéo inexistante
- [ ] Gestion transcript manquant
- [ ] Timeouts et retry logic

## Dépendances Techniques

### Python Packages
```txt
mcp>=0.1.0              # MCP SDK
sqlite3                 # Built-in
pydantic>=2.0           # Validation
```

### Structure de Données (SQLite)
Réutiliser les tables existantes de Blade Runner:
- `transcripts` (video_id, content, timestamps)
- `transcripts_fts` (index FTS5)
- `videos` (metadata)
- `channels` (metadata)

## Cas d'Usage Kiro CLI

### Exemple 1: Recherche Simple
```
User: "Cherche dans mes vidéos YouTube ce qui parle de FastAPI"

Kiro CLI → blade-runner.search_transcripts(query="FastAPI", max_results=5)
→ Retourne 5 vidéos pertinentes avec snippets
```

### Exemple 2: Analyse de Chaîne
```
User: "Quelles sont les dernières vidéos de Tech Channel?"

Kiro CLI → blade-runner.get_channel_videos(channel_id="UC...", limit=10)
→ Liste des 10 dernières vidéos
```

### Exemple 3: Synthèse Multi-Vidéos
```
User: "Fais-moi un résumé des 3 dernières vidéos sur Python"

Kiro CLI → blade-runner.search_transcripts(query="Python", max_results=3)
         → blade-runner.generate_summary(video_ids=[...], format="markdown")
→ Synthèse markdown générée
```

## Prochaines Étapes

1. **Validation du Plan**: Review et ajustements
2. **Setup Initial**: Créer structure `mcp_server/` dans Blade Runner
3. **Proof of Concept**: Implémenter 1 tool simple (search_transcripts)
4. **Test avec Kiro**: Configurer MCP dans Kiro CLI et tester
5. **Itération**: Ajouter tools/resources progressivement

## Questions Ouvertes

- [ ] Faut-il exposer les fonctionnalités AI (synthèses) via MCP ou juste les données brutes?
- [ ] Quelle stratégie de cache pour les recherches fréquentes?
- [ ] Faut-il un mode "read-only" vs "read-write" (ajout de nouvelles vidéos via MCP)?
- [ ] Performance: limite de taille pour les transcripts retournés?

---

**Status**: 🟡 Plan Initial - En Attente de Validation
**Dernière Mise à Jour**: 2026-01-14
