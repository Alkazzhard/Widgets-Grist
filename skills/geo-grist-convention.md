# Convention « Données géo dans Grist » (v1)

**Contrat partagé** de la gamme géospatiale, autour de **Grist comme source de
vérité**. Trois briques, une seule convention :

| Brique | Rôle | Sens |
|---|---|---|
| **QgisRemoteMCP** (serveur, PyQGIS) | exporte des livrables natifs (`.qgz`, GPKG, stylés) **et** vers Grist ; automatisation IA | QGIS ↔ Grist (lourd) |
| **qgis2grist** (navigateur) | importe les exports QGIS (qgis2web / `.qgz` / GPKG / QField / QML) → tables Grist | QGIS → Grist (interactif) |
| **Atlas** (navigateur) | lit / rend (2D+3D) / édite / filtre / raconte (storymaps) ; export léger | Grist ↔ carte |

But : **canva universel** cartes + projets QGIS, standardisé et réversible. Les deux
entrées (QgisRemoteMCP serveur, qgis2grist navigateur) doivent **écrire les mêmes
tables standard** → Atlas est agnostique de la source.

> Statut : **v1 — implémenté côté Atlas** (lecture/liaison, auto-entablement,
> config par objet, contrôles, storymaps). Reste à aligner formellement
> QgisRemoteMCP/qgis2grist sur ce contrat (cf. §11). Faire évoluer ce doc, pas le
> code en silo ; **versionner** (bump l'entête à chaque changement de contrat).

---

## 1. Principe

- Une **couche** = une *vue* sur une **source**. Deux natures de source :
  - `kind: 'blob'` — la couche porte son GeoJSON (dessins, imports rapides). Atlas
    fonctionne **tel quel**, sans Grist.
  - `kind: 'table'` — la couche est **liée** à une table « 1 ligne = 1 objet ».
    Atlas construit la FeatureCollection **à la volée** depuis la table (la jointure
    vit dans le widget, voir §6), édite **ligne par ligne**, et peut garder un blob
    en **cache** (standalone / perf / snapshot).
- La donnée géo stockée est toujours en **WGS84 (EPSG:4326)**.

---

## 2. Table de données « 1 ligne = 1 objet »

Une table par couche. Une ligne = un objet. Colonnes :

| Colonne | Rôle | Statut qgis2grist |
|---|---|---|
| `geometry_json` | géométrie **GeoJSON (string)**, WGS84 | ✅ écrit |
| `latitude` / `longitude` | commodité pour les points | ✅ écrit |
| *attributs* (typés) | données ; `Choice` (ValueMap), `Ref`/`RefList` (relations) | ✅ écrit |
| `fill_color` | couleur **par objet** (hex), dérivée du renderer QGIS | ✅ écrit |
| `stroke_color`, `size`/`radius`, `opacity` | style par objet (optionnel) | ➕ à ajouter |
| `height`, `elevation` | 3D : extrusion / altitude (optionnel) | ➕ |
| `model_id` | modèle 3D par objet : id du catalogue intégré, ou URL custom | ➕ |
| `model_glb` | modèle 3D **en pièce jointe** sur l'objet (type *Attachments*) — **prioritaire** | ➕ |
| `label` *(ou champ désigné)* | étiquette | ➕ |
| `hidden` | masque l'objet | ➕ |

**Détection de la géométrie** (ordre) : `geometry_json` → `geometry` → `geom` →
`wkt` (parse) → paire `latitude`+`longitude` (alias `lat`/`lng`/`lon`).

**Identité** : le **rowId Grist** est la clé stable (sélection, édition, write-back).

**Override = la colonne elle-même.** Chaque colonne d'attribut d'affichage **EST**
l'override par objet : une valeur non nulle écrase le défaut de couche (cf. §2bis).
On ne remplit que ce qu'on personnalise (stockage *sparse*, efficace).

---

## 2bis. Résolution des paramètres (défaut couche → override objet)

Pour chaque paramètre d'affichage (`model`, `scale`, `rotation_*`, `offset_*`,
`height`, `fill_color`, `label`, `hidden`), précédence du plus spécifique au plus
général :

1. **Override par objet** — valeur de la colonne de l'attribut (couche `table`) ; ou
   `feature.properties._*` (couche `blob`).
2. **Liaison à un champ** — symbolisation catégorisée/graduée (param ↔ champ).
3. **Défaut de couche** — `style.common` / `style.library`.
4. **Défaut du modèle** — params portés par le modèle (catalogue).

**Modèle 3D** : `model_glb` (PJ objet) > `model_id` (catalogue / URL) > liaison
champ > défaut couche > défaut modèle.

**Chargement d'un modèle en PJ** : lire l'id d'attachment de la cellule →
`getAccessToken()` → `{baseUrl}/attachments/{id}/download?auth={token}` →
`GLTFLoader`. Modèle mis en **cache** (chargé une fois) ; token court → pris à la
volée au chargement.
**Upload depuis Atlas** : pas de méthode plugin → REST `POST {baseUrl}/attachments`
avec le token (⚠️ à confirmer que le token autorise l'écriture ; sinon repli :
attacher via l'UI Grist, Atlas lit).

**Instancing** : un modèle partagé entre plusieurs objets = **même URL / même id
d'attachment** → un seul `InstancedMesh` (perf conservée). Pour appliquer un modèle
à toute une couche/sélection, on propage le même id.

---

## 3. Registre de couches

Table décrivant **ce qui est une couche géo et comment l'afficher**. C'est la
généralisation de `Maquette_Layers`. Atlas n'affiche **que les couches déclarées
ici** ; les autres tables géo du document sont seulement *proposées « à lier »*.

Une ligne par couche :

| Champ | Rôle |
|---|---|
| `name` | nom de la couche |
| `kind` | `'blob'` \| `'table'` |
| `sourceTable` | **id de la table** liée (texte ; résolu par Atlas — pas un Ref Grist, cf. §7) |
| `geometryColumn` | nom de la colonne géométrie de la source |
| `geomType` | `Point` \| `Line` \| `Polygon` (+ Multi) |
| `geojson` | blob FeatureCollection — source si `kind:'blob'`, **cache** si `'table'` |
| `styleJSON` | style portable (cf. §4) |
| `color`, `visible`, `order`, `group` | défaut couleur, visibilité, ordre, groupe (arbre QGIS) |

---

## 4. Style portable (`styleJSON`)

Superset commun **QGIS QML ↔ symbo Atlas**. Forme :

```json
{
  "mode": "single | categorized | graduated | model",
  "field": "<champ pour categorized/graduated>",
  "color": "#RRGGBB",                         // single
  "categories": [{ "value": "...", "color": "#...", "modelId": "..." }],
  "ranges":     [{ "lower": 0, "upper": 10, "color": "#..." }],
  "size":   { "mode": "fixed|graduated", "value": 8, "field": "...", "range": [0.5,3] },
  "height": { "field": "...", "value": 12 },  // extrusion / 3D
  "model":  { "modelId": "...", "field": "..." },
  "label":  { "field": "...", "enabled": true }
}
```

**Précédence** (du plus fort au plus faible) :
`styleJSON` de la couche (règle) **>** colonnes de style par objet (`fill_color`…,
le « cuit » QGIS) **>** défaut de couche.
→ Le rendu QGIS est fidèle par défaut, et reste modifiable dans Atlas.

**Mapping QML → styleJSON** (déjà parsé par qgis2grist) :
`singleSymbol → single`, `categorizedSymbol → categorized`,
`graduatedSymbol → graduated`.

---

## 5. Conversions réversibles (2 commandes)

- **Descendant — couche → table 1-1** (« Exploser en table ») : Atlas lit le blob,
  crée la table standard (§2), bascule la couche en `kind:'table'` liée.
- **Montant — table 1-1 → couche** (« Lier une table ») : l'utilisateur choisit une
  table géo ; Atlas crée la ligne de registre (binding + style détecté) et rend.

Les deux partagent ce contrat → import qgis2grist et dessins Atlas convergent.

---

## 6. Jointure & rafraîchissement

- La **jointure vit dans Atlas (JS)** : il lit n'importe quelle table via `docApi`
  et construit la FeatureCollection. (Grist ne permet pas une formule/Ref générique
  vers une table variable — cf. §7.)
- Édition `kind:'table'` → **write-back par ligne** dans les colonnes de §2.
- Rafraîchissement des tables liées : **au focus / retour d'onglet** (léger), pas de
  temps réel multi-tables.

---

## 7. Contraintes Grist (à respecter)

- Réactivité (`grist.onRecords`) **uniquement** sur la table mappée du widget → les
  autres tables liées se lisent via `docApi.fetchTable` (one-shot) + refresh §6.
- Une colonne `Ref`/`RefList` cible **une table fixe** → le lien couche→table est
  stocké en **texte** (`sourceTable`) et résolu par Atlas.
- `requiredAccess: 'full'` requis (lecture multi-tables + création/écriture).

---

## 8. Automatismes

**Ligne de sécurité** : *automatique* pour **lire / afficher / dériver / proposer** ;
*explicite (ou réversible + confirmé)* pour **écrire / créer / modifier** la donnée
Grist (création de table, write-back en masse, reprojection).

Automatique :
- montage des couches **déclarées au registre** ; détection des autres tables géo →
  proposées « à lier » ;
- détection colonne géométrie, type géo, champ étiquette ;
- **auto-fit** caméra sur l'emprise des données ;
- application du style « cuit » par objet ; sinon **symbo par défaut** (catégorisé sur
  texte peu varié, gradué sur numérique, palette auto) ;
- **contrôles auto-suggérés** depuis les champs (date → curseur temps/animation,
  numérique → plage, catégoriel → sélecteur), bornes/options déduites des données ;
- pose 3D sur le relief + re-calage (tuiles DEM, toggle terrain, fond, redimension) ;
- repli : modèle absent → cercle de hit ; DEM illisible → tuile plate.

Explicite / confirmé :
- création du registre, explosion blob→table, liaison de table ;
- write-back vers Grist ;
- **CRS non-WGS84** : reprojection best-effort si l'info de projection est dispo,
  **sinon alerte** (renvoi vers qgis2grist).

---

## 9. Contrôles (rack de filtres/animation) — implémenté

Par couche, un tableau `controls` (persisté dans `StyleJSON._controls` côté Grist,
et dans le projet JSON) :
```json
[{ "field": "...", "type": "time|range|select", "active": true,
   "min": <num|ts>, "max": <num|ts>, "values": ["..."] }]
```
- `time` (champ date), `range` (numérique), `select` (catégoriel) — **auto-détectés**
  depuis les champs. Filtrent **la carte 2D ET la 3D** (même prédicat). `time` anime.
- Précédence d'orientation : `styleJSON.orientation.field` lie l'azimut à un champ.

## 10. Récit / Storymaps — table `Atlas_Story`

Séquence d'étapes rejouables (présentations). Une ligne = une étape :

| Colonne | Rôle |
|---|---|
| `Step` (Int) | ordre |
| `Title` (Text) | titre de l'étape |
| `Description` (Text) | texte narratif |
| `StateJSON` (Text) | snapshot : `camera` (center/zoom/pitch/bearing), `projection`, `timeOfDay`, `date`, `terrain3D`, `layers[]` (visibilité + contrôles actifs) |

Atlas capture / rejoue (mode présentation). Éditable côté Grist. (Médias en PJ : à venir.)

## 11. Rôles & conformité (les 3 briques)

- **Écrivains** (QgisRemoteMCP, qgis2grist) : produisent des **tables géo conformes**
  (§2) — colonne géométrie WGS84, attributs typés, `fill_color` par objet optionnel,
  `model_id`/`model_glb` optionnels. **Même schéma quelle que soit la source.**
- **Lecteur/éditeur** (Atlas) : monte les couches déclarées, applique style/contrôles,
  édite par objet (write-back colonnes), n'invente rien hors convention.
- **Aller-retour** :
  - **léger / navigateur** = Atlas exporte **GeoJSON + QML** (style) → relisible QGIS.
  - **lourd / serveur** = QgisRemoteMCP reconstruit `.qgz`/GPKG natifs depuis Grist.
- **Auto-entablement** (Atlas) : tout import devient une table standard (la donnée de
  référence) ; repli blob hors Grist.
- **Suppression** (Atlas) : « retirer d'Atlas » (garde la table) vs « supprimer la
  table » (`RemoveTable`).
- **Test de conformité** (recommandé) : un écrivain doit pouvoir relire ce qu'il a
  écrit via le lecteur de référence (détection géométrie + types + style).

## 12. Hors-périmètre / à venir

- **GPKG** (binaire) et `.qgz` : réservés à QgisRemoteMCP (serveur).
- **Upload de PJ in-widget** : dépend du CORS de l'instance (sinon attache native).
- Médias storymap (images en PJ), transitions ; arbre de couches/groupes ; sélection
  inter-widget (clic carte ↔ ligne).
