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
