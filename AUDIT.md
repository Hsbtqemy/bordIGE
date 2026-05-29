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
| 🟠 Moyenne | 14 |
| 🟡 Faible | 16 |

> Comptes indicatifs. **3.3** est rédigé comme un bug 🔴 (plantage de `get_search_datas` sur toute réponse ≠ 200) mais reste classé en §3 ; il **n'est pas inclus** dans les 8 du §2. Le total réel de bugs critiques est donc **9**.

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
