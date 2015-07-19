---
layout: post
comments: true
title: Bien utiliser Amazon S3 avec Django
tags: [tutorial, django, python, aws]
---

Stocker les fichiers statiques et média d'un site web sur un serveur à part permet d'améliorer sensiblement la vitesse de chargement des pages de votre site web. C'est un rôle qu' Amazon Simple Storage Service (abrégé S3) rempli à merveille.

Après un bref rappel des notions nécessaires, nous allons voir dans cet article comment utiliser au mieux le service Amazon s3 avec un site Django, d'une part pour la gestion des fichiers statiques et média.

##Quelques rappels sur Django _(ici 1.8)_

####Fichiers statiques VS fichiers média

Les fichiers **statiques** (static files) sont les images, fichiers CSS, scripts JS, et tout autre contenu non dynamique (oui c'est la définition de statique...) utilisés par votre site web.

Les fichiers **media** (media files) concernent eux n'importe quel contenu uploadé par les utilisateurs de votre site.

####Variables dans settings.py

Afin de bien comprendre ce que nous allons faire, il faut bien comprendre à quoi servent ces variables :


Pour les fichiers statiques :

- `STATIC_ROOT` : le chemin absolu qui sera parsé par la commande `collectstatic` de Django, afin de collecter les fichiers statiques pour les déplacer dans un répertoire accessible aux utilisateurs en production.
- `STATIC_URL` : l'URL (relative) qui servira les fichiers statiques en production.
- `STATICFILES_STORAGE` : le moteur de stockage de fichiers utilisé par la commande `collectstatic` (nous allons devoir le changer pour utiliser Amazon S3).


De manière quasi "symétrique" pour les fichiers média :

- `MEDIA_ROOT` : le chemin absolu qui contiendra les fichiers uploadés par les utilisateurs.
- `MEDIA_URL` : l'URL (relative) qui gère les fichiers uploadés par les utilisateurs.
- `DEFAULT_FILE_STORAGE` : le moteur de stockage de fichiers utilisé pour toute opération liée à des fichiers (en dehors de collectstatic).

####Utilisation du tag static

Comme l'explique déjà parfaitement [cet article](http://staticfiles.productiondjango.com/blog/stop-using-static-url-in-templates/), on peut rapidement être tenté d'utiliser la variable `{{STATIC_URL}}` à laquelle on concatène le reste de l'URL pour spécifier le chemin d'un fichier dans nos templates Django. Or, il semble que ce soit l'une des pires idées qui soit puisque ça a l'air de parfaitement fonctionner alors que dans certains cas, non.


Lisez l'article en question ou à défaut, retenez simplement que la _bonne façon_ de charger ses fichiers statiques dans un template Django est la suivante :

```python
{{ "{% load static from staticfiles " }}%}
<img src="{{ '{% static "images/toto.jpg" '' }}%}" />
```

## Les étapes à suivre

####Créer un compte Amazon S3 et un bucket

Je pars ici du principe que vous disposez déjà d'un compte Amazon S3, et que vous avez créé un bucket (répertoire) et un utilisateur ayant les droits dessus. Idéalement, si vous utilisez votre compte S3 pour plusieurs sites, il faudrait créer un utilisateur dédié par site, pour assurer un meilleur cloisonnement.
Vous devez aussi disposez de vos credentials (access key ID et secret access key). Si vous ne voyez pas de quoi je parle, rendez vous sur _Security Credentials > Manage access key_, pour en créer une nouvelle. 

####Déplacer les fichiers statiques vers s3

Il n'y a que deux choses dont il faut s'assurer pour ça fonctionne, et elles sont assez évidentes :

1. Etre sûr que quand une page Django est générée, les liens vers les fichiers statiques pointent bien vers l'URL Amazon S3 correspondante.
2. Les fichiers sont effectivement bien présents dans votre bucket Amazon S3.

Pour la 1ère partie, c'est assez simple. Il suffit d'utiliser le tag `static`.

Pour la 2e partie, on voudrait pouvoir déplacer facilement nos fichiers vers le bucket s3, et que ce dernier reste constamment à jour lorsqu'on ajoute de nouveaux fichiers statiques. Nous allons voir qu'il est possible que la commande `collectstatic` fasse ce travail pour nous !

Mais d'abord, installez deux packages : [django-storages-redux](https://github.com/jschneier/django-storages) (il s'agit d'un fork de django-storages, mais maintenu et compatible avec Python3) et [boto](https://github.com/boto/boto) :
`pip install django-storages-redux boto`


Ajoutez la ligne `storages` dans `INSTALLED_APPS` dans le fichier settings.py.

Ensuite, ajoutez les variables suivantes dans votre fichier settings.py:

```python
AWS_STORAGE_BUCKET_NAME = "bucket_name"
AWS_ACCESS_KEY_ID = "your_access_key_id"
AWS_SECRET_ACCESS_KEY = "your_secret_access_key"
AWS_S3_HOST = "your_s3_host"
AWS_S3_URL = 'https://{0}.s3.amazonaws.com/'.format(AWS_STORAGE_BUCKET_NAME)

AWS_STATIC_DIR = 'static'
STATIC_URL = AWS_S3_URL + AWS_STATIC_DIR + '/'
STATICFILES_STORAGE = 'your_project_name.storage.StaticRootS3BotoStorage'
```

Quelques explications s'imposent :

- la variable `AWS_S3_HOST`  est nécessaire quand la location du bucket n'est pas celle US par défaut. Exemple de valeur pour le datacenter Amazon situé en Irlande : `s3-eu-west-1.amazonaws.com`. La liste des différents host est [ici](http://www.bucketexplorer.com/documentation/amazon-s3--amazon-s3-buckets-and-regions.html).
- par défaut, les fichiers sont écrits à la racine du bucket s3. Evidemment, on voudrait éviter de mélanger des fichiers statiques avec des fichiers média uploadés par un utilisateur. Pour cela, `AWS_STATIC_DIR` sera utilisé comme sous-répertoire de notre bucket contenant les fichiers statiques.
- `STATICFILES_STORAGE` : ici c'est assez subtile ! Si on n'avait pas voulu utiliser des sous-répertoires dans notre bucket, on aurait écrit directement : `'storages.backends.s3boto.S3BotoStorage'`. Mais en fait ici, on va surcharger cette classe nous-même pour lui indiquer dans quel sous-répertoire sauvegarder les fichiers.


Pour cela, dans votre projet (j'ai bien dit projet, pas application ^), créez un fichier storage.py, avec ce contenu :

```python
from storages.backends.s3boto import S3BotoStorage
from django.conf import settings

class StaticRootS3BotoStorage(S3BotoStorage):
    location = settings.AWS_STATIC_DIR
```

####Essayez !

Après avoir suivi toutes les étapes, vous devriez pouvoir uploader vos fichiers statiques vers votre bucket s3, avec la commande collectactic: `python manage.py collectstatic`

Baladez vous ensuite sur votre site et regardez l'URL de vos images. Et voilà !

####Au tour des fichiers média !

Vous avez de la chance, on a déjà fait quasiment tout le travail. La logique est la même pour les fichiers média.

Dans settings.py, il suffit d'ajouter :

```python
AWS_MEDIA_DIR = 'media'
MEDIA_URL = AWS_S3_URL + AWS_MEDIA_DIR + '/'
DEFAULT_FILE_STORAGE = 'livinproject.storage.MediaRootS3BotoStorage'
```

et dans le fichier storage.py créé précédemment :

```python
class MediaRootS3BotoStorage(S3BotoStorage):
    location = settings.AWS_MEDIA_DIR
```

####Aller plus loin : changer le storage en fonction de l'environnement

il se peut pour des raisons évidentes que vous n'ayez pas envie d'utiliser s3 (qui n'est pas gratuit après tout) pendant le développement. De plus, devoir relancer la commande `collectstatic` à chaque ajout d'un fichier peut devenir assez rapidement pénible.

Pour faire ça, il vous faudra simplement changer les URLs et le storage en fonction de l'environnement, avec des variables d'environnement. Vous trouverez [ici un exemple de code](https://github.com/jschneier/django-storages/issues/6#issuecomment-66896737) qui montre simplement comment procéder.

C'est terminé ! N'hésitez pas si vous avez des questions. A bientôt.

