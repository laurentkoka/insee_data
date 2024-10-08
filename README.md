
## Traitement des fichiers de décès depuis 1970, fournis par l'INSEE

Depuis 2019 l'INSEE met à disposition du public les fichiers des personnes décédées depuis 1970, établis à partir des informations reçues des communes dans le cadre de leur mission de service public
Ces fichiers sont proposés au format CSV. Pour un décès enregistré, les informations suivantes sont disponibles : nom, prénoms, sexe, date de naissance, code et libellé du lieu de naissance, date du décès, code du lieu de décès et numéro de l’acte de décès.
Voir : https://www.insee.fr/fr/information/4769950

A fin juillet 2023 (dernier fichier pris en compte: `Deces_2023_M07.csv`), les 60 fichiers mis en ligne constituaient un total de plus de 27 millions d'enregistrements (27 329 471 lignes). 

Ce projet propose un certain nombre d'outils de traitement de ces fichiers INSEE, dont la qualité est variable (doublons, qualité des informations: données manquantes ou erronées, incohérences de dates, identification des communes...)

A noter que les mêmes données sont disponibles sur data.gouv.fr (https://www.data.gouv.fr/fr/datasets/fichier-des-personnes-decedees/) dans un format différent. Ce projet utilise les fichiers proposés sur le site de l'INSEE. 

### Qualité des données : doublons 

Le nombre d'enregistrements uniques est en réalité inférieur de quelques centaines de milliers au nombre mentionné ci-dessus car les fichiers contiennent de nombreux doublons: 

 - des lignes identiques (doublons simples). Certains enregistrements sont ainsi dupliqués jusqu'à 40 fois. Comme nous le verrons par la suite, ce type de doublon n'est pas toujours identifié par les principales applications qui réutilisent les données de l'INSEE, et notamment https://deces.matchid.io/. Il a été identifié plus de 160 000 lignes à supprimer. 
 - des lignes non identiques mais pour lesquelles certaines colonnes sont identiques, permettant d'affirmer qu'il s'agit de la même personne (doublons complexes)

L'identification et le traitement des doublons représente un défi, au vu du volume des données manipulées. 

#### Traitement des doublons simples 

Un premier traitement en Python, grâce à la librairie Pandas, a permis de nettoyer chacun des fichiers de ses propres doublons simples.

Chaque fichier CSV peut être stocké dans une dataframe Pandas: 

    df = pd.read_csv('deces-1997.csv', sep=';', encoding='utf-8-sig') 
    
 Il est ensuite possible de détecter les doublons: 

    df_duplicates = df[df.duplicated()]

  Et de les supprimer: 

    df_cleaned = df.drop_duplicates()

Puis de ré-écrire le fichier CSV nettoyé sur le disque : 

    df_cleaned.to_csv('deces-1997-cleaned.csv', index=False, lineterminator='\n')

Ce traitement a été fait fichier par fichier et a permis de détecter 120 333 lignes présentes au moins deux fois dans le même fichier et de les supprimer. Les fichiers contenant les données nettoyées, et les fichiers contenant la liste des enregistrements en doublon sont stockés dans ce projet Github. Le nombre de doublons par fichier est présenté dans un tableau en annexe. 

Le problème s'est cependant avéré plus compliqué car les mêmes enregistrements figurent dans plusieurs fichiers différents : il y a donc beaucoup plus de doublons simples que les 120 333 lignes mentionnées ci-dessus. Ci-après un exemple issu des fichiers 1989 et 1990. Les attributs sont rigoureusement identiques. 

    AARIF	AISSA	1	1965/12/02	NANTERRE	75050		1989/12/29	92073	918	deces-1989.csv-ligne 357409
    
    AARIF	AISSA	2	1965/12/02	NANTERRE	92050		1989/12/29	92073	918	deces-1990.csv-ligne 71674

Ce type de doublon n'est souvent pas détecté par les applications réutilisant les données de l'INSEE : https://deces.matchid.io/search?q=AARIF+AISSA

Il a donc été procédé en trois temps : 
- étape 1 : concaténation de l'ensemble des fichiers CSV bruts fournis par l'INSEE
- étape 2 : transformation du fichier résultant en dataframe Pandas 
- étape 3 : identification et suppression des doublons simples grâce à Pandas (même méthode que précédemment)

Ceci a permis d'obtenir un fichier CSV résultant 27 168 192 enregistrements. Au total 161 279 lignes présentes deux fois ou plus ont été supprimées. Etant donné la taille des données manipulées les étapes #2 et #3 du traitement ont pris une quinzaine de minutes. 

Type de traitement | Nombre total d'enregistrements (au 31/07/2023)
-------- | -----
Sans traitement | 27 329 471
Suppression des doublons simples fichier par fichier | 27 209 138 (120 333 lignes supprimées)
Suppression globale des doublons simples | 27 168 192 (161 279 lignes supprimées)

#### Traitement des doublons complexes

Le stockage dans une base de données des informations débarrassées des doublons simples et une simple requête SQL montre que les doublons sont sans doute beaucoup plus nombreux. La requête suivante : 

    SELECT last_name, first_name, birth_date, COUNT(*)
    FROM records
    GROUP BY last_name, first_name, birth_date 
    HAVING COUNT(*) > 1 

renvoie encore 68 923 lignes correspondant à des personnes ayant le même nom, les mêmes prénoms et la même date de naissance. 

En rajoutant un critère sur la date de décès : 

    SELECT last_name, first_name, birth_date, death_date, COUNT(*)
    FROM records
    GROUP BY last_name, first_name, birth_date, death_date
    HAVING COUNT(*) > 1 

on n'obtient plus que 61 422 personnes qui pourraient être mentionnées plusieurs fois dans les fichiers.

Parmi ces résultats figurent des doublons complexes: l'un des attributs change, mais la personne concernée est sans doute la même. 

L'exemple suivant montre la même personne mentionnée à la fois dans le fichier 2010 et dans le fichier 2012, avec les mêmes attributs, sauf que le lieu de naissance est plus précis dans le fichier 2012. 

    ANA EMILIA	2	1927/01/09	FIGUEIRO (SANTA CRISTINA) AMAR	99139	PORTUGAL	2010/09/06	64102	000000937	Deces_2010.csv-ligne 393211

    ANA EMILIA	2	1927/01/09	FIGUEIRO	99139	PORTUGAL	2010/09/06	64102	937	Deces_2012.csv-ligne 139950

Des sondages dans les fichiers de doublons complexes potentiels ont montré d'autres cas, inexpliqués, de la même personne avec deux dates (voire deux lieux) de décès différentes. Dans certains cas il pourrait s'agir de rectifications faites par les communes. 

Le tableau suivant présente le nombre de doublons potentiels en fonction d'un certain nombre d'attributs identiques (même noms/prénoms et même dates de naissance et de décès, etc.) : 
Nom/prénom|Sexe|Date naissance|Lieu naissance|Code naissance|Date décès|Code décès|Doublons potentiels
------- | ----- | ----- | ----- | ----- | ----- | -----| -----
X | X | X |-|-|X|-|61 777
X | X | X |-|X|-|-|61 371
X | X | X |-|-|X||55 439
X | X | X |-|-|X|X|51 490
X | X | X |-|X|X|X|45 334


Les doublons complexes, qui représentent au maximum 0,25% des données, n'ont pas fait l'objet d'un quelconque traitement pour les supprimer. Ils devraient être traités par un algorithme d'appariement (*matching*), afin d'apporter de la transparence à l'utilisateur. 

### Qualité des données : champs manquants ou de mauvaise qualité

#### Commune de naissance 
La commune de naissance n'est pas toujours connue. Dans ce cas elle est renseignée comme "COMMUNE FICTIVE" avec un code commençant par le numéro de département et se terminant par "990".

#### Codification des communes 
(TBD)

#### Dates invalides
Certaines dates comportent moins de 8 caractères. 

#### Incohérences de dates 
Un certain nombre de dates de décès sont antérieures à la date de naissance. 

 ### Annexe

Fichier CSV INSEE | Nombre de lignes| Doublons simples
-------- | ----- | -----
deces-1970.csv|27 006|0
deces-1971.csv|161 019|1
deces-1972.csv|336 008|0
deces-1973.csv|366 040|2
deces-1974.csv|380 602|0
deces-1975.csv|399 309|2
deces-1976.csv|408 883|1
deces-1977.csv|404 774|0
deces-1978.csv|421 032|0
deces-1979.csv|424 986|0
deces-1980.csv|437 856|0
deces-1981.csv|454 544|0
deces-1982.csv|453 262|0
deces-1983.csv|473 522|0
deces-1984.csv|464 103|0
deces-1985.csv|474 631|1
deces-1986.csv|476 863|3
deces-1987.csv|461 801|1
deces-1988.csv|457 904|1
deces-1989.csv|463 081|16
deces-1990.csv|546 887|6 323
deces-1991.csv|531 675|337
deces-1992.csv|540 832|642
deces-1993.csv|520 434|200
deces-1994.csv|561 326|3 640
deces-1995.csv|522 051|829
deces-1996.csv|579 007|9 217
deces-1997.csv|567 668|41 422
deces-1998.csv|461 460|4 284
deces-1999.csv|697 192|44 724
deces-2000.csv|570 494|2 750
deces-2001.csv|567 111|2 107
deces-2002.csv|549 493|152
deces-2003.csv|573 622|152
deces-2004.csv|537 816|271
deces-2005.csv|557 035|229
deces-2006.csv|535 113|414
deces-2007.csv|536 332|209
deces-2008.csv|553 112|191
deces-2009.csv|557 241|240
Deces_2010.csv|551 015|1 002
Deces_2011.csv|549 115|41
Deces_2012.csv|579 981|327
Deces_2013.csv|582 618|87
Deces_2014.csv|569 445|149
Deces_2015.csv|609 627|86
Deces_2016.csv|603 319|37
Deces_2017.csv|612 926|46
Deces_2018.csv|620 123|20
Deces_2019.csv|625 372|134
deces_2020.csv|679 924|22
Deces_2021.csv|671 469|14
Deces_2022.csv|683 870|8
Deces_2023_M01.csv|64 327|0
Deces_2023_M02.csv|54 285|0
Deces_2023_M03.csv|58 944|0
Deces_2023_M04.csv|50 640|0
Deces_2023_M05.csv|51 143|0
Deces_2023_M06.csv|51 947|0
Deces_2023_M07.csv|46 254|0
TOTAL|27 329 471|120 333
