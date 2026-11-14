# TokenMesh

Reverse-proxy léger pour LLM : cache en mémoire, résilience (retry automatique), et une API FastAPI compatible avec un format de messages type OpenAI.


## Fonctionnalités

- **Cache en mémoire** — deux appels identiques (même modèle + mêmes messages) ne déclenchent qu'un seul appel réel au provider ; le second est servi depuis le cache (`cached: true`).
- **Résilience** — retry automatique avec backoff exponentiel (3 tentatives max) sur les erreurs transitoires : timeout, rate limit, service indisponible, erreur de connexion.
- **Deux endpoints de chat** — un endpoint simple à base de `prompt`, un endpoint avancé à base de `messages` (style OpenAI, multi-tour, avec `system`/`user`/`assistant`).
- **Indépendant du provider** grâce à [LiteLLM](https://github.com/BerriAI/litellm) — le modèle par défaut est Groq (`groq/llama-3.3-70b-versatile`) mais n'importe quel provider supporté par LiteLLM peut être utilisé en changeant le champ `model`.

## Installation

```bash
pip install -r requirements.txt
```

Crée un fichier `.env` à la racine avec ta clé API :

```
GROQ_API_KEY=ta_cle_ici
```

## Lancer le serveur

```bash
uvicorn main:app --reload
```

L'API est accessible sur `http://localhost:8000` et la documentation interactive (Swagger) sur `http://localhost:8000/docs`.

## Endpoints

| Méthode | Route | Description |
|---|---|---|
| `POST` | `/chat` | Chat simple à partir d'un `prompt` unique |
| `POST` | `/v1/chat` | Chat multi-tour à partir d'une liste de `messages` |
| `GET` | `/health` | Vérifie que le serveur répond |
| `GET` | `/cache/stats` | Nombre d'entrées en cache et clés |
| `DELETE` | `/cache` | Vide le cache |

### Exemple — `/chat`

```bash
curl -X POST http://localhost:8000/chat \
  -H "Content-Type: application/json" \
  -d '{"prompt": "Dis bonjour en une phrase"}'
```

```json
{"response": "Bonjour ! Comment puis-je t'aider aujourd'hui ?", "cached": false}
```

### Exemple — `/v1/chat`

```bash
curl -X POST http://localhost:8000/v1/chat \
  -H "Content-Type: application/json" \
  -d '{
    "messages": [
      {"role": "system", "content": "Tu réponds toujours en une phrase courte."},
      {"role": "user", "content": "Explique ce qu'\''est un cache LLM."}
    ]
  }'
```

```json
{"response": "Un cache LLM stocke les réponses déjà générées pour éviter de réappeler le provider.", "cached": false}
```

Le champ `model` est optionnel sur les deux endpoints (par défaut `groq/llama-3.3-70b-versatile`).

Le guide complet de test pas-à-pas (avec tous les cas, y compris les erreurs et le retry) est dans [`TESTING.md`](./TESTING.md).

## Structure du projet

```
.
├── main.py             # API FastAPI (cache, retry, endpoints)
├── step1.py            # Script de fondation — premier appel LLM (étape 1 de la roadmap)
├── requirements.txt
├── .env                # clé API (non versionné)
└── .gitignore
```

## Limites connues

- Le cache est **en mémoire uniquement** : il est perdu à chaque redémarrage du serveur, et ne se partage pas entre plusieurs instances/workers.
- Pas de TTL (durée de vie) ni de limite de taille sur le cache — il grossit indéfiniment tant que le serveur tourne.
- `/cache/stats` expose les clés brutes (JSON des messages) — pratique en dev, à ne pas exposer publiquement en production.
- Pas d'authentification sur l'API elle-même (à ajouter si déployé au-delà d'un usage local/démo).

## Roadmap

1. ✅ Fondations — premier appel LLM (`step1.py`)
2. ✅ API FastAPI
3. ✅ Cache simple
4. ✅ Résilience (retry)
5. ✅ Finitions — logs + README
