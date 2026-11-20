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