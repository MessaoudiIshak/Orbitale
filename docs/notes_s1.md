# Notes TP S1 – Fondamentaux REST

## Étape 1 – Anatomie d'un échange

Commande : `curl.exe -v https://api.github.com/repos/nodejs/node -o NUL`

### 1. Lignes de requête et de statut

- **Ligne de requête** : `GET /repos/nodejs/node HTTP/1.1`
  - méthode : `GET`
  - cible : `/repos/nodejs/node`
  - version : `HTTP/1.1`
- **Ligne de statut** : `HTTP/1.1 200 OK`
  - version : `HTTP/1.1`
  - code : `200`
  - message : `OK`
- **Version HTTP utilisée** : HTTP/1.1. Le client a proposé uniquement http/1.1 lors de la négociation TLS (`ALPN: curl offers http/1.1`) et le serveur l'a accepté (`ALPN: server accepted http/1.1`).

### 2. Quatre en-têtes de réponse

| En-tête | Valeur | Rôle |
|---|---|---|
| Content-Type | application/json; charset=utf-8 | format du corps (JSON) et encodage des caractères (UTF-8) |
| Cache-Control | public, max-age=60, s-maxage=60 | réponse cachable par tous pendant 60 s (`s-maxage` = caches partagés : proxy, CDN) |
| Date | Fri, 20 Nov 2026 13:04:21 GMT | date de génération de la réponse ; point de départ du calcul de `max-age` |
| X-XSS-Protection | 0 | désactive l'ancien filtre XSS des navigateurs (obsolète et lui-même exploitable) ; la protection passe par `Content-Security-Policy: default-src 'none'` |

### 3. x-ratelimit-remaining

- **Signification** : nombre de requêtes encore autorisées dans la fenêtre en cours. Sans authentification, GitHub autorise 60 requêtes par heure et par IP.
- **Valeurs observées** : `X-RateLimit-Limit: 60`, `X-RateLimit-Remaining: 57` → 3 requêtes déjà consommées.
- **Remise à zéro** : `X-RateLimit-Reset: 1795182996` (timestamp Unix), soit le 20/11/2026 à 13:56:36 GMT.
- **À 0** : GitHub refuse les requêtes avec un statut `403` (ou `429`) « API rate limit exceeded », jusqu'à l'heure de `X-RateLimit-Reset`. Dans une salle partagée (même IP publique pour tous), la limite est vite atteinte. Solution : s'authentifier avec un jeton (`Authorization: Bearer ...`), ce qui donne une limite plus haute.

## Étape 2 – Le cache conditionnel

On renvoie la requête de l'étape 1 avec l'en-tête `If-None-Match` contenant l'ETag reçu.
Sous PowerShell 5.1, les guillemets de l'ETag doivent être échappés en `\"`, sinon ils sont supprimés avant d'arriver à curl.

**Requête 1 : avec l'ETag de l'étape 1 (13:04)**
```powershell
PS> curl.exe -s -o NUL -w "%{http_code}\n" -H 'If-None-Match: W/\"b9ddeb84d4c4d3a34666618680bb9c3557578a2ea12cddde833171a04e253b9c\"' https://api.github.com/repos/nodejs/node
200
```
→ `200 OK` : le dépôt a changé depuis l'étape 1, l'ETag n'est plus valable, GitHub renvoie la nouvelle version complète.

**Nouvel ETag récupéré** (relance de `curl.exe -v https://api.github.com/repos/nodejs/node -o NUL`) :
```
< ETag: W/"c082118be2ae7eaeaefe34e86c43143d2c6f558571cdb3402415a9a35a027a95"
```

**Requête 2 : avec l'ETag actuel**
```powershell
PS> curl.exe -s -o NUL -w "%{http_code}\n" -H 'If-None-Match: W/\"c082118be2ae7eaeaefe34e86c43143d2c6f558571cdb3402415a9a35a027a95\"' https://api.github.com/repos/nodejs/node
304
```
→ `304 Not Modified` : la version du client est toujours à jour.

### 1. Statut obtenu et corps vide
- Statut : `304 Not Modified`.
- Le corps est vide car l'ETag envoyé dans `If-None-Match` correspond à la version courante : le serveur n'a rien de nouveau à envoyer, le client réutilise sa copie en cache.

### 2. Intérêt
- **Application mobile** : aucune donnée téléchargée si rien n'a changé (0 octet de corps au lieu de 6618), donc économie de forfait et de batterie, et réponse plus rapide.
- **Quota GitHub** : selon la documentation, une réponse 304 ne compte pas dans le quota **si la requête est authentifiée** (jeton). Sans jeton, on a observé que `X-RateLimit-Remaining` baisse quand même (50 → 49 après un 304). Le gain est donc sur la bande passante, pas sur le quota.

## Étape 3 – Les erreurs

**Requête 1 : dépôt GitHub inexistant**
```powershell
PS> curl.exe -s https://api.github.com/repos/nodejs/ce-depot-n-existe-pas -w "\n%{http_code}\n"
{
  "message": "Not Found",
  "documentation_url": "https://docs.github.com/rest/repos/repos#get-a-repository",
  "status": "404"
}
404
```

**Requête 2 : suppression d'un dépôt GitHub sans authentification**
```powershell
PS> curl.exe -s -X DELETE https://api.github.com/repos/nodejs/node -w "\n%{http_code}\n"
{
  "message": "Requires authentication",
  "documentation_url": "https://docs.github.com/rest",
  "status": "401"
}
401
```

**Requête 3 : code postal invalide sur geo.api.gouv.fr**
```powershell
PS> curl.exe -s "https://geo.api.gouv.fr/communes?codePostal=abc" -w "\n%{http_code}\n"
[]
200
```

### 1. Tableau comparatif

| API | Requête | Statut | Catégorie | Format du corps d'erreur | Information utile pour corriger |
|---|---|---|---|---|---|
| GitHub | `GET /repos/nodejs/ce-depot-n-existe-pas` | 404 Not Found | erreur client (4xx) | JSON maison : `message`, `documentation_url`, `status` | le message « Not Found » + lien vers la doc de l'endpoint |
| GitHub | `DELETE /repos/nodejs/node` (sans jeton) | 401 Unauthorized | erreur client (4xx) | JSON maison : `message`, `documentation_url`, `status` | « Requires authentication » : il faut envoyer un jeton |
| geo.api.gouv.fr | `GET /communes?codePostal=abc` | 200 OK | succès (devrait être 4xx) | aucun : liste vide `[]` | aucune |

Aucune de ces requêtes ne produit d'erreur serveur (5xx) : toutes viennent d'une erreur du client.

**Format commun ?** Non. GitHub a son propre format JSON (et met `status` en chaîne `"404"` au lieu d'un nombre) ; geo.api.gouv.fr ne renvoie aucun corps d'erreur. Le plus utile au client est GitHub : message lisible + lien vers la documentation.

### 2. Erreur qui ne respecte pas la sémantique HTTP
La requête 3 sur geo.api.gouv.fr. `abc` n'est pas un code postal valide (5 chiffres attendus) : c'est une erreur du client, qui devrait recevoir un statut 4xx (`400 Bad Request`, ou `422` pour un filtre invalide). L'API répond `200 OK` avec une liste vide, ce qui fait croire à une recherche valide sans résultat. Le client ne peut pas savoir que sa requête est fausse.

### 3. Réécriture au format RFC 9457 (requête 3)
```http
HTTP/1.1 422 Unprocessable Content
Content-Type: application/problem+json

{
  "type": "https://geo.api.gouv.fr/problems/invalid-parameter",
  "title": "Paramètre de requête invalide",
  "status": 422,
  "detail": "Le code postal 'abc' est invalide : 5 chiffres attendus (ex. 69001).",
  "instance": "/communes?codePostal=abc"
}
```

## Étape 4 – Collections et pagination

**Requête 1 : filtre et sélection de champs (geo.api.gouv.fr)**
```powershell
PS> curl.exe -s "https://geo.api.gouv.fr/communes?nom=Lyon&fields=nom,code,population" | jq
```
Résultat (extrait, 7 communes renvoyées) :
```json
{"nom":"Lyon","code":"69123","population":519127,"_score":0.538}
{"nom":"Cognat-Lyonne","code":"03080","population":706,"_score":0.442}
{"nom":"Lyons-la-Forêt","code":"27377","population":756,"_score":0.435}
{"nom":"Sainte-Foy-lès-Lyon","code":"69202","population":21692,"_score":0.336}
```
Sans `fields`, chaque commune contient aussi `codeDepartement`, `siren`, `codeEpci`, `codeRegion`, `codesPostaux`.

**Requête 2 : pagination (GitHub)**
```powershell
PS> curl.exe -sI "https://api.github.com/repos/nodejs/node/issues?per_page=5" | Select-String '^link:'
Link: <https://api.github.com/repositories/27193779/issues?per_page=5&after=Y3Vyc29yOnYyOpLPAAABofjVUWDPAAAAAYQWBb0%3D&page=2>; rel="next"
```

### 1. Filtrer et choisir les champs (geo.api.gouv.fr)
- **Filtre** : par paramètre de requête, ici `?nom=Lyon`. La recherche est approximative : elle renvoie aussi Lyons-la-Forêt, Sainte-Foy-lès-Lyon…, triées par pertinence (`_score`).
- **Choix des champs** : paramètre `fields`, ici `fields=nom,code,population`. Seuls ces champs sont renvoyés, ce qui allège la réponse.

### 2. Information de page suivante (GitHub)
Elle est dans un **en-tête**, pas dans le corps : l'en-tête `Link`, avec la relation `rel="next"`. Le corps ne contient que le tableau des issues.
Ici il n'y a pas de `rel="last"` : GitHub pagine par curseur (`after=…`), il ne calcule donc pas le nombre total de pages.

### 3. Avantage des liens de pagination dans un en-tête
- Le corps reste la donnée pure (un tableau de ressources), sans enveloppe de métadonnées.
- Le client lit la pagination sans parser le JSON (possible même avec une requête `HEAD`).
- C'est un standard (RFC 8288, `Link`) compris par des clients génériques.
- Le serveur fournit des URL toutes prêtes : le client ne les construit pas, et le serveur peut changer de mécanisme (numéro de page → curseur) sans casser les clients.git push

## Étape 5 – Négociation de contenu

**Requête 1 : README en Markdown brut**
```powershell
PS> curl.exe -s -H "Accept: application/vnd.github.raw+json" https://api.github.com/repos/nodejs/node/readme | Select-Object -First 5
# Node.js

Node.js is an open-source, cross-platform JavaScript runtime environment.

For information on using Node.js, see the [Node.js website][].
```
En-tête de réponse : `Content-Type: application/vnd.github.raw+json; charset=utf-8`

**Requête 2 : README en HTML**
```powershell
PS> curl.exe -s -H "Accept: application/vnd.github.html+json" https://api.github.com/repos/nodejs/node/readme | Select-Object -First 5
<div id="readme" class="md" data-path="README.md"><article class="markdown-body entry-content container-lg" itemprop="text"><div class="markdown-heading" dir="auto"><h1 class="heading-element" dir="auto">Node.js</h1>...
<p dir="auto">Node.js is an open-source, cross-platform JavaScript runtime environment.</p>
```
En-tête de réponse : `Content-Type: application/vnd.github.html+json; charset=utf-8`

**Test supplémentaire : format non disponible**
```powershell
PS> curl.exe -s -D - -o NUL -H "Accept: application/xml" https://api.github.com/repos/nodejs/node/readme
HTTP/1.1 415 Unsupported Media Type
Content-Type: application/json; charset=utf-8
```

### Réponses
- **En-tête qui choisit le format** : `Accept`, envoyé par le client. Même URI, même ressource (le README), deux représentations différentes. Le serveur confirme son choix dans `Content-Type`.
- **Si aucun format demandé n'est disponible** : `406 Not Acceptable`.
- **Observation** : GitHub répond `415 Unsupported Media Type` à `Accept: application/xml`, ce qui est incorrect : 415 concerne le format du corps **envoyé** par le client (`Content-Type`), alors que 406 concerne le format **demandé** (`Accept`).

## Étape 6 – L'échelle de Richardson

Rappel : niveau 0 = une seule URI et tout en POST ; niveau 1 = une URI par ressource ; niveau 2 = verbes HTTP et codes de statut utilisés correctement ; niveau 3 = hypermédia (la réponse contient des liens vers les actions et ressources liées).

### GitHub REST API : niveau 3 (partiel)
- Ressources : `/repos/nodejs/node`, `/repos/nodejs/node/issues`.
- Verbes et statuts : `GET` → 200, `DELETE` sans jeton → 401, dépôt inexistant → 404, `If-None-Match` → 304.
- Hypermédia : la réponse contient des liens vers les ressources liées :
```json
"url": "https://api.github.com/repos/nodejs/node",
"issues_url": "https://api.github.com/repos/nodejs/node/issues{/number}",
"owner": { "url": "https://api.github.com/users/nodejs" }
```
  et l'en-tête `Link: <...&page=2>; rel="next"` pour la pagination (étape 4).
- Partiel : les liens décrivent les ressources liées, pas les actions possibles selon l'état de la ressource.

### geo.api.gouv.fr : niveau 2 (imparfait)
- Ressources identifiées par URI, avec sous-ressources :
```powershell
PS> curl.exe -s "https://geo.api.gouv.fr/communes/69123?fields=nom,codeDepartement"
{"nom":"Lyon","codeDepartement":"69","code":"69123"}      → 200
PS> curl.exe -s "https://geo.api.gouv.fr/communes/99999"
Not Found                                                  → 404
GET /departements/69/communes                              → 200
```
- Mais les statuts ne sont pas toujours respectés :
```powershell
PS> curl.exe -s -X POST "https://geo.api.gouv.fr/communes"
Method POST not allowed                                    → 401
```
  Il faudrait `405 Method Not Allowed` (avec l'en-tête `Allow: GET`). Et `?codePostal=abc` renvoie 200 au lieu d'un 4xx (étape 3).
- Aucun lien hypermédia dans les réponses → pas de niveau 3.

### Pathly (API NestJS du projet d'équipe) : niveau 2
Requêtes observées dans les tests HTTP de l'API (`apps/api/src/api.http.spec.ts`) :
- `GET /api/v1/accounts` sans session → 401 ; avec session → 200
- `GET /api/v1/accounts/{id inconnu}` → 404
- `GET /api/v1/accounts/abc` → 400
- `GET /api/v1/transactions?limit=1&cursor=...` → pagination par curseur en paramètre de requête

Ressources au pluriel, version dans l'URI, statuts cohérents → niveau 2.
Limites : `POST /api/v1/import/preview` et `POST /api/v1/import/commit` désignent des actions (style RPC) ; aucun lien hypermédia → pas de niveau 3.