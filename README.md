# TeamStarterUC - Enzo Marti

## Introduction

Pour ce projet j'ai été amené à répondre à différentes questions dans le cadre de l'arrivée d'un nouveau CFO.
Pour ce faire j'ai à disposition 2 fichiers : un premier contenant un mapping de différentes informations géographiques des Etats-Unis, et un second contenant les déclarations de notes de frais déclinées en 12 attributs.

Entre l'exploration et les réponses aux questions, différentes technologies ont été utilisées : Python (Jupyter Notebook), SQLite DB Browser, SQL, Excel.

## Setup

Le script de création des tables et de chargement des données se basent sur des fichiers qui ont été transformés. En effet, concernant les déclarations notes de frais, les informations permettant de répondre aux questions n'étaient pas toutes directement accessibles. J'ai créé de nouvelles dimensions et ai nettoyé certaines données (le nettoyage n'est pas parfait - d'autres méthodes seront abordées plus tard) telles que :
- Création d'une dimension  _Quarter_, basée sur la date permettant d'obtenir le numéro du trimestre associé.
- Création d'une dimension _DayName_ permettant d'obtenir le jour de la semaine correspondant à la date.
- Après exploration du fichier Excel initial, j'ai pu observer que quelques dates ne correspondaient pas au même format que les autres (range de dates au lieu d'une date fixe, inversion du jour et du mois), elles ont été mises à jour.
- Enfin, j'ai créé 2 dimensions afin d'extraire le maximum d'informations concernant les états américains. La première précise si un caractère spécifique (_:_) est présent dans une des dimensions de base (pattern permettant d'identifier un grand nombre de lignes ayant de l'information sur les états). La seconde utilise une regex, grâce à l'identification de la première dimension, permettant d'extraire le bigramme de l'état. Grâce à cela, je serai capable plus tard de faire une liaison entre les 2 fichiers en entrée.

Une fois ces traitements faits, on exporte le dataframe en csv. Ainsi, nous avons nos 2 fichiers au format csv et donc nous pouvons les exploiter afin d'en faire des scripts de chargement SQL facilement implémentable dans SQLite (fichier **create_load_tables.sql**).

## Partie 1 - Questions

Les requêtes SQL et les résultats associés se trouvent dans le fichier Excel **answers.xlsx** ; j'aborderai ici les différentes hypothèses et remarques concernant chaque question.

### Q1 : Quels sont les employés qui “dépensent anormalement” par typologies d’activités sur l’année 2015 ?

L'enjeu ici est de trouver une méthode ou une formule permettant de définir le seuil d'une dépense _anormale_. 
Initialement, j'ai considéré la loi normale ; avec la moyenne et l'écart-type on peut définir des intervalles remarquables et avec la formule _moyenne + 2 * écart-type_ on peut définir un seuil permettant d'obtenir les 2,5% des données les plus hautes pour chaque catégorie.

Malheureusement, peu de fonctions sont disponibles en SQLite, il m'aurait fallu passer un certain temps pour implémenter une fonction calculant l'écart-type (+ faire les requêtes pour faire les différents calculs). J'ai trouvé une autre solution : les **window functions**, notamment la fonction _ntile()_ permettant de découper un résultat en x parties homogènes.
En triant les données par catégories et dépenses décroissantes, on sait qu'en utilisant la fonction ntile(20) on aura 20 parties distinctes et que chaque partie contiendra 5% des données ; étant donné que les données sont triées, le premier ntile contiendra toujours (au minimum, dépendant du nombre de lignes retournées par catégorie) les 5% des données les plus extrêmes.

Après observation des données, j'ai jugé que la meilleure dimension pour les _typologies d'activités_ était _Activity Short Description_ (renommée _ActivityLabel_ dans mes scripts) car toutes les lignes (sauf une) ont bien un combo **id d'activité - label d'activité** pertinent et propre. C'est donc la meilleure dimension pour répondre à cette question ; elle ne nécessite pas d'autres traitements.

```
SELECT ActivityLabel, EmployeeName, AmountGross
FROM (
	SELECT ActivityLabel, 
		   NTILE(20) OVER (PARTITION BY ActivityLabel ORDER BY AmountGross DESC) AS tile, 
		   EmployeeName, 
		   AmountGross
	FROM expenses 
	WHERE ActivityLabel IS NOT NULL
)
WHERE tile = 1
ORDER BY ActivityLabel, AmountGross DESC
```

### Q2 : Quels sont les 5 états, avec leur nom complet, dans lesquels les dépenses ont le plus augmenté entre Q1 et Q4 2015 ?

Première question où il est nécessaire de faire un lien entre les 2 fichiers. Grâce aux précédentes transformations, on peut utiliser le bigramme des états américains pour lier ces deux fichiers.

La méthode la plus commune pour répondre à cette question est le **taux d'évolution** ; en comparant les dépenses entre Q1 et Q4 on obtient un pourcentage sur lequel on applique un _order by_ et un _limit_ afin d'obtenir les 5 états où il y a eu le plus d'évolution.
La formule du taux d'évolution est : _((DépensesQ4-DépensesQ1) / DépensesQ1) * 100_.

```
SELECT StateCode, txEvolution
FROM (	
	SELECT q1.StateCode, (q4_total - q1_total) / q1_total * 100 AS txEvolution 
	FROM (
		SELECT StateCode, sum(AmountGross) AS q1_total
		FROM expenses
		WHERE Quarter = 1 AND length(StateCode) = 2 # pour éviter de sélectionner les données mal / pas encore nettoyées
		GROUP BY StateCode) q1 
	JOIN (
		SELECT StateCode, sum(AmountGross) AS q4_total
		FROM expenses
		WHERE Quarter = 4 AND length(StateCode) = 2
		GROUP BY StateCode) q4
	ON q1.StateCode = q4.StateCode)
ORDER BY txEvolution DESC limit 5
```

Comme précisé dans la requête, une méthode a été proposée pour extraire l'information sur l'état mais cette méthode n'est pas parfaite : j'extrais actuellement ~8400 lignes sur un total ~11100 lignes, soit ~75% des données à disposition. Ainsi le résultat est impacté ; par contre cette requête est tout de même pertinente car, si les données en entrées sont toutes nettoyées, elle répondra bien à la question.

### Q3 : Quels sont les top 10 “lieux ou restaurent” de dépenses en Q2 ?

Cette question m'a amené à de nouveau explorer les données afin d'identifier les dimensions et valeurs pertinentes pour répondre à cette question. 
J'ai pu voir (comme pour le bigramme des états), que la dimension _vendor comment_ contenait des informations complémentaires (voire différentes) à celles présentes dans la dimension _vendor name_.

J'ai identifié que les _ExpenseCategory_ contenant le pattern _meals_ permettent d'identifier des lignes pertinentes. De plus j'ai aussi observé que si la dimension _vendor name_ contenait un string en majuscule, c'est que ce dernier représentait une entreprise (il est possible de trouver un _vendor name_ similaire au _employee name_) ; on obtient ainsi :

```
SELECT DISTINCT VendorName
FROM (
	SELECT VendorName, sum(AmountGross) AS total 
	FROM expenses
	WHERE EmployeeName <> VendorName AND VendorName = upper(VendorName) AND Quarter = 2 AND ExpenseCategory LIKE '%Meals%'
	GROUP BY VendorName)
ORDER BY total DESC LIMIT 10
```

Ainsi ici je fais un double-check sur les majuscules et la différence entre _employee_ et _vendor_, mais comme dit plus haut, la dimension _vendor comment_ contient beaucoup d'informations sur les enseignes que je n'ai pas pu exploiter. Des méthodes seront proposées à la fin pour potentiellement mieux nettoyer les données et extraire les informations clés.

### Q4 : Quels sont les jours de la semaine avec le plus de dépenses, en nombre et en montants ?

J'ai pu, en Python, transformer les dates en jours de la semaine. La requête devient plus facile à produire : 
```
SELECT DayName, count(*) AS number, sum(AmountGross) AS amount
FROM expenses
GROUP BY DayName
ORDER BY sum(AmountGross) DESC
```

### Q5 : Quel est le prix moyen des dépenses de nuits d’hotels par région (south, west, …) ?

Avant de construire la requête on doit trouver l’élément différenciant permettant d’isoler les dépenses concernant les nuits d’hôtels. On arrive rapidement aux _expenses categories_ **LODG et LODGU**.

En regardant plus en détail on trouve bien des _vendor name_ concernant des hôtels pour **LODGU** par contre cela est moins clair pour LODG.
Ce dernier ne contenant qu’une centaine de lignes et l’identification étant plus compliquée, cette catégorie sera écartée.

```
SELECT Region, avg(AmountGross) AS AverageAmount
FROM expenses e LEFT JOIN us u ON e.StateCode = u.StateCode
WHERE ExpenseCategory = 'LODGU-Lodging' AND Region IS NOT NULL
GROUP BY Region
```

### Méthodes de nettoyages et divers

Comme précisé plus la dimension _vendor comment_ contient beaucoup d'informations : bigramme de l'état, nom de la ville, enseigne concernée par la note de frais. Je n'ai cependant pas pu extraire toutes ces informations.

J'ai pu identifier un pattern grâce au caractère _:_ permettant de localiser les lignes contenant de l'information, mais cette méthode n'est pas parfaite ; on peut avoir tout autant d'informations dans des lignes ne contenant pas ce pattern, ou encore le bigramme n'est pas en majuscule par exemple.
C'est encore plus complexe pour les enseignes (cf. Q3) car certaines enseignes sont spécifiées avec une sorte d'ID ou autre ne permettant pas une agrégation de toutes les lignes pour une même enseigne.
Certaines méthodes permettraient d'extraire plus d'informations : 
- Amélioration de la regex,
- Découpage de la cellule en ne gardant que les lettres (hors chiffres et caractères spéciaux),
- Utiliser des fonctions comme **SequenceMatcher** ou **python-Levenshtein** pour regrouper des strings se ressemblant (à tester avec différents seuils).

De plus j'ai à ma dispostion 3 métriques mais seule une a été utilisée car je manquais de connaissances métier sur ces dernières (OTP et MN). J'ai tout de même un peu exploré et j'ai quelques observations : 
-  _Gross_ peut être égal à _OTP_
	- Si tel est le cas alors _MN_ =~ 49% de _Gross_
- Si _Gross_ et _OTP_ diffèrent 
	- Alors _OTP_ correspond soit à ~35%, ~53,3% ou ~66,6% de _gross_

Cela pourrait-il être une forme de taxe ou de TVA ? 
J'ai aussi approfondi ce point et j'ai découvert que chaque état américain avait son propre taux de TVA.

## Partie 2 - Scalling

Lorsqu'il est question de répondre aux mêmes questions sur une base temporelle (hebdomadaire ici), la réponse est généralement la création d'un dashboard permettant de mettre en exergue les différentes réponses aux questions.
Ainsi j'ai produit ce sketch d'un potentiel dashboard avec les différentes réponses attendues :

<img title="Skecth CODIR dashboard" alt="CODIR dashboard" src="/images/visualisation.png">

- Les questions 1 et 3 seraient présentées via un tableau (on souhaite obtenir les lignes où il y a le plus de dépenses par ordre décroissant).
- La question 2 consiste à faire sortir les états où il y a eu le plus d'augmentation, pour ce faire je propose d'utiliser une carte des Etats-Unis, découpée par état, en utilisant une gradation de couleur permettant de mettre en avant les états où cette dernière était la plus importante (Looker Data Studio propose une implémentation _facile_ d'une carte par exemple) - _petite erreur mais la gradation peut évidemment dépasser 100%_.
- Pour la question 4 on a fait remonter une métrique par région. Vu qu'il y a 4 régions on peut proposer 4 étiquettes contenant l'information clé et pouquoi pas un taux d'evolution.
- Enfin pour la dernière question, un graphique présentant 2 métriques (histogramme et ligne) pour le nombre de lignes et le montant associé.

Il y a aussi la présence d'un filtre, pouvant être dynamique (semaine du jour actuel) ; chaque visualisation sera surement liée à une dimension temporelle qui lui est propre mais on peut facilement imaginer que le filtre puisse avoir un effet sur les différentes visualisations.

De plus, pour faciliter les réunions CODIR, il pourrait être possible d'automatiser l'envoi de mails de façon hebdomadaire pour présenter en quelques lignes les chiffres clés de la semaine en amont de la réunion, afin que chaque participant soit plus ou moins sensibilisé aux informations de la semaine (exemple de technologie : Flask-Mail).

Enfin, pour s'assurer que les données soient régulièrement mises à jour, on pourrait imaginer l'utilisation d'une plateforme de planification de flux comme Airflow. On pourrait imaginer une stack avec un projet Python, un repository Git et une infrastructure GCP qu'Airflow viendrait checker, exécuter de manière journalière avant de spécifier l'envoi des données dans une table dédiée au reporting.

## Partie 3 - Analyse personnelle

Les points d’analyse et conseils pouvant aider le CFO sont assez situationnels, notamment concernant le budget alloué.
Ainsi, chaque proposition sera doublée : une première sera peu onéreuse et la seconde considérera des moyens quasi illimités.

Une première méthode serait de contrôler les champs remplis par les employés (au moins sur les champs clés) : 
- Soit en mettant en place différentes règles dans un Excel afin de s'assurer que la date est bien remplie, que le vendor est clairement spécifié, ...
- Il pourrait aussi être possible de retirer cette partie manuelle de l'enregistrement des notes de frais et utiliser des applications dédiées comme par exemple un scan qui, une fois la photo prise ou le fichier ajouté, identifierait les informations clés et les enregistrerait en base.

Une seconde méthode consisterait à maitriser les dépenses des employées. Pour ce faire, on pourrait imaginer l'utilisation d'un moyen de paiement spécifique afin de contrôler les dépenses, mais aussi d'éviter l'avancement de frais comme par exemple via une carte spécifique plafonnée reliée au compte de l'entreprise. Dans un autre style il pourrait être intéressant d'ajouter une supervision humaine des différentes notes de frais afin d'identifier rapidement et efficacement les entrées pouvant poser soucis.
