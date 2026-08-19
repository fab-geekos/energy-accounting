# energy-accounting — mémoire du projet

Jauge d'énergie personnelle de Fabien. **Un seul `index.html`** (HTML/CSS/JS inline),
sans build. Hébergé par **GitHub Pages** sur https://fab-geekos.github.io/energy-accounting/
(dépôt public `fab-geekos/energy-accounting`). Données dans **Firebase** (Google Auth +
Firestore), avec le `localStorage` en cache.

## Trois onglets, un seul document

`🌅 Matin` (questionnaire à six familles) · `⚡ Journée` (drains et recharges) ·
`📊 Données` (deux courbes et l'historique). L'onglet actif est dans l'ancre de l'adresse
(`#matin`, `#journee`, `#donnees`).

`energy.html` n'est plus qu'une **redirection** vers `index.html#journee`. Elle existe
uniquement pour les anciens favoris. Ne pas la supprimer sans vérifier qu'aucun lien ne
pointe dessus.

⚠️ **Convention de nommage à tenir.** Les deux pages d'origine avaient chacune leurs
`gaugeValue`, `results`, `saveBtn`, `calculate()`. Réunies dans un document unique, ces noms
entraient en collision et la dernière définition gagnait en silence. Tout ce qui appartient à
un onglet est donc préfixé : `m*` Matin, `j*` Journée, `d*` Données. Ce qui n'est pas préfixé
est délibérément commun.

## Le modèle de données

```js
energyData[date] = {
  morning: { gauge, details, at },   // jauge du réveil
  day:     { gauge, start, at },     // jauge de fin de journée
  del:     <horodatage>              // pierre tombale, si la journée a été supprimée
}
```

Firestore : **un seul document**, `users/{uid}/energy/state`, avec tout sous la clé `days`.
Un document et non un par jour, pour que l'app se charge en une lecture au lieu d'une par
journée d'historique. La limite de 1 Mo par document représente des décennies de jauges.

⚠️ **Ne jamais revenir à `data[date] = {...}`.** Les deux pages d'origine écrivaient toutes
les deux à la racine de la journée et **s'écrasaient mutuellement** : la dernière sauvegarde
détruisait l'autre, et l'historique mélangeait silencieusement deux grandeurs. Toute écriture
passe par `saveToday(branche, valeur)`, qui ne touche que sa branche.

## La synchronisation multi-poste

C'est la raison d'être du passage à Firebase : la jauge du matin est saisie à la maison, la
comptabilité de la journée au bureau. Quatre mécanismes, chacun réglant un problème précis.
Aucun n'est décoratif.

1. **Fusion branche par branche**, jamais « le plus récent gagne » globalement. `mergeJours`
   prend l'union des dates et, pour chaque branche, la version au `at` le plus grand. Deux
   machines qui remplissent des choses différentes le même jour conservent les deux.
   La fonction est **commutative et idempotente**, ce qui garantit que l'échange local/nuage
   converge au lieu de boucler.
   ➡️ C'est ce que `todo` n'a pas : sa mémoire de projet note que le nuage y écrase toujours
   le local faute de réconciliation. Acceptable là-bas, fatal ici.

2. **Horloge logique** (`maintenant()`, `observerHorloge()`). `Date.now()` est l'horloge
   locale, et deux machines ne sont jamais à la même heure. Un poste en retard écrirait dans
   le passé et perdrait l'arbitrage **alors qu'il a la donnée la plus récente**. L'horodatage
   est donc borné par tout ce que la machine a vu passer, nuage compris.

3. **`set(..., { merge:true })`**, jamais un `set()` nu. Un `set` ordinaire remplace le
   document entier : une écriture mise en attente hors ligne, calculée sur une vue périmée,
   écraserait ce que l'autre machine a écrit depuis. Avec `merge`, Firestore fusionne les
   cartes clé par clé sur le serveur.

4. **Pierres tombales.** Supprimer une journée écrit `{ del: horodatage }` au lieu d'enlever
   la clé. Sans ça, l'autre machine, qui possède encore la donnée, la ressusciterait au
   premier contact.

Écriture : `persist()` sert le `localStorage` immédiatement et Firestore 600 ms plus tard.
⚠️ `pousser()` est aussi branché sur `visibilitychange` et `pagehide` : sans ce filet, une
modification faite juste avant la fermeture de l'onglet n'atteint jamais le nuage, et la
prochaine ouverture ressuscite l'ancienne version. Le manque avait coûté des données sur `todo`.

**Comportements assumés**, à ne pas « corriger » : deux comptages de la même journée sur deux
machines ne se cumulent pas, le dernier gagne (les additionner produirait un chiffre faux) ;
une branche éteinte par une pierre tombale reste physiquement dans le document Firestore, elle
est filtrée à la lecture.

## Sécurité

- Projet Firebase dédié : `energy-accounting-436bc`, région européenne.
- L'`apiKey` du code est **publique par nature**, elle ne fait qu'adresser le projet. Vérifié
  empiriquement : lecture et écriture avec cette clé mais sans session renvoient
  `403 PERMISSION_DENIED`.
- La protection réelle est la **règle Firestore**, qui exige une session, un `uid`
  correspondant au chemin, et l'email `fabien.labbe.91@gmail.com`.
- Chart.js est chargé avec son empreinte `integrity`. ⚠️ Elle est liée à la version : en
  changeant 4.4.1, reprendre l'empreinte sur api.cdnjs.com, sinon le script ne charge plus et
  les graphes disparaissent sans message.
- Le dépôt reste **public** : GitHub Pages gratuit l'exige, et rien de sensible n'est dans le
  code. Ne pas suivre l'étape « dépôt privé » du skill `site-perso`, qui suppose un
  hébergement chez Firebase.

## Déploiement

Push sur `main`, GitHub Pages construit tout seul. **Pas** de GitHub Action, **pas** de
Firebase Hosting, **pas** de secret `FIREBASE_SERVICE_ACCOUNT` : ils n'ont de sens que pour
un hébergement Firebase.

⚠️ Si un déploiement semble ne jamais arriver alors que le commit est sur origin, le build
Pages peut être resté bloqué (déjà vu pendant une panne GitHub, coincé seize heures) :

```bash
gh api -X POST repos/fab-geekos/energy-accounting/pages/builds
```

## Avancement

- [x] Fusion des deux pages en trois onglets, avec séparation matin/journée
- [x] Reprise automatique de la jauge du matin dans l'onglet Journée
- [x] Empreinte SRI sur Chart.js
- [x] Firebase : Auth Google, Firestore, règles publiées et vérifiées de l'extérieur
- [x] Synchronisation multi-poste avec fusion, horloge logique et pierres tombales
- [ ] Affiner les barèmes, jugés « grossiers » par Fabien
- [ ] `energyData_avant_fusion` (copie de sauvegarde locale d'avant la fusion) pourra être
      supprimée quand Firebase aura fait ses preuves

## Règles de collaboration

- Committer et pousser à la fin de chaque lot, Fabien teste directement en ligne.
- Commits directs sur `main`, c'est la branche servie.
- Pas de tiret cadratin dans les textes produits pour Fabien.
