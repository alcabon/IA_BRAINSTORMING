Les cases vertes indiquent le gagnant par critère. La logique centrale reste celle qu'on a dégagée ensemble : **function tooling gagne sur tout ce qui est technique et précision, MCP gagne sur tout ce qui est écosystème et distribution**. Ce sont deux outils qui répondent à deux problèmes différents — pas vraiment des concurrents directs.

<img width="1440" height="3120" alt="image" src="https://github.com/user-attachments/assets/9f83e3af-ee61-4985-b74b-aa9d675f0a09" />


## Oui, c'est exactement le comportement natif du function calling

Le LLM **raisonne sur les outils disponibles** et décide seul de les invoquer. Vous n'avez pas à lui dire "utilise query_salesforce".

---

### Ce qui se passe concrètement

```
User → "Fais-moi une synthèse de nos cases Salesforce du mois"

LLM (interne) → "J'ai besoin de données pour répondre.
                  J'ai un outil query_salesforce disponible.
                  Je vais l'appeler avec les bons paramètres."

LLM → tool_use { 
         name: "query_salesforce",
         input: { 
           object_type: "Case",
           filters: { CreatedDate: "THIS_MONTH" },
           fields: ["Subject", "Status", "Priority", "OwnerId"]
         }
       }

Backend → exécute la requête, retourne les données

LLM → analyse les données et génère la synthèse
```

---

### La clé : la qualité de la description de l'outil

C'est **la description** qui pilote cette décision autonome. Plus elle est précise, plus le LLM sait *quand* et *comment* l'invoquer :

```javascript
{
  name: "query_salesforce",
  description: `Interroge la base Salesforce. 
                Utiliser cet outil dès qu'une question 
                porte sur des Cases, Contacts, Accounts 
                ou Opportunities. Supporte les filtres 
                temporels (THIS_MONTH, LAST_WEEK...) 
                et les regroupements.`,  // ← crucial
  parameters: { ... }
}
```

---

### Mais il y a une nuance importante sur les "deux temps"

Selon la complexité, le LLM peut enchaîner **plusieurs appels autonomes** avant de répondre :

```
"Synthèse des cases + qui sont les clients concernés ?"

  Tour 1 → query_salesforce (Cases du mois)
  Tour 2 → query_salesforce (Accounts liés aux cases)
  Tour 3 → réponse finale synthétisée
```

C'est ce qu'on appelle un **agentic loop** — votre backend doit être conçu pour le gérer, pas seulement un aller-retour unique.

---

### Ce que ça implique côté architecture

```
                    ┌─────────────────┐
                    │   Votre backend  │
                    │                 │
User ──────────────►│  LLM API call   │
                    │       │         │
                    │  tool_use ?─────┼──► Salesforce
                    │       │◄────────┼─── données
                    │  tool_use ?─────┼──► Salesforce (si besoin)
                    │       │◄────────┼─── données
                    │  réponse finale │
User ◄──────────────│                 │
                    └─────────────────┘
```

**Votre backend boucle** tant que `stop_reason === "tool_use"`, et ne retourne la réponse à l'utilisateur que quand le LLM produit du texte final.

---

### En résumé

> Le LLM ne "demande" pas les données — il **décide** de les chercher, formule la requête structurée, et synthétise le tout de façon transparente pour l'utilisateur.

C'est la promesse du function calling : transformer un LLM statique en **agent capable d'aller chercher ce dont il a besoin**.

---

## Oui, et Gemini 2.5 le fait mieux que ses prédécesseurs

Les modèles Gemini 2.5 utilisent un processus de "thinking" interne pour raisonner sur les requêtes, ce qui améliore significativement les performances du function calling — le modèle détermine mieux *quand* appeler une fonction et *quels paramètres* utiliser.

---

### Les modèles 2.5 supportés

Gemini 2.5 Pro, Gemini 2.5 Flash et Gemini 2.5 Flash-Lite supportent tous le function calling.

---

### Ce que ça change concrètement avec 2.5

La couche de **raisonnement interne** avant d'invoquer un outil est la vraie nouveauté :

```
Gemini 2.0  →  prompt → décision → function call
Gemini 2.5  →  prompt → [thinking interne] → décision raisonnée → function call
```

Avec Gemini 2.5 sur votre cas Salesforce, il va naturellement :
- Évaluer si `query_salesforce` est pertinent
- Choisir les bons filtres et champs
- Chaîner plusieurs appels si nécessaire

---

### Une nuance technique spécifique à 2.5

Comme l'API Gemini est stateless, les modèles utilisent des **thought signatures** pour maintenir le contexte entre les tours de conversation multi-étapes. Si vous construisez les requêtes manuellement via REST, vous devez gérer ces signatures — mais les SDKs officiels s'en occupent automatiquement.

Donc si vous utilisez le SDK Python ou JS, vous n'avez rien à gérer de spécial.

---

### En résumé

| | Gemini 2.0 | Gemini 2.5 |
|---|---|---|
| Function calling | ✅ | ✅ |
| Parallel tool calls | ✅ | ✅ |
| Raisonnement avant appel | ❌ | ✅ (thinking) |
| Qualité du choix des paramètres | Bonne | Meilleure |

Pour votre projet Salesforce, **Gemini 2.5 Flash** serait le bon choix — performant sur le function calling, et moins coûteux que 2.5 Pro.

----

## Sécurité et guardrails avec le function calling

Il n'existe pas de mécanisme natif unique — c'est une **combinaison de couches** que vous construisez vous-même.

---

### Couche 1 : Le System Prompt (première ligne de défense)

C'est là que vous définissez les règles d'usage des outils :

```javascript
system: `Tu es un assistant CRM. 
Tu as accès à query_salesforce.

RÈGLES STRICTES :
- Tu ne peux interroger QUE les objets : Case, Account, Contact
- Tu ne peux JAMAIS modifier ou supprimer des données
- Si l'utilisateur demande des données d'un autre utilisateur 
  sans en être le manager, refuse et explique pourquoi
- Ne retourne JAMAIS les champs : SSN, BankAccount, Password__c
- En cas de doute sur la légitimité d'une requête, demande confirmation`
```

---

### Couche 2 : Le schéma de l'outil lui-même (guardrail structurel)

Le schéma **contraint physiquement** ce que le LLM peut demander :

```javascript
{
  name: "query_salesforce",
  description: "Interroge Salesforce en lecture seule",
  parameters: {
    type: "object",
    properties: {
      object_type: {
        type: "string",
        enum: ["Case", "Account", "Contact"],  // ← whitelist stricte
        description: "Seuls ces objets sont autorisés"
      },
      fields: {
        type: "array",
        items: { 
          type: "string",
          enum: [                              // ← champs autorisés uniquement
            "Id", "Subject", "Status", 
            "Priority", "CreatedDate", "OwnerId"
          ]
        }
      },
      filters: {
        type: "object",
        properties: {
          limit: { 
            type: "integer", 
            maximum: 100    // ← pas de dump massif
          }
        }
      }
    },
    required: ["object_type"]
  }
}
```

---

### Couche 3 : Validation côté backend (la plus importante)

**Ne jamais faire confiance au LLM seul.** Votre backend valide tout avant d'exécuter :

```javascript
async function executeTool(toolName, toolInput, userContext) {

  // 1. Whitelist des outils appelables
  const ALLOWED_TOOLS = ["query_salesforce"];
  if (!ALLOWED_TOOLS.includes(toolName)) {
    throw new Error(`Outil non autorisé : ${toolName}`);
  }

  // 2. Validation du payload
  if (toolName === "query_salesforce") {
    
    // Objet autorisé ?
    const ALLOWED_OBJECTS = ["Case", "Account", "Contact"];
    if (!ALLOWED_OBJECTS.includes(toolInput.object_type)) {
      throw new Error("Objet Salesforce non autorisé");
    }

    // Champs sensibles filtrés
    const BLACKLISTED_FIELDS = ["SSN__c", "BankAccount__c", "Password__c"];
    toolInput.fields = toolInput.fields?.filter(
      f => !BLACKLISTED_FIELDS.includes(f)
    );

    // Limit forcée
    toolInput.filters = {
      ...toolInput.filters,
      limit: Math.min(toolInput.filters?.limit || 50, 100)
    };

    // 3. Contrôle d'accès métier (RBAC)
    if (!userContext.permissions.includes(`read:${toolInput.object_type}`)) {
      throw new Error(`Permission refusée pour ${toolInput.object_type}`);
    }
  }

  return await callSalesforce(toolInput);
}
```

---

### Couche 4 : Logging et auditabilité

```javascript
async function trackedToolCall(toolName, toolInput, userContext) {
  const entry = {
    timestamp: new Date().toISOString(),
    userId: userContext.userId,
    tool: toolName,
    input: toolInput,        // ce que le LLM a demandé
    result: null,
    error: null
  };

  try {
    entry.result = await executeTool(toolName, toolInput, userContext);
    return entry.result;
  } catch (err) {
    entry.error = err.message;
    throw err;
  } finally {
    await auditLog.write(entry);  // toujours logué, même en cas d'erreur
  }
}
```

---

### Couche 5 : Rate limiting et détection d'abus

```javascript
// Détection d'agentic loop anormal
const MAX_TOOL_CALLS_PER_REQUEST = 5;
let toolCallCount = 0;

while (response.stop_reason === "tool_use") {
  
  toolCallCount++;
  if (toolCallCount > MAX_TOOL_CALLS_PER_REQUEST) {
    throw new Error("Trop d'appels outils — requête interrompue");
  }

  // ... exécution normale
}
```

---

### Vue d'ensemble des couches

```
User prompt
    │
    ▼
[System Prompt]          ← règles déclaratives pour le LLM
    │
    ▼
[Schéma de l'outil]      ← whitelist structurelle (enum, types, max)
    │
    ▼
[LLM décide]             ← génère le tool_use
    │
    ▼
[Backend validation]     ← RBAC, blacklist champs, sanitization
    │
    ▼
[Audit log]              ← traçabilité complète
    │
    ▼
[Salesforce]             ← avec token scopé en lecture seule
```

---

### Principe fondamental

> Le LLM est **non-déterministe** — il peut toujours produire quelque chose d'inattendu. Le system prompt oriente, le schéma contraint, mais **seul votre backend garantit**.

Ne déléguez jamais la sécurité au LLM seul, quelle que soit la qualité du prompt.

---

## D'abord, correction sur les chiffres : c'est bien plus que 100K

Gemini 2.5 Flash dispose d'une fenêtre de contexte de **1 million de tokens** en entrée. Et Gemini 2.5 Pro atteint également 1 million de tokens, avec un taux de rappel de 100% jusqu'à 530 000 tokens et 99,7% jusqu'à 1 million.

Donc la contrainte est bien moins serrée que vous ne le pensiez.

---

## Mais non — ce n'est pas la seule contrainte réelle

Il y en a plusieurs autres qui comptent vraiment :

### 1. Le coût par token (contrainte économique)

Si vous avez une app où un utilisateur uploade 10 PDFs, une vidéo et des documents, historiquement vous deviez utiliser du RAG pour gérer ces requêtes et payer un montant significatif pour les tokens injectés dans la fenêtre de contexte.

Concrètement pour votre cas Salesforce :

```
1 Case Salesforce textuel ≈ 200-500 tokens
1000 Cases               ≈ 200K-500K tokens
→ Coût à chaque requête  ← c'est là que ça pique
```

Et surtout : dans un contexte multi-tours, vous payez pour **tous les tokens accumulés** des tours précédents à chaque nouveau tour, pas seulement les nouveaux.

---

### 2. La taille de l'output (contrainte souvent oubliée)

Les modèles Pro et 2.5 Pro imposent des caps d'output jusqu'à 64 000 tokens par réponse, tandis que les modèles Flash sont typiquement capés entre 8 000 et 32 000 tokens.

```
Input  : 1M tokens  ✅ pas de problème
Output : 65K tokens ← la synthèse finale est contrainte ici
```

---

### 3. La qualité d'attention sur les très gros volumes

C'est la vraie limite technique — un LLM injecté avec 800K tokens de données va :

```
┌─────────────────────────────────────────┐
│  Début du contexte  → attention forte   │
│  Milieu             → attention faible  │ ← "lost in the middle"
│  Fin du contexte    → attention forte   │
└─────────────────────────────────────────┘
```

Même avec 99,7% de recall annoncé, sur des données métier denses, **le LLM peut rater des patterns non saillants** noyés dans la masse.

---

### 4. La latence (contrainte UX)

```
100K tokens injectés  → réponse en ~5-10s
500K tokens injectés  → réponse en ~30-60s
```

Pour une synthèse interactive, ça devient problématique.

---

### La vraie question architecturale

| Volume de données | Approche recommandée |
|---|---|
| < 50K tokens | **Tout injecter** — simple, efficace |
| 50K - 300K tokens | **Injecter + filtrer** côté backend avant injection |
| > 300K tokens | **RAG ou pré-agrégation** — le function calling récupère déjà des données résumées |

Pour votre cas Salesforce, la bonne architecture est souvent **hybride** :

```javascript
// Au lieu de renvoyer 10 000 cases bruts au LLM :
{
  name: "query_salesforce",
  // Le backend pré-agrège avant de renvoyer
  // → retourne des stats, pas des enregistrements bruts
  returns: {
    total: 1247,
    by_status: { Open: 823, Closed: 312, Pending: 112 },
    by_priority: { High: 89, Medium: 445, Low: 713 },
    top_issues: [ ... ] // 20 cases les plus représentatifs
  }
}
```

Le LLM reçoit alors **de l'information utile**, pas du volume brut — ce qui améliore à la fois la qualité, la vitesse et le coût.

---

## La raison principale : le MCP résout un problème d'**écosystème**, pas de technique

---

### Le problème concret que MCP adresse

Sans MCP, chaque intégration est **bilatérale** :

```
Claude    → fonction Salesforce    (vous codez)
GPT       → fonction Salesforce    (vous recodez)
Gemini    → fonction Salesforce    (vous recodez encore)
Cursor    → fonction Salesforce    (vous recodez encore)
```

Avec MCP, Salesforce écrit **une fois** :

```
Salesforce MCP Server
        ↑
Claude ─┤
GPT    ─┤  ← tous se connectent au même serveur
Gemini ─┤
Cursor ─┘
```

**C'est un protocole de standardisation**, exactement comme HTTP a standardisé le web. Salesforce n'a pas à connaître ou supporter chaque LLM — et inversement.

---

### Pourquoi Salesforce en particulier a adopté MCP

Salesforce a des **milliers d'intégrateurs partenaires**. Sans MCP :

- Chaque partenaire devait coder ses propres function callings
- Schémas différents, comportements différents, maintenance explosive
- Salesforce ne pouvait pas garantir la qualité ni la sécurité

Avec un MCP Server officiel Salesforce :

```
Salesforce contrôle → le schéma
                    → les permissions (OAuth intégré)
                    → les rate limits
                    → les mises à jour d'objets
                    → la documentation exposée au LLM
```

C'est une **décision business et d'écosystème**, pas technique.

---

### Le paradoxe central

```
MCP          = function calling standardisé + protocole de découverte
             = moins précis MAIS universellement connectable

Function     = maximum de contrôle et précision
calling      = mais vous portez tout le poids de l'intégration
```

MCP a gagné la bataille de la **visibilité** parce que :

| Raison | Explication |
|---|---|
| **Anthropic l'a promu** | Créateur du protocole, donc marketing fort |
| **Cursor / Claude Desktop l'ont adopté** | Les outils dev grand public l'ont rendu viral |
| **"Plug and play"** | Un dev peut brancher un MCP server en 5 minutes |
| **Vendeurs SaaS y voient un marché** | Salesforce, Linear, Notion publient leurs serveurs |
| **Presse tech** | "MCP est le USB-C de l'IA" — ça fait des titres |

---

### Ce que la presse ne dit pas

Le function calling direct **reste dominant en production réelle** — il est juste invisible car :

- Il vit dans du code backend propriétaire
- Les entreprises ne communiquent pas sur leur architecture interne
- Il n'y a pas d'écosystème à marketer

```
Ce qu'on voit    → MCP (open source, annonces, conférences)
Ce qui tourne    → function calling (dans 90% des agents en prod)
```

---

### En résumé

> MCP est populaire parce qu'il **résout un problème de distribution et d'écosystème** que le function calling ne peut pas résoudre seul — pas parce qu'il est techniquement supérieur.

Pour un éditeur comme Salesforce, publier un MCP server c'est comme publier une API REST : vous exposez votre produit à tout un écosystème sans dépendre de chaque intégrateur individuel. **C'est une stratégie de plateforme**, pas un choix d'architecture IA.

---

## Bonne question — il faut clarifier ce qui se passe où

Le function calling implique **deux couches distinctes** qu'il ne faut pas confondre.

---

### Ce que le LLM voit : la déclaration JSON

Le LLM ne connaît **jamais** Apex. Il ne voit qu'un schéma JSON :

```json
{
  "name": "query_salesforce",
  "description": "Interroge les données Salesforce",
  "parameters": {
    "type": "object",
    "properties": {
      "object_type": { "type": "string" },
      "filters": { "type": "object" }
    }
  }
}
```

C'est **tout ce que le LLM connaît** — un contrat JSON, langage-agnostique.

---

### Ce que votre backend fait : l'implémentation réelle

```
LLM → tool_use { name: "query_salesforce", input: {...} }
                        │
                        ▼
            [Votre backend : Node / Python / Java]
                        │
            traduit en appel Salesforce
                        │
               ┌────────┴────────┐
               ▼                 ▼
         REST API SF        Apex REST
         (SOQL direct)      (endpoint custom)
```

L'implémentation peut être **n'importe quel langage** côté backend — Apex n'intervient que si vous exposez un endpoint Apex REST sur Salesforce.

---

### Les 3 patterns d'implémentation avec Salesforce

**Pattern 1 — REST API Salesforce standard (le plus simple)**
```javascript
// Votre backend Node.js exécute directement
async function query_salesforce({ object_type, filters }) {
  const soql = buildSOQL(object_type, filters);
  const response = await fetch(
    `${SF_INSTANCE}/services/data/v59.0/query?q=${soql}`,
    { headers: { Authorization: `Bearer ${token}` } }
  );
  return response.json();
}
// Zéro Apex — vous utilisez l'API REST native Salesforce
```

**Pattern 2 — Apex REST (logique métier complexe)**
```apex
// Dans Salesforce, vous exposez un endpoint Apex
@RestResource(urlMapping='/llm-tools/cases/*')
global class LLMCaseService {
  
  @HttpPost
  global static Map<String, Object> getCases(
    String filters,
    List<String> fields
  ) {
    // Logique métier Apex ici
    // Calculs complexes, règles métier, accès multi-objets
    List<Case> cases = Database.query(buildQuery(filters, fields));
    return formatForLLM(cases);
  }
}
```

```javascript
// Votre backend appelle cet endpoint Apex
async function query_salesforce(input) {
  return await fetch(`${SF_INSTANCE}/services/apexrest/llm-tools/cases`, {
    method: 'POST',
    body: JSON.stringify(input)
  });
}
```

**Pattern 3 — Agentforce (MCP natif Salesforce)**
```
Salesforce gère tout en interne
Apex Actions exposées comme tools directement
→ Vous ne codez plus le backend intermédiaire
```

---

### L'interface que vous devez respecter

La seule contrainte réelle c'est **côté déclaration JSON** — que votre fonction retourne quelque chose de parsable :

```javascript
// Contrat implicite de toute fonction tool
async function monTool(input) {        // input = ce que le LLM a envoyé
  // ... votre logique (Apex, REST, SQL, peu importe)
  return {                             // return = ce que le LLM recevra
    success: true,
    data: [...],
    metadata: { count: 42 }
  };
}
```

---

### En résumé

```
Déclaration JSON    → pour le LLM         (langage agnostique)
Backend intermédiaire → Node/Python/Java  (orchestre les appels)
Apex REST           → optionnel           (si logique métier SF complexe)
REST API SF native  → souvent suffisant   (SOQL direct sans Apex)
```

> Apex n'est utile que si vous avez de la **logique métier Salesforce complexe** à encapsuler. Pour des requêtes SOQL simples, l'API REST Salesforce standard suffit largement — pas besoin de toucher à Apex.

----

## Réponse courte : chaque LLM a sa propre spec — mais il existe une convergence

Il n'existe **aucun RFC officiel** (IETF, W3C, ISO) pour le function calling. C'est le far west standardisé.

---

### L'état réel de la standardisation

```
IETF / W3C / ISO   →  ❌  aucun RFC sur le function calling
OpenAI             →  ✅  spec de facto (la première, donc la référence)
Anthropic          →  ✅  spec propre, inspirée d'OpenAI
Google             →  ✅  spec propre, inspirée d'OpenAI
Mistral / Cohere   →  ✅  copient la spec OpenAI quasi à l'identique
```

---

### OpenAI est devenu le standard de facto

OpenAI a été le premier (2023) — tout le monde s'est aligné dessus **sans concertation formelle** :

```javascript
// Format OpenAI — copié par Mistral, Cohere, Together, Groq...
{
  tools: [{
    type: "function",
    function: {
      name: "...",
      description: "...",
      parameters: { /* JSON Schema */ }
    }
  }],
  tool_choice: "auto"
}
```

C'est l'équivalent de "Internet Explorer a défini le web" — pas idéal, mais c'est la réalité.

---

### Ce qui EST standardisé : JSON Schema

La seule partie qui repose sur un **vrai standard** (draft-07 principalement) c'est la définition des paramètres :

```json
{
  "type": "object",
  "properties": {
    "location": {
      "type": "string",
      "description": "..."
    },
    "limit": {
      "type": "integer",
      "minimum": 1,
      "maximum": 100
    }
  },
  "required": ["location"]
}
```

Mais même là, chaque LLM supporte un **sous-ensemble différent** de JSON Schema :

| Feature JSON Schema | OpenAI | Claude | Gemini |
|---|---|---|---|
| `enum` | ✅ | ✅ | ✅ |
| `anyOf` | ✅ | ✅ | ❌ |
| `default` | ✅ | ⚠️ | ❌ |
| `$ref` | ✅ | ❌ | ❌ |
| Nesting profond | ✅ | ✅ | ⚠️ |

---

### Les tentatives de standardisation

**1. OpenAI Compatible API**
La plupart des providers exposent un endpoint `/v1/chat/completions` compatible OpenAI — c'est une standardisation de fait par mimétisme, pas par accord :

```
Groq      → compatible OpenAI
Mistral   → compatible OpenAI
Together  → compatible OpenAI
Gemini    → endpoint compatible OpenAI disponible
Claude    → ❌ non compatible (format trop différent)
```

**2. MCP (Anthropic, 2024)**
Premier vrai effort de standardisation — mais uniquement pour la **couche transport/découverte**, pas pour le format des tools lui-même.

**3. ARC (Agent Runtime Contract)**
Initiative émergente, pas encore aboutie.

---

### Ce que ça implique concrètement pour vous

```javascript
// Si vous voulez être multi-LLM, vous devez abstraire :

class ToolRegistry {
  
  declare(tool) {
    this.tools.push(tool); // format neutre interne
  }

  forAnthropic() {
    return this.tools.map(t => ({
      name: t.name,
      description: t.description,
      input_schema: t.parameters    // ← "input_schema"
    }));
  }

  forGemini() {
    return [{ functionDeclarations: this.tools.map(t => ({
      name: t.name,
      description: t.description,
      parameters: t.parameters      // ← "parameters"
    }))}];
  }

  forOpenAI() {
    return this.tools.map(t => ({
      type: "function",             // ← wrapper supplémentaire
      function: {
        name: t.name,
        description: t.description,
        parameters: t.parameters
      }
    }));
  }
}
```

C'est exactement ce que font LangChain, LlamaIndex ou Instructor — des **adaptateurs** qui masquent cette hétérogénéité.

---

### En résumé

```
Pas de RFC           →  aucune autorité de standardisation
JSON Schema          →  seul vrai standard sous-jacent, mais partiellement supporté
OpenAI               →  standard de facto par adoption massive
MCP                  →  standardise la découverte, pas le format
Résultat             →  vous codez soit pour un LLM, soit vous abstraire
```

> C'est un domaine trop jeune et trop compétitif pour qu'un RFC émerge — chaque acteur a intérêt à ce que son format devienne le standard, pas à céder à un tiers. La situation ressemble aux débuts des navigateurs web avant les standards W3C.

----

## Exactement — vous avez parfaitement cerné la dynamique

MCP est né **précisément** de ce chaos de specs incompatibles. C'est sa raison d'être fondamentale.

---

### Le raisonnement d'Anthropic en créant MCP

```
Problème observé :
  - N providers LLM × M intégrations = N×M implémentations
  - Chaque combinaison = code custom + maintenance
  - Aucune portabilité

Solution MCP :
  - 1 protocole standard de découverte et transport
  - N providers + M intégrations = N+M implémentations
  - Portabilité totale
```

C'est le **même raisonnement** qu'USB face aux connecteurs propriétaires, ou HTTP face aux protocoles réseau propriétaires.

---

### Le compromis fondamental assumé

```
Function Calling direct          MCP
─────────────────────────────    ──────────────────────────────
Vous définissez le schéma   →    Le serveur MCP définit le schéma
Précision maximale          →    Précision déléguée
Couplage fort               →    Couplage faible
1 LLM, 1 intégration        →    N LLMs, 1 intégration
Pas de découverte           →    Découverte dynamique des outils
Votre responsabilité        →    Responsabilité partagée
```

MCP a **consciemment sacrifié** la précision pour gagner l'interopérabilité. C'est un choix d'ingénierie délibéré, pas un accident.

---

### Où se situe exactement la perte de précision

**1. La description des outils — vous n'en contrôlez plus la qualité**

```javascript
// Function calling direct — vous maîtrisez chaque mot
{
  description: `Interroge les Cases Salesforce.
    UTILISER quand : synthèse, stats, recherche de cases.
    NE PAS UTILISER pour : modifier, créer, supprimer.
    Filtres temporels supportés : THIS_MONTH, LAST_WEEK, TODAY.
    Toujours inclure Status et Priority dans les fields.`
}

// MCP — vous subissez la description du serveur tiers
{
  description: "Query Salesforce objects"  // ← souvent trop vague
}
```

**2. Le schéma des paramètres — contraint par le protocole MCP**

```javascript
// Function calling — liberté totale de JSON Schema
parameters: {
  filters: {
    type: "object",
    properties: {
      dateRange: { enum: ["TODAY", "THIS_WEEK", "THIS_MONTH"] },
      maxRecords: { type: "integer", maximum: 100 }
    }
  }
}

// MCP — subset plus limité, moins expressif
// Les types complexes imbriqués passent moins bien
```

**3. Le contexte d'invocation — vous ne pouvez plus guider finement**

```javascript
// Function calling — vous pouvez forcer un outil précis
tool_choice: { type: "tool", name: "query_salesforce" }

// MCP — le LLM choisit parmi les outils découverts
// Moins de contrôle sur QUEL outil est invoqué
```

---

### La hiérarchie des niveaux d'abstraction

```
Niveau 0 : Prompt engineering pur
           "Retourne un JSON avec {object, filters}"
           → Maximum de contrôle, zéro structure garantie

Niveau 1 : Function Calling direct
           Schéma JSON précis, implémentation custom
           → Précision maximale, portabilité nulle

Niveau 2 : MCP
           Protocole standardisé, découverte dynamique
           → Précision moyenne, portabilité totale

Niveau 3 : Agents autonomes (LangChain, AutoGPT...)
           Le LLM décide de tout, outils dynamiques
           → Précision faible, flexibilité maximale
```

Chaque niveau gagne en portabilité et flexibilité en échangeant de la précision et du contrôle.

---

### Ce que l'industrie n'a pas encore résolu

Le vrai problème non résolu c'est qu'**il manque une couche intermédiaire** :

```
Idéal non existant :

  Standard RFC ouvert        →  interopérabilité MCP
+ Schéma expressif précis   →  précision function calling
+ Découverte dynamique      →  flexibilité MCP
+ Contrôle fin d'invocation →  puissance function calling

= n'existe pas encore
```

MCP v1 a résolu l'interopérabilité. La précision reste le chantier ouvert — c'est probablement ce que MCP v2 ou un concurrent tentera d'adresser.

---

> C'est le compromis classique en informatique : **les standards gagnent par adoption, pas par supériorité technique**. MCP a gagné la guerre de l'écosystème en perdant des batailles techniques que le function calling direct remporte encore largement.
>
---

## Oui, sans hésitation — et voici le raisonnement complet

---

### Le contexte qui rend ce conseil valide

```
Votre situation probable :
  ✅ LLM fixé (Gemini 2.5 ou Claude)
  ✅ Données métier connues (Salesforce)
  ✅ Schémas maîtrisés (objets SF standards + custom)
  ✅ Backend sous votre contrôle
  ✅ Cas d'usage précis (synthèse, analyse, reporting)
```

Dans ce contexte, adopter MCP serait ajouter de la complexité **sans bénéfice réel**.

---

### La progression naturelle que je recommanderais

**Phase 1 — Function calling direct, schémas minimalistes**

Commencez simple, un seul outil :

```javascript
// Une seule fonction bien décrite vaut mieux
// que cinq fonctions mal définies
{
  name: "query_salesforce",
  description: `...description précise et riche...`,
  parameters: { ...schéma strict et testé... }
}
```

Validez que le LLM l'invoque correctement, avec les bons paramètres, dans les bons cas.

**Phase 2 — Affinage itératif des descriptions**

C'est là que se gagne la vraie précision :

```
Itération 1 → le LLM appelle l'outil trop souvent
              → affiner : "N'utiliser QUE si..."

Itération 2 → le LLM choisit de mauvais filtres
              → affiner : "Pour les dates, toujours utiliser..."

Itération 3 → le LLM demande trop de champs
              → affiner : "Champs minimaux suffisants : ..."
```

**Chaque itération** améliore la précision d'une façon qu'aucun MCP tiers ne pourra jamais faire pour vous.

**Phase 3 — Enrichissement progressif des outils**

Seulement quand Phase 1 est stable :

```javascript
// On ajoute des outils seulement quand le besoin est prouvé
query_salesforce       // ← d'abord ça
get_case_timeline      // ← seulement si besoin avéré
aggregate_by_owner     // ← idem
```

**Phase 4 — MCP uniquement si vous devez ouvrir à l'écosystème**

```
Trigger légitime pour passer à MCP :
  → Plusieurs LLMs différents à supporter
  → Des partenaires externes qui veulent se connecter
  → Un produit plateforme que d'autres vont intégrer
  → Une équipe distribuée qui ne peut pas coordonner les schémas
```

---

### Le vrai avantage concurrentiel du function calling maîtrisé

```
MCP Salesforce officiel     →  tout le monde y a accès
                                même description pour tous
                                même précision (moyenne) pour tous

Votre function calling      →  schéma taillé pour VOS cas d'usage
custom                         description optimisée pour VOTRE LLM
                                précision que personne d'autre n'a
                                = avantage compétitif réel
```

C'est exactement comme la différence entre une API générique publique et un index de base de données taillé pour vos requêtes exactes.

---

### Le seul vrai risque de ce conseil

Si vous changez de LLM, vous devrez adapter les déclarations — mais c'est :

```javascript
// Une heure de travail maximum
forAnthropic() { ... }   // adapter input_schema
forGemini()    { ... }   // adapter functionDeclarations
forOpenAI()    { ... }   // adapter le wrapper type:function
```

La **logique métier et les schémas** ne changent pas — seule l'enveloppe syntaxique change.

---

### En une phrase

> Commencez par le function calling direct, traitez les descriptions comme du **code de production** à itérer et versionner, et ne migrez vers MCP que le jour où votre problème devient un problème d'**écosystème** — pas avant.
