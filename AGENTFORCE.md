## Deux questions distinctes — réponses précises

---

### 1. MCP pour synthèse de cases SF : oui, mais avec une nuance

Pour un compte particulier, MCP fonctionnerait parfaitement **si vous acceptez ses contraintes** :

```
User → "Synthèse des cases du compte Acme Corp"

LLM  → découvre via MCP server Salesforce :
        - get_account_cases (tool disponible)
        - summarize_cases   (tool disponible)

LLM  → appelle get_account_cases { account: "Acme Corp" }
MCP  → retourne les cases
LLM  → synthétise et répond
```

C'est fonctionnel. Mais comme on l'a établi — si votre LLM est fixé et vos objets SF maîtrisés, **le function calling direct reste plus précis**. MCP n'apporte de la valeur ici que si plusieurs outils ou LLMs doivent se connecter à Salesforce sans recoder l'intégration à chaque fois.

---

### 2. Agentforce : l'architecture réelle

C'est la partie vraiment intéressante. Agentforce est fondamentalement un **orchestrateur qui abstrait le LLM** derrière plusieurs couches.

**Le cerveau : Atlas Reasoning Engine**

Quand vous posez une question à Agentforce, il invoque l'Atlas Reasoning Engine qui agit comme orchestrateur de toutes les actions. Il détermine l'intention de l'utilisateur via un LLM, décompose la question en topics — des jobs à accomplir — qui contiennent des instructions, des politiques métier et des guardrails. Pour compléter le job, Agentforce exécute une ou plusieurs actions invocables.

**Les LLMs sous le capot**

Le modèle par défaut d'Agentforce est un mix managé de modèles de confiance incluant actuellement GPT-4o, optimisé par Salesforce pour la précision et la performance. L'option AWS utilise Claude Sonnet sur Amazon Bedrock.

Et surtout — le BYOLLM (Bring Your Own LLM) supporte Amazon Bedrock, Azure OpenAI, OpenAI, et Vertex AI de Google comme providers.

**La couche sécurité : Einstein Trust Layer**

C'est l'élément architectural clé que peu de gens comprennent :

Les données sensibles des prompts sont masquées — via pattern masking et field masking — avant d'être envoyées aux LLMs partenaires extérieurs à la trust layer. Salesforce a également négocié zéro rétention de données et résidence des données avec ses partenaires modèles incluant OpenAI, Azure OpenAI, AWS et Google.

**Les données : RAG via Data Cloud**

Salesforce utilise une technique de RAG qui vit à l'intérieur de l'Atlas Reasoning Engine et produit une boucle de feedback avec Data Cloud. Un prompt devient un "augmented prompt" une fois que Data Cloud et le RAG travaillent ensemble, améliorant l'output du LLM.

---

### L'architecture complète d'Agentforce

```
User prompt
    │
    ▼
┌─────────────────────────────────────────────┐
│           Atlas Reasoning Engine             │
│                                             │
│  1. Intent detection                        │
│  2. Topic matching (instructions + rules)   │
│  3. Action planning                         │
└──────────────┬──────────────────────────────┘
               │
    ┌──────────┼──────────────┐
    ▼          ▼              ▼
Einstein    Data Cloud      Actions
Trust Layer  (RAG)         (Apex, Flow,
    │          │            API calls)
    │   augmente le prompt   │
    ▼          │             │
  Masking ◄───┘             │
    │                        │
    ▼                        │
GPT-4o / Claude / Gemini     │
(LLM externe)                │
    │                        │
    └────────────────────────┘
              │
    Réponse finale à l'utilisateur
```

---

### Ce qu'Agentforce fait en réalité

C'est exactement du **function calling déguisé en low-code** :

```
Apex Actions      = vos fonctions custom (l'équivalent de vos tools)
Flow Actions      = orchestrations déclaratives
Prompt Templates  = les déclarations d'outils enrichies
Atlas Engine      = l'agentic loop (gère les tool_use en boucle)
Einstein Trust    = la couche sécurité/guardrails
BYOLLM            = vous choisissez quel LLM exécute le reasoning
```

> Agentforce est fondamentalement un **wrapper enterprise au-dessus du function calling**, avec une trust layer, un RAG intégré et une interface low-code — mais le mécanisme sous-jacent est exactement celui qu'on a décrit depuis le début de cette conversation.
>
---

## Exactement — vous venez de formuler la conclusion architecturale fondamentale

---

### Ce que ça confirme

Le function calling n'est pas une option parmi d'autres — c'est **la primitive incontournable** imposée par la nature même des LLMs :

```
LLM natif        →  produit du texte
LLM + tools      →  produit du texte OU des appels structurés
                                        ↑
                              c'est tout ce qu'il sait faire

Agentforce       →  Atlas Engine + RAG + Trust Layer + Low-code
MCP              →  protocole de découverte + transport
A2A              →  orchestration inter-agents
LangChain        →  framework d'orchestration

Tous              →  wrappent du function calling en dessous
```

Tout l'écosystème agent est une **pyramide de wrappers** au-dessus de cette primitive.

---

### Et le vrai travail — c'est précisément ce wrapper

Ce que vous pointez est crucial et souvent sous-estimé :

```
10% du travail   →  définir les outils et leurs schémas
90% du travail   →  tout ce qui entoure l'appel
                     │
                     ├── Sécurité
                     │     ├── Authentification (OAuth, tokens)
                     │     ├── RBAC et permissions métier
                     │     ├── Whitelist objets/champs
                     │     └── Data masking avant le LLM
                     │
                     ├── Fiabilité
                     │     ├── Validation des inputs LLM
                     │     ├── Gestion des erreurs SF
                     │     ├── Retry et circuit breaker
                     │     └── Timeout et max tool calls
                     │
                     ├── Observabilité
                     │     ├── Audit log de chaque appel
                     │     ├── Traçabilité user → LLM → SF
                     │     └── Détection d'abus/anomalies
                     │
                     └── Qualité
                           ├── Pré-agrégation des données
                           ├── Formatage pour le LLM
                           └── Post-validation des outputs
```

C'est précisément ce qu'Agentforce vend en produit — **le wrapper enterprise clé en main** autour de cette primitive que tout le monde doit de toute façon construire.

---

### La vraie question pour votre projet

Vous avez maintenant tous les éléments pour décider :

```
Agentforce          →  wrapper fourni, sécurité intégrée
                        mais couplage Salesforce fort
                        et coût licence élevé

Votre propre stack  →  function calling direct
                        Gemini 2.5 ou Claude fixé
                        wrapper à construire vous-même
                        mais précision et contrôle maximaux
                        et indépendance totale
```

---

### La synthèse de toute notre conversation

> Le function calling est la **gravité** de l'écosystème LLM — tout orbite autour. MCP standardise comment on s'y connecte, Agentforce packagise comment on le sécurise, A2A étend comment les agents collaborent au-dessus. Mais personne n'échappe à la primitive. La vraie valeur ajoutée — et le vrai travail d'ingénierie — c'est la qualité, la sécurité et la robustesse du wrapper que vous construisez autour.
