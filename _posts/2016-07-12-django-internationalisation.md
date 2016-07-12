---
layout: post
comments: true
title: Tout comprendre des timezones dans Django
tags: [python, django]
---

L'internationalisation (abrégée i18n) permet aux clients d'avoir le contenu de notre back-end Django adapté à leur langue, à leur fuseau horaire, et aux différents formats particuliers (ex : monnaie, affichage de l'heure, etc.). Django propose une panoplie d'outils intéressants pour gérer cette problématique parfois plus complexe qu'elle en a l'air.

Parmi toutes ces problématiques, la gestion des timezones (fuseaux horaires) est une des parties les plus importantes de l'i18n puisqu'elle va permettre aux utilisateurs d'enregistrer et d'afficher des dates cohérentes, peu importe où ils se situent dans le monde.

C'est sur ce sujet qu'on va se concentrer dans cet article, en passant en revue toutes les notions essentielles à comprendre (hors de question de laisser la magie opérer), et les actions à effectuer pour mettre en place des bases saines. Enfin, on verra des cas particuliers un peu plus complexes.
A la fin de cet article, la mise en oeuvre des timezones dans un projet Django n'aura plus de secret pour vous !

C'est parti !

<hr>

#### Aware VS Naïve ?

Les objets Python `datetime` possèdent un attribut `tzinfo` qui peut être utilisé pour contenir les informations sur la timezone. Concrètement, cet attribut permet de dire à quelle zone de fuseau horaire est lié l'objet datetime, et quel est le décalage avec le temps universel [UTC](https://fr.wikipedia.org/wiki/Temps_universel_coordonn%C3%A9) (qui correspond à l'heure de Londres, sans prendre en compte le passage à l'heure d'été...).

+ Lorsque l'attribut tzinfo est nul, on dit que le datetime est "naive" : cela signifie qu'on a aucune info de fuseau horaire lié. Il est alors impossible d'utiliser cet objet dans un contexte international, ou bien plus simplement après un passage à l'heure d'été/hiver.
+ Lorsque l'attribut tzinfo n'est pas nul, le datetime est "aware" : les infos de cet attribut permettent d'utiliser l'objet dans un contexte international.
+ Il faut bien sûr éviter de mélanger en base de données des datetime naive et aware, ce qui n'aurait aucun sens et provoquerait des erreurs à coup sûr.

Exemple de représentation Python :

+ `datetime.datetime(2016, 5, 10, 12, 12)` est un datetime naïve.
+ `datetime.datetime(2016, 5, 10, 12, 12, tzinfo=<DstTzInfo 'Europe/Paris' CEST+2:00:00 DST>)` est un datetime aware, représenté en utilisant le fuseau horaire de Paris.

<hr>

#### UTC vs Localtime ?

Vous l'avez compris, vous aurez besoin d'utiliser des datetimes aware, tout le temps.
Lorsqu'un datetime est aware, il peut être représenté selon n'importe quelle timezone. En général, vous trouverez des datetimes soit au format UTC (vous verrez pourquoi après), soit au temps local (localtime) qui correspond à l'heure locale de votre serveur.

Attention, la *représentation* dans une timezone ou une autre ne change pas la "valeur" de l'objet date, comme le montre l'exemple ci-dessous :

```python
>>> now_utc = timezone.now()
>>> now_utc
datetime.datetime(2016, 7, 12, 13, 55, 57, 317577, tzinfo=<UTC>)  # Il est 13h55 au temps universel ...
>>> now_paris = localtime(now_utc)
>>> now_paris
datetime.datetime(2016, 7, 12, 15, 55, 57, 317577, tzinfo=<DstTzInfo 'Europe/Paris' CEST+2:00:00 DST>)  # ... et il est 15h55 à Paris !
>>> now_utc == now_paris
True  # Ces objets sont égaux, car il s'agit de la même date représentée de 2 manières différentes !
```

<hr>

#### Comment Django gère la création des datetimes ?

+ Si `USE_TZ = False` dans les settings : tous les datetimes sont stockés de manière "naïve".
+ Si `USE_TZ = True` dans les settings (ce qui nous intéresse ici) : tous les datetimes sont stockés de manière "aware". Mais avant d'être stockés, ces objets sont tous convertis en UTC. Pourquoi ? C'est plutôt logique comme comportement par défaut, puisqu'on n'a pas envie de se retrouver avec une base de données remplie de datetimes hétérogènes. Imaginez que vous ayez des clients du monde entier, ça serait un peu le bazare dans la base de données. Autant avoir un seul fuseau horaire, que tous nos datetimes soient relatifs à celui-ci.

#### Comment Django gère l'affichage des datetimes ?

Mais du coup, si tout est stocké en UTC, il va falloir faire une conversion pour l'affichage dans le fuseau horaire du client ? Tout à fait !

Par défaut, lorsqu'une date est affichée via un template Django ou dans l'admin, cette conversion UTC -> timezone locale est effectuée automatiquement afin d'afficher la bonne heure au client.

Par contre, lorsque vous souhaitez afficher une date en dehors du mécanisme classique de Django (ex: dans une commande, dans un simple `print()` etc.), il est nécessaire d'appeler vous même la fonction `localtime()` de Django qui convertira votre datetime dans le format local, en utilisant la timezone courante (celle définie dans les settings).

#### Et moi ? comment dois-je instancier les objets datetime de la bonne façon ?

Lorsque les timezones sont activées, il est crucial de stocker en base des datetimes awares.
Pour cela :

+ Lorsque vous créez votre datetime depuis la date actuelle, utilisez `timezone.now()` qui contiendra un tzinfo, contrairement à `datetime.now()`.
+ Lorsque vous créez votre objet en instanciant un `Datetime()` directement, malheureusement, cela créé un datetime naïve.
Je vous invite alors à créer une fonction "chapeau" qui permet d'effectuer la transformation à la volée quand vous en avez besoin. Voici celle que j'ai créée :

```python
def custom_localize(date_or_datetime):
    """ Take a naïve date or datetime and return the corresponding aware date or datetime with localtime """

    return pytz.timezone(get_current_timezone_name()).localize(date_or_datetime)

>>> datetime(2016, 12, 20, 0, 0)
datetime.datetime(2016, 12, 20, 0, 0)  # Naïve datetime, pas bon !!!
>>> custom_localize(datetime(2016, 12, 20, 0, 0))
datetime.datetime(2016, 12, 20, 0, 0, tzinfo=<DstTzInfo 'Europe/Paris' CET+1:00:00 STD>)  # Tout va bien, notre objet sera automatiquement converti en UTC s'il doit être stocké en base.
```


<hr>

### Et si j'ai une programming API pour mes applis mobiles, comment je gère la sérialisation et désérialisation des datetimes ?

A ce moment, il y a besoin d'une prendre décision assez importante, à savoir qui est responsable de définir la timezone courante :

1. C'est le serveur ? -> dans ce cas, peu importe où se trouve géographiquement le client avec son smartphone, c'est le serveur qui dictera la timezone courante (ça peut être par exemple un champ du profil utilisateur qui contient cette info).
2. C'est l'appli mobile ? -> dans ce cas, quand le client voyagera avec son téléphone, la timezone détectée par le smartphone changera, et les dates que l'application mobile enverra à l'API changeront par la même occasion.


#### Désérialisation (objet Python -> string)

Puisque les datetimes Django sont stockés en UTC, ils seront désérialisés par défaut de la même manière.

##### Cas 1
L'application mobile reçoit, via l'API, un datetime contenant un tzinfo local.
L'applicatio mobile n'a aucune conversion à effectuer et l'affiche l'heure telle quelle.

##### Cas 2
L'application mobile reçoit, via l'API, un datetime en UTC, et doit elle-même effectuer la conversion, via la timezone détectée, pour l'afficher en localtime au client.

#### Sérialisation (string -> objet Python)

Dans les deux cas, l'application mobile peut envoyer le datetime représenté avec n'importe quel timezone, puisque de toute manière, Django va le stocker en UTC in fine. Le seul point important est de respecter la norme ISO 8601 pour que le datetime puisse être parsé correctement.
Mais dans le cas 1, le datetime envoyé devra utiliser une tzinfo récupérée du serveur auparavant, alors que dans le cas 2, l'application détectera elle-même la timezone en fonction de localisation géographique courante de l'utilisateur.

<hr>

### Au secours, je suis déjà en prod sans les timezones activées !

Pas de panique, ça m'est aussi arrivé ! J'ai pu travailler sur un système de transport urbain en mini-bus, où des utilisateurs réservent leur trajet via une appli Smartphone. Autant dire que les horaires jouent un rôle majeur, et que l'on a pas le droit à l'erreur !

Par soucis de simplicité/rapidité, le projet existant était sétté sans timezones (`USE_TZ = False`) en production. Puis, un contrat avec des anglais a été signé. Oups... Que faire avec tous ces naive datetimes parsemés dans notre base de données ?

En effet, un projet Django ne devrait jamais être sans timezones, sauf si on n'utilise pas du tout de datetimes, ou qu'ils n'ont vraiment aucune importance dans le projet (autant dire que ces cas doivent être rares). Même en restant sur un seul pays comme la France, il y a des changements d'heure à chaque été/hiver, qui ne peuvent pas être pris en compte avec des naive datetimes.

#### La solution ?

Un script d'introspection capable de parcourir automatiquement tous les objets datetime dans votre base de données a été développé (je ne vais pas détailler ici comment l'écrire).
A l'intérieur de ce script, sur chaque objet datetime trouvé, il a fallu effectuer la conversion en appliquant une fonction de ce genre, qui devra bien sûr être adapté en fonction du contenu initial de votre base : 

```python
def get_updated_time_or_datetime(time_or_datetime):

    if time_or_datetime:
        if is_naive(time_or_datetime):
            return make_aware(time_or_datetime, timezone.utc) - timedelta(hours=2)
        else: 
            return time_or_datetime
```

En soi, cela n'a rien de sorcier, mais cette opération de maintenance nécessite de couper totalement la prod, et d'avoir fait des tests intensifs avant. Il faut aussi peut être exclure certaines tables qui appartiennent à des applications tierces et utilisent des objets datetime (ex : Celery).
C'est assez risqué comme opération si on n'est pas sûr de ce que l'on fait, mais je ne vois pas d'autre moyen.


#### Conclusion

Passer un peu de temps à bien comprendre le fonctionnement des objets Python datetime, et des timezones dans Django, est un investissement vite rentabilisé. Il n'y a rien de plus dangereux que d'éviter ce genre de sujet sous pretexte qu'il n'est pas très passionnant, pour après se retrouver bloqués à devoir faire une migration lourde et risquée pour notre environnement de prod.
Si vous souhaitez rentrer encore plus dans le détail, allez faire un tour sur [la doc Django des timezones](https://docs.djangoproject.com/en/1.9/topics/i18n/timezones/).

A bientôt !


