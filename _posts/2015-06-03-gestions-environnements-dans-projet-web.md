---
layout: post
comments: true
title: La gestion des environnements dans un projet web.
tags: [production, methodology, security]
---

Parmi les tâches d'un CTO qui ne sont pas du développement pur, la bonne gestion des différents environnements me parait être l'une des plus importantes. Retour sur les bases de la gestion des environnements

##Qu'est-ce qu'un environnement ?

De manière générale (laissons de côté le web pour le moment), un environnement est un ensemble de matériels et de logiciels utilisés par l'application en question.


On cherche à isoler les environnements les uns des autres tant que possible, pour plusieurs raisons :

- Parce que chaque environnement a un but précis, ces derniers requièrent des performances différentes, des données différentes, des utilisateurs différents.
- Parce qu'on souhaite éviter à tout prix que les problèmes éventuels de l'un puissent affecter l'autre. Imaginez vous un instant, si j'arrivais à planter un système bancaire parce que j'ai fait une boucle infinie pendant le développement ?

####Quels environnements retrouve-t-on typiquement dans un projet ?

Typiquement dans un projet standard nous pourrions avoir :

- __N__ environnements de **développement** : chaque développeur dans le projet va avoir son propre environnement. Tout ce qu'il développe n'impacte dans un premier temps que lui. Un même développeur peut même avoir plusieurs environnements de développement (par exemple s'il travaille depuis un desktop au travail, et depuis un laptop chez lui). Dans un projet web, le développeur a très souvent en local : un serveur web , un SGBD avec une base, et son code.

- 1 environnement de **production** : c'est l'environnement utilisé par les clients. Il doit tant que possible être fonctionnel (non buggué) et disponible sur les plages horaires prévues, en fonction des [SLA](http://fr.wikipedia.org/wiki/Service_level_agreement).

- 1 environnement de **pré-production** (ou de **qualification**) : c'est un environnement intermédiaire utilisé après le développement, et avant la mise en production. Tout l'intérêt de cet environnement réside dans le fait qu'il soit identique à celui de la production (en tout cas qu'il s'en rapproche le plus possible), et qu'il va ainsi nous permettre d'obtenir des résultats de tests pertinents. J'insiste sur le fait que ce n'est pas parce que le code fonctionne en environnement de dév, qu'il fonctionnera en production. Car entre ces deux environnements, quasiment tout est différent :
  - Le serveur HTTP : en dév, il est probable que votre serveur soit un petit outil à exécuter localement, mais qui se comporte pas exactement de la même manière qu'un serveur dédié à la production comme Nginx ou Gunicorn. C'est le cas quand vous lancez la commande `python manage.py runserver localhost:8001` sur Django par exemple.
  - Le contenu de la base de données : à moins que vous ayez fait un [dump](http://en.wikipedia.org/wiki/Database_dump) de votre base de prod vers votre environnement de dév avant d'effectuer vos tests (ce qui n'est pas conseillé pour la sécurité), il est évident que vous ne disposez pas des mêmes données par défaut !
  - Autres détails : 
      - il est probable que les pages soient servies en HTTP**S** en prod, mais pas en dév.
      - En dév, il y a un mode DEBUG activité qui change le comportement de votre framework web pour servir les pages.
      - Etes vous certains que **tous** les composants logiciels de chaque environnement de dév (langage,framework, packages, base de données) sont au même niveau de versionning que la prod ? A moins de volontairement s'y contraindre, il y a peu de chances.
 
  Si avec cette liste, vous n'êtes toujours pas convaincus de la nécessité absolue d'un environnement intermédiaire dans un projet web sérieux, j'abandonne !

Il existe aussi des environnements à but plus spécifique, comme par exemple : 

  - Un environnement de **démonstration** : utilisé pour faire la démonstration de votre site web. Il contient en général un jeu de données fictives et peut facilement être restauré pour revenir au même état initial.
  - Un environnement de **formation** : utilisé pour former des utilisateurs, notamment en entreprise pour faciliter la gestion du changement.
  - *etc*...

Il y a beaucoup d'autres possibilités en fonction des besoins du projet en question, et vous n'êtes limités que par les ressources matérielles et le temps dont vos disposez.

####Comment sont utilisés les environnements pendant le cycle de développement ?

Il n'y a pas qu'une réponse à cette question, mais voici une manière logique de travailler en développant seul un site web :

- Développement d'une ou plusieurs features et développement des tests associés.
- Dump de la base de données de production vers la pré-production : à ce moment précis, la pré-production a exactement les mêmes données que votre production.
- Push du code développé sur la pré-production et lancement des tests unitaires / intégration / non régression / etc.
- Si les tests sont OK en pré-prod, alors push du code sur la production, en faisant attention à toute coupure éventuelle, et en ayant d'avance une procédure de retour arrière (rollback) en cas de problème.

Rien qu'avec cette étape intermédiaire de la pré-production, on vient de limiter énormément les risques de mise en production ratée. Bien évidemment, ceci est condtionné par la qualité des tests associés, mais c'est un autre sujet :) 

Voilà, vous avez eu un aperçu très théorique de la gestion des environnements au sein d'un projet. Dans un prochain article, nous verrons comment mettre tout ça en pratique sur [Heroku](http://fr.wikipedia.org/wiki/Plate-forme_en_tant_que_service).

A bientôt !
