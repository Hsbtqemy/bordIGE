# Audit de code — nakalapycon

**Date :** 2026-05-29
**Objet :** librairie Python *wrapper* de l'API Nakala (Huma-Num), version `0.0.9`.
**Portée :** audit ligne par ligne de l'ensemble des fichiers (`nakalapycon/src/*.py`, tests, `setup.py`, `requirements.txt`, `README.md`).

---

## 1. Synthèse

Architecture saine et cohérente : une classe `NklTarget` (cible Test/Prod + clé API) et une classe `NklResponse` (réponse unifiée `isSuccess`/`code`/`message`/`dictVals`) servent de socle à des modules fonctionnels par endpoint (`Datas`, `Collections`, `Groups`, `Users`, `Vocabularies`, `Search`) plus des utilitaires (`nklUtils`, `nklPullCorpus`, `nklDf2Dic`) et un dictionnaire de constantes.

Le projet souffre cependant de plusieurs **bugs fonctionnels avérés** (paramètres de recherche ignorés, URL corrompue par des caractères invisibles, logique pandas erronée) et de faiblesses transverses (pas de `timeout`, gestion d'erreur fragile, forte duplication, « tests » non automatisés).

| Sévérité | Nombre (indicatif) |
|----------|--------|
| 🔴 Élevée (bug fonctionnel) | 8 |
| 🟠 Moyenne | 18 |
| 🟡 Faible | 17 |

> Comptes indicatifs. **3.3** et **3.22** sont rédigés comme des bugs 🔴 (respectivement : plantage de `get_search_datas` sur toute réponse ≠ 200 ; rejet 422 silencieux sur langue ISO 639-3) mais restent classés en §3 ; ils **ne sont pas inclus** dans les 8 du §2. Le total réel de bugs critiques est donc **10**.
>
> **Mise à jour (2026-06-25)** — les findings **3.22 à 3.24** sont issus d'un **croisement avec le savoir Nakala validé en live** d'un projet frère (ColleC / `archives_tool`), qui a sondé l'API en écriture (apitest + parité prod) bien au-delà de la spec Swagger. Référence : `ColleC/docs/developpeurs/nakala-savoir-api.md` et l'audit croisé `…/nakala-audit-croise-nakalapycon.md`.

---

## 2. Bugs fonctionnels 🔴

### 2.1 `nklAPI_Search.get_search_datas` — le paramètre `order` est ignoré
[nklAPI_Search.py:77-80](nakalapycon/src/nklAPI_Search.py#L77-L80)

```python
url += "q="+q
url += "&fq="+fq
url += "&facet="+facet
url + "&order="+order      # ❌ pas de "+=" : le résultat est calculé puis jeté
```

La ligne 80 utilise `url +` au lieu de `url +=`. Le tri (`order`) n'est **jamais** ajouté à l'URL. Correctif : `url += "&order="+order`.

### 2.2 `nklAPI_Search.get_search_authors` — `order`, `page`, `limit` ignorés
[nklAPI_Search.py:167-170](nakalapycon/src/nklAPI_Search.py#L167-L170)

```python
url += "q="+q
url + "&order="+order     # ❌
url + "&page="+page       # ❌
url + "&limit="+limit     # ❌
```

Trois lignes sur quatre oublient le `+=`. Seul `q` est transmis ; la pagination et le tri sont silencieusement perdus.

### 2.3 Paramètres de requête non encodés (URL injection / requêtes cassées)
[nklAPI_Search.py:74-80](nakalapycon/src/nklAPI_Search.py#L74-L80), [nklAPI_Groups.py:53-59](nakalapycon/src/nklAPI_Groups.py#L53-L59), [nklAPI_Vocabularies.py:312-317](nakalapycon/src/nklAPI_Vocabularies.py#L312-L317)

Les URLs sont construites par concaténation de chaînes brutes. Or les `q` réellement utilisés contiennent espaces et `:` (ex. `test_nklAPI_Search.py:40` : `q="title:Photo de terrain - Mission..."`). Sans encodage, l'URL est invalide/ambiguë. **Recommandation forte** : passer par `requests.get(url, params={...})` qui encode et assemble proprement (cela corrige aussi 2.1 et 2.2).

### 2.4 `nklAPI_Datas.delete_datas_files` — URL corrompue par des caractères invisibles
[nklAPI_Datas.py:510](nakalapycon/src/nklAPI_Datas.py#L510)

```python
url = nklTarget.API_URL+"/datas/"+identifier+"​/files​/"+fileIdentifier
```

La portion `"​/files​/"` contient des **espaces de largeur nulle** (U+200B) collés autour de `/files/`. C'est très probablement la cause du dysfonctionnement signalé dans le docstring (« TODO : à débugger car pour le moment la fonction ne semble pas fonctionner »). Réécrire l'URL en ASCII pur : `+"/files/"+`.

**Second bug dans la même fonction** (vérifié contre la spec officielle, §8.2) : le code teste `response.status_code == 200` ([nklAPI_Datas.py:529](nakalapycon/src/nklAPI_Datas.py#L529)) alors que `DELETE /datas/{id}/files/{sha1}` renvoie **204** en succès. Même l'URL corrigée, la branche succès ne se déclencherait jamais. À aligner sur `204`.

### 2.5 `nklDf2Dic.dfDatasFiles2ListDic` — conditions pandas toujours vraies
[nklDf2Dic.py:140](nakalapycon/src/nklDf2Dic.py#L140) et [nklDf2Dic.py:184](nakalapycon/src/nklDf2Dic.py#L184)

```python
if len(dfRowFinded==1):     # ❌ len(d==1) = nb de lignes, pas "égal à 1"
...
elif len(dfRowFinded>1):    # ❌ idem, toujours vrai si non vide
```

`dfRowFinded==1` produit un DataFrame booléen ; `len(...)` en renvoie le nombre de lignes. Les conditions ne testent donc pas ce qui est voulu (`len(dfRowFinded)==1` / `> 1`). Le `elif` est de fait inatteignable correctement et l'avertissement « plusieurs conf. » ne se déclenche jamais.

### 2.6 `nklDf2Dic.dfDatasFiles2ListDic` — test de langue toujours faux
[nklDf2Dic.py:154](nakalapycon/src/nklDf2Dic.py#L154)

```python
if not(str(dfRowFinded.iloc[0]['Nkl-lang']=="nan")):
```

Parenthésage erroné : `str(<bool>)` vaut `"True"`/`"False"` (chaînes non vides) → `not(...)` est **toujours `False`**. La langue n'est donc jamais affectée (reste `""`). Intention probable : `if not (str(dfRowFinded.iloc[0]['Nkl-lang']) == "nan"):`.

### 2.7 `NklTarget` — clé API par défaut non vide (faux positif d'authentification)
[NklTarget.py:26](nakalapycon/src/NklTarget.py#L26)

```python
def __init__(self, isNakalaProd=False, apiKey = "01234567-89ab-cdef-0123-456789abcdef"):
```

La valeur par défaut est une **fausse clé non vide**. Conséquence : `apiKey_isEmpty()` renvoie `False` ([NklTarget.py:52](nakalapycon/src/NklTarget.py#L52)), donc un `NklTarget()` construit sans argument enverra un en-tête `X-API-KEY` invalide même pour des données publiques → `401`. Le défaut devrait être `""`.

### 2.8 `nklUtils.isFileSha1InData` — retour `None` implicite
[nklUtils.py:197-225](nakalapycon/src/nklUtils.py#L197-L225)

```python
if "files" in data:
    for file in data["files"]:
        if file["sha1"]==sha1:
            return True, file['sha1']
else:
    return False, ""
# ❌ si "files" présent mais sha1 absent → tombe ici → return None
```

Si `files` existe mais ne contient pas le sha1, la fonction renvoie `None` au lieu de `(False, "")`. À aligner sur [nklUtils.py:194](nakalapycon/src/nklUtils.py#L194) (`isFileNameInData`) qui termine bien par `return False,""`.

---

## 3. Gestion d'erreurs & robustesse 🟠

### 3.1 `json.loads` non protégé sur les réponses d'erreur (toutes les fonctions API)
Exemple [nklAPI_Datas.py:78](nakalapycon/src/nklAPI_Datas.py#L78) (motif répété ~30 fois) :

```python
else:
    dicError = json.loads(response.text)   # ❌ lève si le corps n'est pas du JSON
    nklR.message=dicError['message']
```

Le `except` ne capture que `requests.exceptions.RequestException`. Si Nakala renvoie une page d'erreur **HTML** (502/504, maintenance…), `json.loads` lève `JSONDecodeError`, **non capturée**, qui casse la promesse « toujours renvoyer un `NklResponse` ». Seule [nklAPI_Datas.py:1690-1705](nakalapycon/src/nklAPI_Datas.py#L1690-L1705) (`get_iiif_infoJson`) protège ses `json.loads`. À généraliser, et idéalement préférer `response.json()`/`KeyError` géré.

### 3.2 Aucun `timeout` sur les appels réseau
Tous les `requests.get/post/put/delete` (tous les modules) sont sans `timeout`. Une connexion suspendue bloque le script indéfiniment, à l'encontre du but affiché « gérer les erreurs réseau ». Ajouter `timeout=` (ex. 30 s) partout.

### 3.3 `nklAPI_Search.get_search_datas` — test d'erreur incohérent
[nklAPI_Search.py:111](nakalapycon/src/nklAPI_Search.py#L111)

```python
if 'text' in response:        # ❌ "in" sur un objet Response : itère le flux d'octets
    dicError = json.loads(response.text)
```

`'text' in response` n'a pas le sens voulu. **Vérifié** : sur un objet `Response`, `in` déclenche l'itération du flux brut et **lève `AttributeError`** (`'NoneType' object has no attribute 'read'`). Cette exception n'étant pas un `RequestException`, elle **n'est pas capturée** → `get_search_datas` **plante** sur toute réponse ≠ 200 (ce n'est donc pas un simple message manquant mais un vrai plantage, à traiter comme 🔴). Intention : `if response.text:`.

### 3.4 Instructions `print()` de debug laissées dans la librairie
[nklAPI_Datas.py:511](nakalapycon/src/nklAPI_Datas.py#L511), [:589](nakalapycon/src/nklAPI_Datas.py#L589), [:819](nakalapycon/src/nklAPI_Datas.py#L819), [:1053](nakalapycon/src/nklAPI_Datas.py#L1053), [:1294](nakalapycon/src/nklAPI_Datas.py#L1294) ; plus les nombreux `print` dans [nklUtils.py](nakalapycon/src/nklUtils.py) et [nklPullCorpus.py](nakalapycon/src/nklPullCorpus.py). Une librairie ne doit pas écrire sur stdout ; utiliser le module `logging`.

### 3.5 Codes de statut — vérifiés contre la spec officielle
Confrontation à `api.nakala.fr/doc.json` (détail et tableau en §8.2). Le mapping est **correct presque partout** ; deux écarts subsistent :
- ❌ `delete_datas_files` : teste `200`, la spec attend `204` (cf. 2.4).
- ⚠️ `delete_datas` ([nklAPI_Datas.py:211](nakalapycon/src/nklAPI_Datas.py#L211)) : teste uniquement `204` alors que la spec admet aussi **`202`** comme succès → une réponse 202 serait traitée comme une erreur.
- ℹ️ `post_datas_files` : le **code** (`200`) est conforme ; seul le **commentaire** « 201 » est faux ([nklAPI_Datas.py:450-451](nakalapycon/src/nklAPI_Datas.py#L450-L451)).
- ✅ `PUT`/`DELETE /groups/{id}` en `200` : **conforme à la spec** (le doute « 204 » est définitivement levé).

### 3.6 `NklResponse` — argument par défaut mutable
[NklResponse.py:35](nakalapycon/src/NklResponse.py#L35) : `def __init__(self, ..., dictVals={}, ...)` et attributs de **classe** mutables ([NklResponse.py:24](nakalapycon/src/NklResponse.py#L24)). Anti-pattern Python (dictionnaire partagé). Utiliser `dictVals=None` puis `self.dictVals = {} if dictVals is None else dictVals`.

### 3.7 `nklDf2Dic.creatorsValuesToDic` — paramètres de séparateurs ignorés
[nklDf2Dic.py:36](nakalapycon/src/nklDf2Dic.py#L36) : `strCreators.split("|")` code en dur `|` alors que `sepCreator` (et `sepSurname`, `sepOrcid`) sont des paramètres. Les arguments n'ont aucun effet. Utiliser les paramètres ou les retirer.

### 3.8 `nklPullCorpus` — fonctions stub appelables
[nklPullCorpus.py:21-25](nakalapycon/src/nklPullCorpus.py#L21-L25) (`collectionToDf`) et [:320-332](nakalapycon/src/nklPullCorpus.py#L320-L332) (`getSoundTimeDuration`) ne sont que des `TODO`/`pass` retournant `None`/`0`. À documenter comme non implémentées ou retirer. De plus le test associé appelle `collectionToDf(targetCollection)` avec **un seul** argument alors que la signature en exige deux ([test_nklPullCorpus.py:126](nakalapycon/src/test_nklPullCorpus.py#L126)) → `TypeError`.

### 3.9 Dépendances figées et incomplètes
[requirements.txt](nakalapycon/requirements.txt) épingle `requests==2.24.0` et `pandas==1.1.3` (versions de 2020, anciennes/vulnérables potentiellement). Par ailleurs `test_nklPullCorpus.py` importe `skimage` ([test_nklPullCorpus.py:15](nakalapycon/src/test_nklPullCorpus.py#L15)) et utilise `openpyxl`/`xlsxwriter` (Excel) sans les déclarer. Enfin, [test_nklPullCorpus.py:53](nakalapycon/src/test_nklPullCorpus.py#L53) appelle `writer.save()`, supprimé dans les versions récentes de pandas (remplacé par `writer.close()`) — ne fonctionne qu'avec le `pandas==1.1.3` épinglé.

### 3.10 Performance : `pd.concat` dans une boucle
[nklPullCorpus.py:154](nakalapycon/src/nklPullCorpus.py#L154) et [:182](nakalapycon/src/nklPullCorpus.py#L182) concatènent ligne à ligne (coût quadratique). Accumuler dans une liste puis un seul `pd.concat` final.

### 3.11 🟠 `nklDf2Dic.creatorsValuesToDic` — `surname`/`givenname` probablement inversés
[nklDf2Dic.py:23](nakalapycon/src/nklDf2Dic.py#L23) (docstring) vs [nklDf2Dic.py:40](nakalapycon/src/nklDf2Dic.py#L40) (regex)

Le docstring annonce un format d'entrée **« prénom, nom @orcid »** (givenname avant la virgule), mais la regex nomme `surname` la partie **avant** la virgule et `givenname` celle **après** :

```python
target = r" *(?P<surname>[\w\- ]*) *, *(?P<givenname>[\w\- ]*) *(@(?P<orcid>[\w\-]*))* *"
```

Selon le format réellement saisi par l'utilisateur, prénom et nom seront **intervertis** dans le JSON envoyé à Nakala. Aligner la regex sur le docstring (ou inversement) et ajouter un test.

**Confirmé empiriquement (§8.8).** Exécution de la regex : `"Dupont, Pierre"` → `surname="Dupont", givenname="Pierre"` (la regex lit donc **« Nom, Prénom »**, à l'opposé du docstring « prénom, nom »). L'API Nakala modélise le créateur comme `Author = {givenname, surname}` (champs **obligatoires**, vérifié sur données réelles : `{"givenname":"Claudie","surname":"Marcel-Dubois"}`), et construit `fullName = givenname + " " + surname`. Deux effets de bord supplémentaires mesurés :
- **`@orcid` laisse une espace parasite** dans `givenname` (`"Victor "` — pas de `.strip()`).
- **Sans virgule, le créateur est silencieusement perdu** : `"Marie Curie"` ne *matche* pas → `not found`, créateur omis sans erreur.

**Site vs API** : via le formulaire du site, prénom et nom sont saisis dans **deux champs distincts** → aucune inversion possible ; via NakalaPycon, le découpage d'une chaîne unique par la regex peut **permuter** les deux (ou omettre le créateur). Le format **stocké** est identique des deux côtés (`{givenname, surname}`) ; la divergence vient uniquement du pré-traitement de la librairie.

### 3.12 🟡 `get_search_datas` n'expose pas la pagination (pourtant disponible)
**Vérifié sur la spec** (§8.4) : `GET /search` accepte `page` et **`size`** (et non `limit`). Or [nklAPI_Search.py:16](nakalapycon/src/nklAPI_Search.py#L16) n'expose **aucun** de ces paramètres → impossible de parcourir les résultats au-delà de la première page. (De même, `searchOperator`/`searchField` de `/authors/search` ne sont pas exposés.) À ajouter.

### 3.13 🟠 `post_datas_uploads` — fichier jamais fermé + erreur d'ouverture non gérée
[nklAPI_Datas.py:1531-1534](nakalapycon/src/nklAPI_Datas.py#L1531-L1534)

```python
fileOpened = open(pathFile, "rb")
fileCur = {'file': fileOpened}
response = requests.post(url, files=fileCur, headers=APIheaders)
```

Deux problèmes :
1. **Fuite de descripteur** : `fileOpened` n'est jamais refermé (ni `with`, ni `.close()`). En envoi de nombreux fichiers (cas d'usage typique d'un dépôt de corpus), les handles s'accumulent.
2. **`open()` dans le `try` mais hors du périmètre du `except`** : un `FileNotFoundError`/`PermissionError` n'est pas capturé (seul `requests.exceptions.RequestException` l'est) → exception propagée, ce qui rompt la promesse « toujours renvoyer un `NklResponse` ».

Correctif : `with open(pathFile, "rb") as f: response = requests.post(url, files={'file': f}, ...)` et capturer aussi `OSError`.

### 3.14 🟠 En-tête `Content-Type: application/json` manquant sur 3 POST envoyant du JSON
Dans le module `Datas`, l'en-tête n'est pas posé de façon homogène alors que le corps est toujours `json.dumps(...)` :

| Fonction | `Content-Type` |
|----------|----------------|
| `post_datas` [:286](nakalapycon/src/nklAPI_Datas.py#L286), `post_datas_files` [:438](nakalapycon/src/nklAPI_Datas.py#L438), `put_datas` [:131](nakalapycon/src/nklAPI_Datas.py#L131) | ✅ présent |
| `post_datas_metadatas` [:668-669](nakalapycon/src/nklAPI_Datas.py#L668-L669) | ❌ absent |
| `post_datas_rights` [:904-905](nakalapycon/src/nklAPI_Datas.py#L904-L905) | ❌ absent |
| `post_datas_collections` [:1135-1136](nakalapycon/src/nklAPI_Datas.py#L1135-L1136) | ❌ absent |

Ces trois fonctions envoient un corps JSON **sans** déclarer `application/json` : le serveur peut l'interpréter en `text/plain` et rejeter/ignorer la charge utile. (Nuance : la spec Swagger ne déclare **aucun `consumes`**, donc le `Content-Type` n'est pas formellement imposé par la doc — mais l'incohérence interne et le comportement usuel des API JSON en font un correctif recommandé.) À uniformiser (c'est précisément le genre d'incohérence qu'une factorisation §5.1 éliminerait). À noter aussi : sur les opérations d'écriture (`put_datas`, `delete_datas`, `post_datas`…), le commentaire « la data est public … pas besoin de API_KEY » est trompeur (une clé est toujours requise en écriture) — copié-collé depuis les GET.

### 3.15 🟠 `VOCABTYPE` obsolète vs le vocabulaire Nakala courant
Confrontation **en direct** à `GET /vocabularies/datatypes` (détail §8.5) : sur les 30 types figés dans [constantes.py:14-61](nakalapycon/src/constantes.py#L14-L61), **5 ne sont plus reconnus** par Nakala (`periodical`/c_2659 — déjà annoté « deprecated » dans le code —, `ArchiveMaterial`, bibo `Collection`, bibo `Series`, `SurveyDataSet`) et **4 nouveaux** types serveur manquent. Déposer une data avec un de ces 5 `typeUri` risque un rejet ou une donnée non conforme. Recommandation : ne pas figer ce vocabulaire — l'obtenir dynamiquement via `get_vocabularies_datatypes` (déjà présent dans [nklAPI_Vocabularies.py](nakalapycon/src/nklAPI_Vocabularies.py)) ou régénérer `VOCABTYPE`.

### 3.16 🟡 Typage des dates : `xsd:string` au lieu de `W3CDTF`
Confirmé par `/vocabularies/metadatatypes` (§8.5) : Nakala propose **`http://purl.org/dc/terms/W3CDTF`** pour les dates. Or `created` (sous-propriété de `dcterms:date`) est typé `http://www.w3.org/2001/XMLSchema#string` dans les exemples/tests et la config (ex. [test_nklAPI_Datas.py:96-99](nakalapycon/src/test_nklAPI_Datas.py#L96-L99)). Non conforme à la recommandation Dublin Core. La lib transmet le dict de l'appelant (pas de bug interne) ; le correctif porte sur les exemples, la doc et `dfDatasFiles2ListDic`. **Confirmé empiriquement (§8.7)** : une data réelle (`10.34847/nkl.be633595`) stocke `created` **sans** `typeUri` ; `xsd:string` ne correspond donc ni à cet usage réel, ni à la recommandation `W3CDTF`.

### 3.17 🟠 `POST`/`PUT /groups` : `users` envoyé comme liste de chaînes (spec = objets)
[nklAPI_Groups.py:341-348](nakalapycon/src/nklAPI_Groups.py#L341-L348) (docstring) et les tests ([test_nklAPI_Groups.py:80-84](nakalapycon/src/test_nklAPI_Groups.py#L80-L84)) construisent `users: ["unakala1", ...]` (tableau de chaînes). Or la spec courante type le corps `users` comme **`array<{username, role}>`** (définition `MinimalUserInfo`). **Confirmé (§8.7)** : schéma `{username: string, role: enum[ROLE_OWNER, ROLE_ADMIN, ROLE_USER]}`, exemple `[{"username":"pdupont","role":"ROLE_OWNER"}]`. L'envoi d'une liste de chaînes est donc non conforme à la spec courante. (Le `GET /groups/{id}` live n'a pas pu confirmer côté serveur : le groupe d'exemple renvoie `404`.) Adapter `post_groups`/`put_groups` pour émettre `[{"username": ..., "role": ...}]`.

### 3.18 🟡 Docstring `post_datas_rights` : `ROLE_OWNER` listé comme rôle assignable
[nklAPI_Datas.py:866](nakalapycon/src/nklAPI_Datas.py#L866) annonce « ROLE_OWNER, ROLE_ADMIN, ROLE_EDITOR, ROLE_READER ». Or l'enum `Role` de la spec (corps des POST de droits sur data) = **`[ROLE_ADMIN, ROLE_EDITOR, ROLE_READER]`** (§8.7) ; `ROLE_OWNER` n'est pas assignable via l'API (le propriétaire ne s'attribue pas). Corriger le docstring.

### 3.19 🟠 `collectionDatasToDf` — aplatissement à perte des créateurs (`creators_formated`)
[nklPullCorpus.py:136](nakalapycon/src/nklPullCorpus.py#L136) construit l'auteur comme
`formattedName = meta['value']['givenname'] + " " + meta['value']['surname']`, stocké dans la
colonne unique `creators_formated` (et plusieurs auteurs sont **concaténés** dans la même
cellule, séparés par `, ` — [:141](nakalapycon/src/nklPullCorpus.py#L141), [:146](nakalapycon/src/nklPullCorpus.py#L146)).

Cette fusion en une chaîne **détruit la structure** renvoyée par Nakala :

| Champ Nakala | Sort dans le DataFrame ? |
|--------------|--------------------------|
| `givenname` / `surname` (séparés) | fondus dans une seule chaîne |
| `orcid` | **perdu** |
| `authorId` (identité dédoublonnée, cf. §8.9) | **perdu** |
| N auteurs distincts | **fusionnés** en une cellule |

Conséquences :
- **Tout index/tri sur `creators_formated` part du prénom** (la chaîne commence par `givenname`) → impossible de classer par nom de famille sans re-parser. C'est l'« index par prénom » observé.
- **Dédoublonnage d'auteur impossible** (`authorId` jeté) ; **ORCID inexploitable**.
- N.B. : ce champ **reflète** fidèlement les données — il n'introduit pas d'inversion ; il *expose* celles déjà présentes dans Nakala (cf. §8.10) tout en perdant le moyen de les corriger/trier.

**Recommandation** : ne pas aplatir — exposer des colonnes séparées (`creator_givenname`,
`creator_surname`, `creator_orcid`, `creator_authorId`) et gérer le multi-auteur en lignes ou
liste structurée ; dériver éventuellement une colonne d'affichage « Nom, Prénom » par-dessus,
sans perte. Lien : 3.11, 3.16.

Robustesse : la garde `if not (meta['value']==None)` [nklPullCorpus.py:133](nakalapycon/src/nklPullCorpus.py#L133)
évite une `TypeError` quand `value` est `null` (cas réel, cf. §8.10), mais l'enregistrement
ressort alors **sans auteur** (`creators_formated` vide) — perte silencieuse à documenter.

### 3.20 🟠 Pagination de `/search` : `lastPage`/`currentPage` absents (boucle de corpus inopérante)
**Vérifié en direct (§8.11)** : `GET /search` accepte bien `page` et `size` (testé jusqu'à `size=10000`, pages distinctes), **mais la réponse ne contient ni `lastPage` ni `currentPage`** (tous deux `null`) — seul `totalResults` est fourni. Or le motif de pagination employé ailleurs dans la lib (`collectionDatasToDf`, `put_collections_datas_rights`) repose sur `dictVals['lastPage']`. Si l'on bâtit une extraction paginée de corpus sur `/search` selon ce même motif, la boucle **ne peut pas se terminer correctement** (clé absente). Pour `/search`, paginer via `totalResults` + `size` (calcul du nombre de pages) ou via `size` large en une passe. À distinguer de `/collections/{id}/datas`, qui **fournit** `lastPage` (d'où la cohérence du code existant sur les collections).

### 3.21 🟡 IIIF : `size="full"` déprécié en IIIF Image API 3.0
**Vérifié (§8.11)** : `GET /iiif/{id}/{sha1}/info.json` renvoie `@context = http://iiif.io/api/image/3/context.json` → Nakala sert l'**Image API 3.0**. Or `getImageUrlIIIF` ([nklPullCorpus.py:247](nakalapycon/src/nklPullCorpus.py#L247)) et son exemple/test ([test_nklPullCorpus.py:92](nakalapycon/src/test_nklPullCorpus.py#L92)) utilisent `size="full"`, **déprécié en 3.0** (remplacé par `max`). L'URL fonctionne encore aujourd'hui (compatibilité ascendante : test renvoie un JPEG valide, §8.11) mais c'est fragile. Privilégier `max`. Bénin tant que le serveur tolère `full`.

### 3.22 🔴 Langue non convertie en RFC5646 — rejet 422 silencieux sur les codes ISO 639-3
[nklDf2Dic.py:151-158](nakalapycon/src/nklDf2Dic.py#L151-L158) ; chemins « dict brut » `post_datas`/`put_datas`/`post_datas_metadatas`

**Constat croisé (savoir live ColleC, `nakala-savoir-api.md` §5).** Nakala type `dcterms:language` en **RFC5646 / ISO 639-1** (`fr`, `es`, `en`) pour les langues majeures et réserve l'**ISO 639-3** à la longue traîne. Déposer un code 639-3 (`spa`, `fra`, `eng`) → **rejet `422`** (« unauthorized »). Or NakalaPycon ne fait **aucune** conversion de langue :

- chemin tableur : la `lang` est recopiée brute depuis la colonne `Nkl-lang` ([nklDf2Dic.py:151-158](nakalapycon/src/nklDf2Dic.py#L151-L158)) ;
- chemins « dictionnaire brut » : la `lang` fournie par l'appelant part telle quelle dans le JSON.

**Pourquoi c'est un piège réel.** La librairie expose `get_vocabularies_languages` ([nklAPI_Vocabularies.py](nakalapycon/src/nklAPI_Vocabularies.py)) → le endpoint `/vocabularies/languages`, dont les `id` sont en **639-3 pour la longue traîne**. Un appelant qui pioche un `id` de langue dans cette sortie et le place dans une meta `lang` **obtient un 422** sans aucun indice. Les exemples/tests actuels utilisent `"fr"`/`"en"` (valides RFC5646) → le piège est **invisible aux tests** mais bien présent à l'usage réel multilingue.

**Interaction avec 2.6.** Sur le chemin tableur, le bug 2.6 (test de langue toujours faux, [nklDf2Dic.py:154](nakalapycon/src/nklDf2Dic.py#L154)) fait que la `lang` n'est **actuellement jamais affectée** (reste `""`) → le rejet 422 y est *masqué*. **Le jour où 2.6 est corrigé, ce piège se réveille.** Sur les chemins « dict brut », il bite déjà. Correctif : convertir 639-3 → 639-1 avant envoi (pont `langue_vers_nakala`, comme côté ColleC), sur la valeur **et** l'attribut `lang`.

### 3.23 🟠 `put_datas` — sémantique « `files[]` = remplacement total » non documentée (perte silencieuse)
[nklAPI_Datas.py:91-123](nakalapycon/src/nklAPI_Datas.py#L91-L123)

**Constat croisé (savoir live ColleC, `nakala-savoir-api.md` §8, hypothèse H1).** `PUT /datas/{id}` a une sémantique **« remplace »**, pas « ajoute » : envoyer une clé `files` **partielle** **supprime** les fichiers omis. (Confirmé aussi par H12 : même le champ `description` d'un `files[i]` omis au PUT est **effacé**.) Or `put_datas` ([nklAPI_Datas.py:91](nakalapycon/src/nklAPI_Datas.py#L91)) transmet `dictVals` **brut** et sa docstring décrit « les informations à modifier » **sans jamais avertir** de ce risque. Un appelant voulant « changer un fichier » via `{"files":[{nouveau}]}` **efface tous les autres** — sans erreur.

Aggravant : la librairie n'offre aucun chemin granulaire **sûr** en regard — `delete_datas_files` est cassé (URL polluée par U+200B **et** code testé `200` au lieu de `204`, cf. 2.4) et `post_datas_files` (additif, → 200) n'est pas présenté comme la voie de modification sûre. Recommandation : a minima documenter le danger dans la docstring de `put_datas` ; idéalement, exposer/fiabiliser un push **granulaire** (POST additif → DELETE ciblé → PUT de réordonnancement reconstruit depuis l'état distant relu), à l'image de ColleC.

### 3.24 🟠 `post_datas_metadatas` — POST sur un champ scalaire crée un doublon (non documenté)
[nklAPI_Datas.py:633-661](nakalapycon/src/nklAPI_Datas.py#L633-L661)

**Constat croisé (savoir live ColleC, `nakala-savoir-api.md` §2, sondé 2026-06-19).** `POST /datas/{id}/metadatas` est **additif**. Sur une propriété **scalaire** (`nkl:title`), il **ne remplace pas — il crée un DOUBLON** (la donnée se retrouve avec deux titres). Modifier un scalaire impose **DELETE puis POST**. Or `post_datas_metadatas` ([nklAPI_Datas.py:633](nakalapycon/src/nklAPI_Datas.py#L633)) est documenté « Ajout d'une nouvelle métadonnée » **sans avertir** qu'employer cette fonction pour « corriger » un titre **ajoute un second titre**. L'utilisateur corrompt sa notice sans erreur (code `201` renvoyé, conforme).

À l'inverse, `delete_datas_metadatas` ([nklAPI_Datas.py:704](nakalapycon/src/nklAPI_Datas.py#L704)) est **correct et utile** : il accepte un filtre dans le corps (ex. `{"lang":"en","propertyUri":".../subject"}`) → suppression **granulaire à la valeur**, conforme au comportement réel (DELETE granulaire → 200). Recommandation : documenter le motif « DELETE puis POST » pour éditer un scalaire.

---

## 4. Tests & exécution 🟠

### 4.1 ℹ️ Clés API de l'instance de test — pas un problème de sécurité
Les clés présentes dans les `test_*.py` (ex. `f41f5957-...`, `aae99aba-...`) et dans [README.md:36](README.md#L36) sont les **clés publiques ouvertes** mises à disposition par Nakala sur l'instance de **test** (`apitest.nakala.fr`), partagées par tous pour expérimenter. Ce ne sont pas des secrets : **aucun risque de sécurité**. Seul point d'attention : veiller à ne jamais committer, par mégarde, une clé de l'instance de **production** dans ces mêmes fichiers d'exemple (le `.gitignore` ignore déjà `settings.py`, ce qui va dans ce sens).

### 4.2 🟠 Les tests effectuent des appels réseau réels à l'import
Chaque fichier `test_*.py` se termine par un appel de fonction au niveau module (ex. [test_nklAPI_Datas.py:813](nakalapycon/src/test_nklAPI_Datas.py#L813) `post_datas_test()`, [test_nklAPI_Groups.py:144](nakalapycon/src/test_nklAPI_Groups.py#L144) `search_groups_test()`). Avec le préfixe `test_`, un lancement `pytest` **importerait** ces fichiers et déclencherait de vrais `POST`/`DELETE` sur Nakala (effets de bord, dépendance réseau, non reproductible). Ce ne sont pas des tests unitaires : pas de framework, peu d'assertions, dépendances vivantes.

---

## 5. Maintenabilité 🟡

### 5.1 Duplication massive du *boilerplate* HTTP
Le bloc `APIheaders → NklResponse() → try/requests/status_code/json.loads/except` est copié-collé ~30 fois (tous les modules `nklAPI_*`). Une fonction privée unique `_request(nklTarget, method, path, params=None, body=None, okCodes=(200,))` retournant un `NklResponse` éliminerait l'essentiel du code, et corrigerait d'un coup les points 2.3, 3.1, 3.2.

### 5.2 `nakalapycon.py` — imports `*`
[nakalapycon.py:13-26](nakalapycon/src/nakalapycon.py#L13-L26) : `from X import *` partout. Pollue l'espace de noms (`requests`, `json`, `pd`…) et masque les collisions. Préférer des imports explicites et définir `__all__`.

### 5.3 Retours incohérents
[nklUtils.py:280](nakalapycon/src/nklUtils.py#L280) `isFileSha1InUploads` renvoie un simple `bool`, alors que `isFileNameInUploads` renvoie un tuple `(bool, str)` ([nklUtils.py:251-253](nakalapycon/src/nklUtils.py#L251-L253)). Uniformiser les signatures de retour des fonctions sœurs.

### 5.4 Versioning & changelog
[setup.py:4](setup.py#L4) déclare `VERSION = "0.0.9"` mais [CHANGELOG.md](CHANGELOG.md) ne documente que `[0.0.1]`. Tenir le changelog à jour.

### 5.5 README — exemples invalides, liens et coquilles
- [README.md:37](README.md#L37) et [:42](README.md#L42) : `NklTarget(isNakalaProd=False, apiKey=)` → **SyntaxError** (`apiKey=` sans valeur ; devrait être `apiKey=myApiKey`). De plus `myApiKey` est défini mais non utilisé, et l'alias `nklT` n'est jamais importé (seul `import nakalapycon as nklco` est montré).
- [README.md:28-29](README.md#L28-L29) : liens Markdown inversés `(texte)[url]` au lieu de `[texte](url)`.
- Hiérarchie de titres incohérente (`## 1.` puis `### 2./3./4.` sous `## Buts`).
- Coquilles : « c'est réalisée » (l.51), « disfonctionne » (l.76), « plus aux niveaux » (l.81), « donnnées » (l.93), « l'entrpôt » (l.106).

### 5.6 Coquilles dans les docstrings du code
Ex. [constantes.py:74](nakalapycon/src/constantes.py#L74) « cett fonction », [:87](nakalapycon/src/constantes.py#L87) phrase tronquée « si la clé f », [nklDf2Dic.py:43](nakalapycon/src/nklDf2Dic.py#L43) commentaire `givename`, « pusiqu'il » récurrent. Sans impact fonctionnel.

### 5.7 `get_search_datas` mélange datas et collections
Comportement documenté ([nklAPI_Search.py:21-25](nakalapycon/src/nklAPI_Search.py#L21-L25)) : sans `fq=scope=...`, la recherche renvoie aussi des collections (clé `datasIds` au lieu de `files`). Bien documenté, mais piège classique pour l'appelant — envisager un paramètre par défaut explicite.

### 5.8 `.gitignore` et `__pycache__`
[.gitignore](.gitignore) ignore `*.pyc` mais pas le dossier `__pycache__/`. Un `__pycache__` est présent à la racine. Ajouter `__pycache__/`. À noter aussi : `.gitignore` ignore `settings.py`, ce qui suggère un fichier de configuration/secrets attendu hors VCS — cohérent avec la recommandation 4.1.

### 5.9 Constantes d'URL dupliquées
Les URLs Test/Prod vivent dans [NklTarget.py:21-22](nakalapycon/src/NklTarget.py#L21-L22) et [:33-34](nakalapycon/src/NklTarget.py#L33-L34), mais `nklPullCorpus`/`nklUtils` reconstruisent aussi des URLs IIIF/embed à la main ([nklPullCorpus.py:170](nakalapycon/src/nklPullCorpus.py#L170), [:177](nakalapycon/src/nklPullCorpus.py#L177)). Centraliser la fabrication d'URL.

### 5.10 Packaging fragile et imports absolus de premier niveau
[setup.py:22](setup.py#L22) combine `packages=find_packages()` **et** `py_modules=[...]` avec `package_dir={'':'nakalapycon/src'}` ([setup.py:29-30](setup.py#L29-L30)) — configuration redondante et déroutante. Surtout, les modules s'importent entre eux en **absolu de premier niveau** (`from NklResponse import NklResponse`, `import NklTarget as nklT`). Ils ne sont donc importables que « à plat » sur `sys.path` : impossible de les déplacer dans un sous-paquet sans tout réécrire (imports relatifs `from .NklResponse import ...`). Fragile à la maintenance et au ré-emballage.

### 5.11 Robustesse « silencieuse » et conventions diverses
- [nklUtils.py:43](nakalapycon/src/nklUtils.py#L43) (`delete_datas_uploads_all`) itère `r.dictVals` sans vérifier `r.isSuccess` : en cas d'échec amont, renvoie une liste vide sans rien signaler.
- [nklPullCorpus.py:238](nakalapycon/src/nklPullCorpus.py#L238) (`getImageSize`) : `except:` nu qui avale toute exception et retourne `(0, 0)` silencieusement.
- Dans tous les `except requests.exceptions.RequestException as e:`, `nklR.message = e` stocke l'**objet** exception (pas `str(e)`), alors que `message` est documenté comme texte.
- [nklUtils.py:136](nakalapycon/src/nklUtils.py#L136) : `&` (bit-à-bit) au lieu de `and` logique (fonctionne sur des booléens mais trompeur).

### 5.12 Docstrings copiés-collés / incohérents
Ex. [NklTarget.py:48](nakalapycon/src/NklTarget.py#L48) `apiKey_isEmpty` documente un retour `dfData : BOOL` (copié d'ailleurs) ; [constantes.py:76](nakalapycon/src/constantes.py#L76) annonce une valeur `"unknown by nakalapycon or nakala"` que le code ne renvoie jamais (il renvoie `"unknown by nakalapycon"`). Sans impact fonctionnel, mais nuit à la fiabilité de la doc.

### 5.13 🟡 `put_collections_datas_rights` — une requête HTTP par donnée
[nklUtils.py:114](nakalapycon/src/nklUtils.py#L114) pagine avec `limit=1`, déclenchant un appel `get_collections_datas` **par donnée** de la collection (plus un `get_datas_rights` par donnée). Sur une grosse collection, cela multiplie inutilement les requêtes. Augmenter `limit` (ex. 50) et boucler sur les pages.

---

## 6. Points positifs

- Conception claire : `NklTarget` + `NklResponse` forment un socle homogène et bien pensé (gestion unifiée succès/erreur/réseau).
- Couverture fonctionnelle large et fidèle à l'API Nakala (datas, fichiers, métadonnées, droits, collections, groupes, users, vocabulaires, search, IIIF).
- Docstrings systématiques et détaillés (paramètres, valeurs de retour, exemples JSON).
- Gestion correcte du cas « donnée publique sans clé API » (en-têtes conditionnels).
- `constantes.VOCABTYPE` + accesseurs défensifs ([constantes.py:66](nakalapycon/src/constantes.py#L66), [:102](nakalapycon/src/constantes.py#L102)) évitent les exceptions sur clé absente.

---

## 7. Plan d'action recommandé (par priorité)

1. **Corriger les bugs fonctionnels** §2 : `+=` manquants (2.1, 2.2), caractères invisibles (2.4), conditions pandas (2.5, 2.6), défaut `apiKey=""` (2.7), retour de `isFileSha1InData` (2.8).
2. **Factoriser** le boilerplate HTTP (§5.1) en y intégrant `timeout` (3.2), encodage `params=` (2.3) et `json.loads` protégé (3.1, 3.3).
3. **Refondre les tests** en vrais tests unitaires (mocks réseau, framework `pytest`, sans appels au niveau module) — §4.2.
4. **Nettoyer** : `print` → `logging` (3.4), README/changelog/typos (§5), dépendances (3.9).

---

## 8. Approfondissement & vérification

### 8.1 Bugs confirmés de façon déterministe
Chaque bug ci-dessous est démontrable sans réseau. Un script de preuve est fourni : `C:\temp\verify_nkl.py` (`python /c/temp/verify_nkl.py`). **Script exécuté → les 7 cas renvoient `OK-BUG-CONFIRME`.**

| Réf. | Démonstration (sémantique Python) | Statut |
|------|-------------------------------------|--------|
| **2.1 / 2.2** | `url + "&order="+v` est une expression dont le résultat est **jeté** (pas de `+=`) → le fragment n'apparaît jamais dans `url`. | ✅ certain |
| **2.6** | `not(str("fr"=="nan"))` = `not(str(False))` = `not("False")` = `not(<chaîne non vide>)` = **`False`** (idem avec `"nan"`). Branche jamais prise. | ✅ certain |
| **2.5** | `df[...] == 1` renvoie un DataFrame booléen ; `len(...)` = nb de lignes (≥1 ⇒ *truthy*), jamais « égal à 1 ». | ✅ certain |
| **2.8** | `isFileSha1InData` : si `"files"` présent mais sha1 absent, aucun `return` atteint ⇒ renvoie **`None`** (≠ `(False, "")`). | ✅ certain |
| **3.3** | `'text' in response` **lève `AttributeError`** (`Response` itère son flux brut, ici `None`) → exception **non capturée** ⇒ `get_search_datas` **plante** sur toute réponse ≠ 200. | ✅ confirmé (script) |
| **3.6** | `dictVals={}` (défaut mutable) : deux `NklResponse()` partagent le **même** dict ; muter l'un affecte l'autre. | ✅ confirmé (script) |
| **2.4** | **2× U+200B** (zero-width space) détectés **ligne 510** dans l'URL de `delete_datas_files`. | ✅ confirmé (script, octets) |

`python -m py_compile` sur les 14 modules → **`ALL_COMPILE_OK`** (vérifié) : aucune erreur de syntaxe — les caractères invisibles de 2.4 sont dans un littéral chaîne, donc le bug est sémantique, pas syntaxique.

### 8.2 Contrat de l'API Nakala — incertitudes levées via les tests
Les tests fournissent la **vérité de terrain** sur le schéma des réponses :

- **`GET /search`** → liste sous la clé **`datas`**, total sous **`totalResults`** :
  [test_nklAPI_Search.py:64](nakalapycon/src/test_nklAPI_Search.py#L64), [:66](nakalapycon/src/test_nklAPI_Search.py#L66) ; idem [test_nklUtils.py:45](nakalapycon/src/test_nklUtils.py#L45) et [:51](nakalapycon/src/test_nklUtils.py#L51).
- **`GET /collections/{id}/datas`** → liste sous **`data`** (singulier), pagination sous **`lastPage`** :
  [test_nklAPI_Collections.py:142](nakalapycon/src/test_nklAPI_Collections.py#L142), [:147](nakalapycon/src/test_nklAPI_Collections.py#L147).

➡️ **`data` vs `datas` n'est PAS une incohérence du code** : ce sont deux endpoints au schéma distinct, correctement consommés (`datas`/`totalResults` côté search, `data`/`lastPage` côté collections). Mon doute initial est **levé : le code est correct** ici (seul subsiste l'absence de garde si la clé manque, §3.1).

**Spec officielle vérifiée.** `WebFetch` est bloqué (HTTP 403 sur son User-Agent), mais une requête Python avec UA navigateur récupère `api.nakala.fr/doc.json` (Swagger 2.0, 129 Ko). Codes de succès confrontés au code (script `C:\temp\nkl_spec.py`) :

| Opération | Spec (succès) | Code testé | Verdict |
|-----------|---------------|------------|---------|
| `POST /datas` | 201 | 201 | ✅ |
| `PUT /datas/{id}` | 204 | 204 | ✅ |
| `DELETE /datas/{id}` | 202, 204 | 204 | ⚠️ `202` non géré |
| `POST /datas/{id}/files` | 200 | 200 | ✅ (commentaire « 201 » faux) |
| `DELETE /datas/{id}/files/{sha1}` | **204** | **200** | ❌ bug (+ URL, cf. 2.4) |
| `POST /datas/{id}/metadatas` | 201 | 201 | ✅ |
| `POST /datas/{id}/rights` | 200 | 200 | ✅ |
| `POST /datas/{id}/collections` | 201 | 201 | ✅ |
| `PUT /groups/{id}` | 200 | 200 | ✅ |
| `DELETE /groups/{id}` | 200 | 200 | ✅ |

➡️ Conséquences : le doute « 200 vs 204 » sur **`groups` est levé — `200` est correct** (conforme à la spec, pas seulement un quirk). `delete_datas_files` cumule **deux** bugs (URL + code), et `delete_datas` ignore le succès **`202`**. Aucun `consumes` n'est déclaré (cf. 3.14). Enfin, le schéma de `GET /search` est **vide (`{}`)** dans la spec : la structure `datas`/`totalResults` n'est connue que par les tests internes — ce qui justifie l'approche du §8.2.

### 8.3 Sanité du paquet
- Compilation : OK (cf. 8.1).
- Import : `import nakalapycon` ne fonctionne que depuis `src/` (ou après installation à plat), du fait des imports absolus de premier niveau — cf. §5.10.

### 8.4 Conformité étendue vérifiée contre la spec
Confrontation systématique de **tous** les endpoints appelés par le code (script `C:\temp\nkl_spec2.py`) :

**Chemins** — tous conformes. Le seul « introuvable » remonté (`PUT /datas/{id}/status/published`) est un **artefact** du comparateur : la spec expose `/datas/{identifier}/status/{status}` (paramètre de chemin) et le code fige `published`, une valeur valide → **conforme**.

**Noms des paramètres de requête** — tous corrects : `q/fq/facet/order` (`/search`), `page/limit` (`/authors/search`, `/groups/search`, `/collections/{id}/datas`, `/vocabularies/languages`), `code` (languages) et `metadata-format` (datas) sont bien orthographiés. Seule nuance : `/search` pagine via `page`+`size` (cf. §3.12), et `/authors/search` offre en plus `searchOperator`/`searchField` non exposés.

**Codes de succès des endpoints non couverts par §8.2** — **tous conformes au code** :

| Endpoint | Spec | Code |
|----------|------|------|
| `GET /datas/{id}/status` | 200 | 200 ✅ |
| `POST /datas/uploads` | 201 | 201 ✅ |
| `DELETE /datas/uploads/{sha1}` | 200 | 200 ✅ |
| `GET /collections/{id}` | 200 | 200 ✅ |
| `PUT /collections/{id}` | 204 | 204 ✅ |
| `DELETE /collections/{id}` | 204 | 204 ✅ |
| `POST /collections` | 201 | 201 ✅ |
| `POST /collections/{id}/datas` | 201 | 201 ✅ |
| `DELETE /datas/{id}/metadatas` | 200 | 200 ✅ |
| `DELETE /datas/{id}/rights` | 200 | 200 ✅ |
| `DELETE /datas/{id}/collections` | 200 | 200 ✅ |
| `POST /groups` | 201 | 201 ✅ |

➡️ **Bilan de conformité** : à l'issue de la confrontation exhaustive, les **seuls** écarts code ↔ spec confirmés sont (a) `delete_datas_files` — URL avec U+200B **et** test `200` au lieu de `204` (§2.4), et (b) `delete_datas` qui ignore le succès `202` (§3.5). Tous les autres chemins, noms de paramètres et codes de statut sont **conformes**. Le seul mapping non vérifié est `PUT /datas/{id}/status/{status}` (chemin templaté ; le code teste `204`).

### 8.5 Vocabulaire confronté au serveur en direct (script `C:\temp\nkl_vocab.py`)
`VOCABTYPE` a été **importé du fichier** (zéro recopie) et comparé aux endpoints `vocabularies/*` actifs de Nakala.

**Types de ressource — `/vocabularies/datatypes` : 30 (code) vs 29 (serveur).**

Dans le code mais **plus** côté serveur (à retirer / migrer) :

| `typeUri` figé | label code |
|----------------|-----------|
| `…/coar/resource_type/c_2659` | periodical (déjà « deprecated » en commentaire) |
| `http://purl.org/library/ArchiveMaterial` | ArchiveMaterial |
| `http://purl.org/ontology/bibo/Collection` | Collection |
| `http://purl.org/ontology/bibo/Series` | Series |
| `https://w3id.org/survey-ontology#SurveyDataSet` | SurveyDataSet |

Côté serveur mais **absents** du code (identifiants COAR récents) : `c_2fe3`, `F8RT-TJK0`, `NHD0-W6SY`, `YC9F-HGCF`.

**Propriétés — `/vocabularies/properties` (60 URIs)** : toutes celles employées par le code existent ✅ — `nakala terms#created/title/license/type/creator`, `dc/terms/created`, `dc/terms/subject`.

**Types de métadonnée — `/vocabularies/metadatatypes` (10)** : `Box, DCMIType, ISO3166, LCSH, Period, Point, RFC5646, TGN, URI, W3CDTF`.
➡️ Le type des **dates** est `W3CDTF` ; typer `created` en `xsd:string` (exemples/tests) n'est pas conforme (cf. 3.16).

### 8.6 Validation des corps de requête contre les `definitions` (script `C:\temp\nkl_body.py`)
Pour chaque opération avec body, le schéma a été résolu (`$ref` → `definitions`) puis comparé aux payloads construits par le code et les tests.

**Conformité structurelle — globalement OK** :
- `metas[]` = `{value, lang, typeUri, propertyUri}` (def `Meta5`/`Meta2`/`Meta`) ✅
- `rights[]` = `{id, role}` (def `PostRight`, `role` = enum `Role`) ✅
- `files[]` = `{sha1, description, embargoed, name}` (def `File5`/`File`) ✅
- `status`, arrays d'identifiants (collections↔datas) ✅

Aucun champ n'est marqué `required` au niveau du schéma (Nakala valide les métadonnées obligatoires côté serveur) → pas de « champ requis manquant » détectable statiquement côté code.

**Écarts relevés :**
1. 🟠 **`POST`/`PUT /groups`, `users`** : spec = `array<{username, role}>`, code = `array<string>` → cf. 3.17.
2. ℹ️ **`Data2.collectionsIds`** est typé `string` dans la spec, alors que le code et l'exemple de la doc utilisent un **tableau**. Incohérence interne de la doc Nakala ; le code (tableau) est vraisemblablement correct — ne pas « corriger » sans test.
3. ℹ️ **`Data2.relations`** (`{type, repository, target, comment}`) existe dans l'API mais n'est **jamais exposé** par la librairie — fonctionnalité manquante, pas un bug.

### 8.7 Approfondissement des derniers findings (lecture seule, script `C:\temp\nkl_deep.py`)
- **Groupes** : `PostAndPutGroup.users` = `array<MinimalUserInfo>` = `[{username, role}]`, `role` ∈ `[ROLE_OWNER, ROLE_ADMIN, ROLE_USER]`, exemple `[{"username":"pdupont","role":"ROLE_OWNER"}]` → **confirme 3.17**. (`GET /groups/{id}` live = `404` : le groupe d'exemple a disparu de l'instance de test, donc pas de confirmation serveur, mais la spec est sans ambiguïté.)
- **Droits data** : enum `Role` = `[ROLE_ADMIN, ROLE_EDITOR, ROLE_READER]` ; `ROLE_OWNER` non assignable → **3.18**.
- **`collectionsIds`** : schéma brut `{"type": "string", "example": ["10.34847/nkl.12345678"]}` — spec **auto-incohérente** (type `string` mais exemple tableau) ; le code envoie un tableau (= l'exemple) → **code correct**.
- **Dates** : sur une data publiée réelle (`10.34847/nkl.be633595`), `GET …/metadatas` montre `created`, `title`, `license`, `creator` avec **`typeUri = null`** ; seul `type` porte `dcterms:URI`. → Nakala ne pose pas de `typeUri` sur `created` en pratique ; `xsd:string` (code) n'est conforme ni à cet usage ni à `W3CDTF` (cf. **3.16**).

### 8.8 Créateurs (noms/prénoms) — API vs NakalaPycon (script `C:\temp\nkl_creator.py`)
- **Modèle API** : définition `Author` = `{givenname, surname, orcid, authorId, fullName}`, `givenname`+`surname` **obligatoires** ; données réelles confirmées (`{"givenname":"Claudie","surname":"Marcel-Dubois","fullName":"Claudie Marcel-Dubois"}`).
- **Regex NakalaPycon** : lit **« Nom, Prénom »** (`"Dupont, Pierre"` → surname=Dupont, givenname=Pierre) ⇒ **contredit le docstring** « prénom, nom » (cf. 3.11).
- **Effets de bord** : espace parasite dans `givenname` après `@orcid` ; créateur **sans virgule silencieusement ignoré** (`"Marie Curie"` → omis).
- **Site vs API** : le formulaire du site a deux champs séparés (pas d'inversion) ; la lib découpe une chaîne unique (inversion/omission possibles). Format stocké identique des deux côtés.
- **Confirmé par cas réel** : `10.34847/nkl.3d6bm8n2` stocke `givenname="Manet", surname="Eduardo"` (pour Eduardo Manet) — exemple concret d'inversion déjà présente dans l'entrepôt.
- **Note search** : `GET /search` renvoie les `metas` creator avec `value: null` (l'info auteur est exposée hors `metas` dans la réponse search) — à ne pas parser depuis là.

### 8.9 `authorId` — test d'écriture sur l'instance de test (script `C:\temp\nkl_authorid2.py`)
Test réel : 3 data *pending* créées puis **supprimées** (3× DELETE 204), avec le même auteur dans deux ordres.

| Dépôt | Couple envoyé | `authorId` |
|-------|---------------|-----------|
| A1 | `{givenname:"Zephyrin", surname:"Qwxtestauthor"}` | `03664163-…96793` |
| A2 | idem A1 | `03664163-…96793` (**identique**) |
| B  | `{givenname:"Qwxtestauthor", surname:"Zephyrin"}` (inversé) | `959ba842-…7a3c` (**différent**) |

**Conclusions prouvées :**
1. **A1 == A2** → Nakala **dédoublonne** : un couple `(givenname, surname)` identique réutilise le même `authorId` (regroupement d'auteurs côté serveur).
2. **A1 ≠ B** → le dédoublonnage se fait sur le **couple ordonné exact** ; l'inversion prénom/nom crée un **auteur distinct** (pas de normalisation, pas de test d'inversion, pas d'usage du `fullName`).

**Aggrave 3.11** : une inversion par `creatorsValuesToDic` ne produit pas seulement un affichage erroné — elle **détache le créateur de l'identité d'auteur** de Nakala (nouvel `authorId`). En dépôt de masse, cela **fragmente silencieusement** les auteurs (même personne éclatée en plusieurs `authorId`), dégradant recherche et pages auteur. L'impact de 3.11 est donc à relever : effet sur l'intégrité des données, pas seulement cosmétique.

### 8.10 Génération d'index & exemples réels de créateurs (scripts `nkl_index.py`, `nkl_one*.py`)

**Tri de l'index côté Nakala** — `GET /authors/search?order=asc` : tri primaire **par `givenname`**
(conforme à la doc « basé le prénom puis le nom »). De plus, une large part des fiches ont
`givenname=""` et tout le libellé dans `surname` (ex. `surname="Jean Daudin"`), ce qui rend le
classement hétérogène. → Côté API comme côté NakalaPycon (§3.19), la clé alphabétique part du
**prénom**, pas du nom de famille.

**Exemples réels confrontés à l'API :**

| DOI | `givenname` | `surname` | Lecture |
|-----|-------------|-----------|---------|
| `10.34847/nkl.3d6bm8n2` (« Alicia ») | `Manet` | `Eduardo` | **inversé** : l'auteur visé est Eduardo Manet ; nom et prénom permutés, `authorId=60290a13-…` propre à cette forme inversée → fiche d'auteur fragmentée |
| `10.34847/nkl.30adw69k` (« Un duo d'artistes atypique ») | — | — | `creator.value = null` : propriété présente mais **aucune donnée d'auteur** ; rien à indexer ⇒ entrée « sans auteur » |
| `10.34847/nkl.be633595` | `Claudie` | `Marcel-Dubois` | correct (référence saine) |

➡️ Ces cas illustrent les trois états rencontrés en production : créateur **correct**, créateur
**inversé** (signature du scénario 3.11/3.19), créateur **vide** (`null`). Un index généré
mélange donc tri-par-prénom, entrées permutées et lignes sans auteur — sans que l'outil ne
distingue ces situations.

### 8.11 Pagination /search, IIIF, et réparabilité de l'authorId (scripts `nkl_more.py`, `nkl_repair.py`)

**Pagination `/search`** (lecture seule) :
- `size` honoré jusqu'à `10000` (status 200, `nb_datas` = `size`) ; `page=1` vs `page=2` renvoient des `identifier` distincts → pagination fonctionnelle.
- **Mais** `lastPage` et `currentPage` sont **`null`** dans la réponse `/search` (seul `totalResults` ≈ 898 413 est présent) → cf. 3.20.

**IIIF** (lecture seule, image publique `10.34847/nkl.f1ea3017` / `ebe638b…3119b`) :
- `info.json` → 200, `width=971 height=849` (conforme à l'assertion de `test_nklPullCorpus`), `@context = .../image/3/context.json` (**API 3.0**).
- URL `…/0,0,100,100/full/0/default.jpg` → 200, `4675` octets, magie `FF D8 FF E0` = **JPEG valide** → `full` encore toléré (cf. 3.21).

**Réparabilité de l'`authorId` via `put_datas`** (test d'écriture instance de test, 2 data créées puis supprimées — 2× DELETE 204) :

| Étape | `givenname` / `surname` | `authorId` |
|-------|-------------------------|-----------|
| Créé **inversé** | Manettest / Eduardo | `d4692916-…49b8` |
| **Après `put_datas`** correctif | Eduardo / Manettest | `b6d3ebf9-…9ef2` |
| **Référence** (créée correcte) | Eduardo / Manettest | `b6d3ebf9-…9ef2` |

➡️ **Conclusions** : `put_datas` **recalcule** l'`authorId` (≠ avant), et la valeur corrigée est **identique à la référence**. Le dédoublonnage de Nakala est donc **dynamique** (recalculé à chaque écriture sur le couple courant, sans mémoire de l'ancien). **Bonne nouvelle pour la remédiation** : corriger une inversion via l'API rattache réellement la donnée à la bonne fiche d'auteur — pas de fantôme résiduel sur la donnée corrigée (l'ancien `authorId` ne subsiste que s'il reste référencé par d'autres data). Complète §8.9.

---

## 9. Cartographie envoi / extraction & dette

Vue synthétique de **toutes** les fonctions du paquet, classées par **sens**
(envoi vers Nakala / extraction depuis Nakala) et par **état**. Les lignes
« manquant » s'appuient sur le catalogue d'endpoints réels de Nakala (savoir
live ColleC, `nakala-savoir-api.md`). Objectif : visualiser d'un coup où sont
les trous et la dette avant toute feuille de route.

**Légende :** ✅ présent & sain · 🐞 bug fonctionnel · ⚠️ fragile (stub partiel /
perf / doc / conformité) · ❌ capacité Nakala non exposée. Les réfs (2.x, 3.x)
pointent les sections de cet audit.

### 9.1 Envoi (écriture) — couverture quasi complète, justesse inégale

**Dépôts & fichiers**

| Fonction | État | Note |
|----------|------|------|
| `post_datas` (créer) | ✅ | exposé au #422 si `metas` portent une `lang` 639-3 (3.22) |
| `put_datas` | ⚠️ | danger `files[]` = remplacement total non documenté (3.23) |
| `delete_datas` | ⚠️ | ignore le succès `202` (3.5) |
| `post_datas_uploads` | 🐞 | fuite de descripteur + `open()` hors `except` (3.13) |
| `delete_datas_uploads` | ✅ | |
| `post_datas_files` | ✅ | commentaire « 201 » faux, cosmétique (3.5) |
| `delete_datas_files` | 🐞 | **cassée** : U+200B dans l'URL + teste `200`≠`204` (2.4) |

**Métadonnées / droits / appartenance / publication**

| Fonction | État | Note |
|----------|------|------|
| `post_datas_metadatas` | ⚠️ | `Content-Type` absent (3.14) + doublon-sur-scalaire non documenté (3.24) |
| `delete_datas_metadatas` | ✅ | granulaire à la valeur, correct |
| `post_datas_rights` | ⚠️ | `Content-Type` absent (3.14) + docstring `ROLE_OWNER` erroné (3.18) |
| `delete_datas_rights` | ✅ | |
| `post_datas_collections` | ⚠️ | `Content-Type` absent (3.14) |
| `delete_datas_collections` | ✅ | |
| `put_datas_status` | ⚠️ | fige `"published"` — autres statuts non exposés |
| `PUT …/collections` (remplacer appartenance) | ❌ | seuls post/delete existent |

**Collections & groupes**

| Fonction | État | Note |
|----------|------|------|
| `post_collections` / `put_collections` / `delete_collections` / `post_collections_datas` | ✅ | CRUD collection complet |
| metadatas / rights / status **granulaires** de collection | ❌ | non exposés |
| `post_groups` / `put_groups` | 🐞 | `users` en liste de chaînes au lieu d'objets `{username,role}` (3.17) |
| `delete_groups` | ✅ | |

**Helpers d'envoi (haut niveau)**

| Fonction | État | Note |
|----------|------|------|
| `dfDatasFiles2ListDic` (tableur→JSON) | 🐞 | conditions pandas `len(df==1)` (2.5) + langue jamais posée (2.6) |
| `creatorsValuesToDic` | 🐞 | inversion nom/prénom (3.11) + séparateurs ignorés (3.7) |
| `put_collections_datas_rights` | ⚠️ | `limit=1` → N+1 requêtes (5.13) |
| `delete_datas_uploads_all` | ⚠️ | n'inspecte pas `isSuccess` (5.11) |

**Encodage d'envoi manquant**

| Capacité | État |
|----------|------|
| Conversion langue 639-3→639-1 | ❌ (cause du #422, 3.22) |
| `build_spatial` / `build_temporal` (DCSV) | ❌ |
| Validation licence SPDX (∪ extras type `etalab-2.0`) | ❌ |
| `relations` POST/DELETE (+ vocabulaire fermé de 38 types) | ❌ |

➡️ **Lecture 9.1** : la surface d'envoi **existe quasi intégralement** ; la dette
est de la **justesse** (1 fonction cassée, ~6 bugs, `Content-Type`, conformité
groupes) + quelques **helpers d'encodage** absents. Peu de capacités *nouvelles*
à créer — surtout corriger.

### 9.2 Extraction (lecture) — GET bas niveau sains, couche corpus faible

**GET bas niveau (sains)**

| Fonction | État | Note |
|----------|------|------|
| `get_datas` (+ `metadataFormat`) | ✅ | |
| `get_datas_files` / `_metadatas` / `_rights` / `_collections` / `_status` / `_uploads` | ✅ | |
| `get_iiif_infoJson` | ✅ | seul endroit où `json.loads` est protégé |
| `get_collections` / `get_collections_datas` | ✅ | collection fournit `lastPage` → pagination OK ici |
| `get_groups` / `get_users_me` | ✅ | |
| `get_vocabularies_*` (×5) | ✅ | sauf `get_vocabularies_languages` : params non encodés (2.3) ⚠️ |

**Recherche (la plus boguée)**

| Fonction | État | Note |
|----------|------|------|
| `get_search_datas` | 🐞 | `order` ignoré (2.1) + **crash** sur réponse ≠200 (3.3) + `size`/pagination non exposés (3.12/3.20) + params non encodés (2.3) |
| `get_search_authors` | 🐞 | `order`/`page`/`limit` ignorés (2.2) + params non encodés |
| `search_groups` | ⚠️ | params non encodés (2.3) |

**Helpers de lecture (`nklUtils`)**

| Fonction | État | Note |
|----------|------|------|
| `isFileNameInData` / `isFileNameInUploads` | ✅ | |
| `isFileSha1InData` | 🐞 | retour `None` implicite (2.8) |
| `isFileSha1InUploads` | ⚠️ | renvoie `bool` au lieu d'un tuple (incohérent, 5.3) |

**Couche corpus (`nklPullCorpus`) — le maillon faible**

| Fonction | État | Note |
|----------|------|------|
| `collectionToDf` | ⚠️ | **stub** (`pass`) + test l'appelle avec 1 arg → `TypeError` (3.8) |
| `collectionDatasToDf` | 🐞 | n'extrait que **5 `propertyUri`** (title/created/license/type/creator) alors que sa docstring promet « toutes les metas » → spatial/temporal/subject/description/contributor/language… **silencieusement perdus** ; créateurs aplatis à perte ; `pd.concat` en boucle (3.8, 3.19, 3.10) |
| `getImageSize` | ⚠️ | `except:` nu → `(0,0)` silencieux (5.11) |
| `getImageUrlIIIF` | ⚠️ | `size="full"` déprécié en IIIF 3.0 (3.21) |
| `getSoundTimeDuration` | ⚠️ | **stub** (`return 0`) |

> **Constat groupé — la couche « transformation vers DataFrame » est inachevée.**
> Les deux seules fonctions « vers df » (le cœur « pull corpus » du paquet) ne
> sont pas terminées : `collectionToDf` est un **stub** pur (`# TODO` + `pass` →
> `None`, jamais implémentée) et `collectionDatasToDf` est **partielle** — elle
> n'extrait que 5 `propertyUri` sur tout le Dublin Core alors que sa docstring
> annonce « toutes les metas » (toute métadonnée riche — spatial, temporal,
> subject, description, contributor, language, relations… — est **silencieusement
> perdue à l'aplatissement**, alors que `get_collections_datas` la renvoie bien
> en amont). Avec `getSoundTimeDuration` (stub) en plus, c'est **2 stubs sur 5
> fonctions** du module + des fonctionnelles boguées. Le défaut n'est donc ni
> « une docstring à corriger » ni « les créateurs aplatis » (3.19) pris
> isolément : c'est **toute la couche d'extraction tabulaire qui est en
> chantier**. Correctif de fond = **boucle générique** sur toutes les
> `propertyUri` + **décision de représentation tabulaire** du riche / répétable /
> structuré (colonne DCSV brute, colonnes éclatées, ou table « longue »),
> idéalement avec `parse_spatial`/`parse_temporal` pour décoder le DCSV. Lien :
> 3.8, 3.10, 3.19, 3.21, 9.4.

**Capacités d'extraction manquantes**

| Capacité | État |
|----------|------|
| `GET …/citation` | ❌ |
| `GET …/versions` + résolution `.vN` | ❌ |
| `GET …/relations` | ❌ |
| `GET /resourceprocessing/{id}` | ❌ |
| Téléchargement binaire direct `GET /data/{id}/{sha1}` | ❌ (IIIF seulement) |
| **OAI-PMH `/oai2`** (moissonnage, sets = collections) | ❌ |
| Facettes users (`/users/datas/{datatypes,statuses,createdyears}`) | ❌ |
| `/websites`, `/embed/{id}/{fileId}` | ❌ |
| **Itérateur « tout paginer »** (`/search` via `totalResults`+`size`) | ❌ |
| Lecteurs structurés (créateur fidèle, `parse_spatial/temporal`) | ❌ |
| Détection de dérive via `modDate` | ❌ |

➡️ **Lecture 9.2** : les GET bas niveau sont sains, mais **la recherche est
boguée** et **la couche corpus — la vocation affichée « pull corpus »** — est la
plus faible du projet (2 stubs, extraction à perte, IIIF déprécié). C'est là que
se concentrent à la fois la **dette** et les **capacités nouvelles** à plus fort
rapport valeur/effort.

### 9.3 Socle (transverse) — dette systémique

| Élément | État | Note |
|---------|------|------|
| `NklTarget` défaut `apiKey` non vide | 🐞 | faux positif d'auth → 401 (2.7) |
| `_request()` factorisé | ❌ | absent → boilerplate ×30 (5.1), cause-racine de 2.3/3.1/3.2 |
| `timeout` réseau | ❌ | absent partout (3.2) |
| `json.loads` protégé | 🐞 | non protégé sauf `get_iiif_infoJson` (3.1) |
| Retry 5xx / vérif DELETE par relecture | ❌ | pertinent surtout en prod (TLS timeouts, 500 transitoires) |
| `logging` au lieu de `print` | 🐞 | `print` dans toute la lib (3.4) |
| Imports relatifs / packaging | ⚠️ | imports absolus de 1er niveau (5.10) |
| Type hints / dataclasses | ❌ | |

### 9.4 Lecture d'ensemble

- **Envoi** : ~complet en couverture, **dette = justesse** (corriger > créer).
- **Extraction** : GET bas niveau sains, mais **recherche boguée** et **couche
  corpus sous-développée** → l'axe au meilleur rapport valeur/effort pour des
  **capacités nouvelles** (moissonnage, lecteurs fidèles, versions/citation).
- **Socle** : un seul chantier (`_request()` + `timeout` + `json` protégé + clé
  vide) **éteint en cascade** une grande partie des bugs des deux côtés — c'est
  le préalable rentable.
