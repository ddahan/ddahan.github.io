---
layout: post
comments: true
title: Présentation technique du projet Magnetiz
tags: [python, django]
---

Aujourd’hui, je publie un article un peu particulier puisque j'ai décidé de présenter [Magnetiz](http://www.magnetiz.fr) en rentrant dans le détail de certains aspects techniques, y compris dans le code lui-même.

Après avoir rappelé brièvement le fonctionnement de la solution, je présenterai l'architecture logique et physique de celle-ci, puis je rentrerai plus dans le détail du projet Django lui même, afin d'évoquer quelques problématiques rencontrées, et les solutions utilisées.

Il ne s'agit pas d'une présentation exhaustive mais plutôt d'un tour d'horizon technique. Cet article pourra donc être complété au fil du temps.

### Avant-propos : un point sur la sécurité

Traditionnelement, un article tel que celui-ci ne devrait jamais être publié avec une solution en production car il révèle des détails techniques qui pourraient aider des potentiels hackers.
Je le fais malgré tout car d'une part, la solution est sur le point d'être arrêtée, et d'autre part, je pense que la [sécurité pas l'obscurité](https://fr.wikipedia.org/wiki/S%C3%A9curit%C3%A9_par_l%27obscurit%C3%A9) est une mauvaise pratique.

### Qu'est-ce que Magnetiz ?

Magnetiz est une solution de fidélisation destinée aux commerces locaux. Chaque commerçant est équipé d'une ou plusieurs tablette(s) mobiles sur son comptoir, ainsi que de cartes de fidélité *vierges* à disposition pour les clients.

Lors de son premier passage en magasin, un client récupère une carte, et la scanne sur la tablette. Il est invité à entrer son adresse e-mail (et uniquement son e-mail), et cumule ses premiers points (par défaut : 5 points).

Après sa pré-inscription en magasin, le client reçoit un mail pour finaliser son compte (choisir un mot de passe et compléter ses infos personnelles). Il pourra s'il le souhaite télécharger et se connecter ensuite à l'application mobile.

A chaque passage en magasin, le client scanne sa carte (ou son smartphone) directement sur la tablette tactile. A chaque fois, il cumule des points. A tout moment, il peut réclamer un cadeau directement en le sélectionnant sur la tablette, à condition d'avoir suffisament de points, bien sûr.

Une même carte est utilisable dans tous les commerces Magnetiz.

le commerçant dispose d'un dashboard lui permettant entre autres de suivre ses clients, créer et maintenir sa liste de cadeaux, avoir des statistiques détaillées, et surtout, d'envoyer des e-mails et coupons promotionnels à durée limitée à ses clients, afin de les inciter à revenir.
Le client peut récupérer son coupon promotionnel qui s'affichera directement sur la tablette au côté des cadeaux disponibles.

Le client peut vérifier ses points et les cadeaux disponibles de deux manières : soit en se connecter au dashboard client, soit directement dans l'application mobile. Cette dernière permet aussi de remplacer la carte physique car elle affiche le même QR code.

### Architecture

L'idée est de comprendre ici comment s'articulent les différents composants de la solution entre eux, avant de rentrer dans le détail du projet Django.

##### Architecture globale physique

<img src="/res/misc/magnetiz_vue_physique.png" class="img-responsive">

D'un point de vue purement physique, le schéma est assez simple et parle de lui-même. L'hébergement est centralisé chez Heroku (et une partie sur Amazon S3 qui n'apparait pas sur ce schéma). Seuls l'envoi de mails et la gestion des paiements sont gérés par des services externes.

<hr>

##### Architecture globale logique

<img src="/res/misc/magnetiz_vue_logique.png" class="img-responsive">

Sur ce schéma logique, on distingue 4 composants internes principaux qui ont été développés :

+ L'application [Magnetiz pour commerçants](https://play.google.com/store/apps/details?id=com.ionicframework.magnetiztablet479260)
+ L'application [Magnetiz](https://play.google.com/store/apps/details?id=com.ionicframework.magnetizsmartphone768939) classique pour les clients. Ces deux applications utilisent exactement la même stack technologique.
+ un serveur node.js (qui n'est plus utilisé à ce jour) qui permettait des fonctionnalités "temps réel" grâce aux websockets.
+ Le serveur HTTP Django, composant principal de l'application, puisqu'il contient à lui-seul toute la logique métier du projet, les dashboards clients/commerçants, la landing page, et la programming API permettant aux applications mobiles de communiquer avec le back-end.

L'utilisation des mêmes technos (HTML/CSS) pour les applications mobiles, pour les dashboards, et pour la landing page, a permis un gain de temps considérable, en n'affectant quasiment pas l'expérience utilisateur (les applications natives avaient dans ce projet peu de valeur ajoutée).

<hr>

##### Architecture web du projet

L'architecture web est un choix important qui doit tenir compte de plusieurs critères : temps disponible, conaissances des développeurs, besoin de séparation front/back fort ou pas, etc.

Ici, l'architecture retenue a été du simple MVC (MVT dans Django) + une API + 2 applis front externes. Les dashboards clients/commerçants et la landing page utilisent directement Django.

L'avantage évident est de bénéficier de nombreux mécanismes bien rodés de Django (ex : authentification) qui permettent de simplifier la vie du développeur.
L'inconvénient est un certain couplage entre le front et le back, malgré une séparation MVT.
Un autre risque est de répéter la logique métier plusieurs fois, mais cela peut être évité en s'assurant que 100% de la logique soit dans les models (et donc : 0% dans les vues ou dans l'API !)

Une autre architecture aurait aussi pu être :

+ d'utiliser Django (ou un autre framework tel que Flask) uniquement pour les models et l'API. Ici, les web services auraient été appelées pour n'importe quelle requête effectuée.
+ développer une appli front-web par composant (en utilisant AngularJS par exemple) : 1 pour les dashboards, 1 pour l'appli smartphone, 1 pour l'appli tablette.

L'avantage principal aurait été ici de découpler totalement la partie back (Django) de la partie clients (front). Un autre avantage aurait été de bénéficier "nativement" de features de mise à jour du DOM dans les dashboards, sans devoir utiliser jQuery par dessus Django.
L'inconvénient aurait été de ne pas profiter des mécanismes de Django dans la partie "View" et "Template".

<hr>

##### Arborescence de fichiers Django

L'arborescence simplifiée *(je ne conserve ici que les fichiers additionnels aux fichiers par défaut)* du projet Django est décrite ci-dessous. Elle est somme toute assez classique :

+ monprojet/
  + magnetiz/
      * management/  *-> contient différentes commandes d'admin Django*
  + restless_api/
      * api.py  *-> contient l'api des deux applications mobiles*
  + templates/
      * components/ *-> permet de séparer les pages en plusieurs parties*
      * dashboard_client/ 
      * dashboard_owner/
      * front/ *--> pages liés à aucun dashboard (ex: landing page)*
      * tools/ *-> Script Google Anaytics, messages snippets, etc.*
  + tests/ *-> les tests sonts séparés par fichiers*
      * tests_api.py
      * tests_models.py
      * tests_myutils.py
      * tests\_views\_and_forms.py
  + strings.py *-> contient tous les messages (hors templates)*
  + myconstants.py *-> contient toutes les constantes*
  + myutils.py *-> contient les fonctions génériquées non liées à des classes*
  + monprojet/
  + scripts/ *-> scripts SH divers (ex : RAZ environnement)*
  + venv/ *-> environnement virtuel Python 3.3 spécifique au projet*
  + requirements.txt *-> contient la liste des packages nécessaires au projet*
  + .env -> *différent dans chaque environnement, contient des variables propres à chaque environnement.*

<hr>

### Gestion des environnements

La [gestion des environnements](http://david-dahan.com/post/gestions-environnements-dans-projet-web/) est importante dans un projet web.

Dans Magnetiz, nous disposons de quatre environnements distincts :

+ **Un environnement de développement** (il y en aurait eu plusieurs hétérogènes si j'avais travaillé en équipe sur le projet.)
+ **Un environnement de démo** : il est utilisé pour effectuer les démos du produit auprès du client. Il a la particularité de devoir être en permanence remis à zero avec un jeu de données pré-défini.
+ **Un environnement de pré-prod** : il est le plus proche possible de celui de prod (même données, mêmes packages, etc.). Il permet de tester le comportement de l'application et détecter toute anomalie avant une mise en production.
+ **Un environnement de production** : l'environnement utilisé par les clients de l'application. Il est bien sûr plus critique que les autres.

Heroku (hébergeur PaaS) nous facilite grandement le travail pour séparer nos environnements. D'une part, il nous autorise à créer plusieurs applications par compte, mais surtout, il utilise un fichier `.env` contenant des variables pouvant être définies différemment entre chaque environnement. De cette manière, on garde une unique base de code pour nos différents environnements, seul le fichier `.env` diffère.

Voici un exemple d'une partie du fichier .env de développement (*valeurs modifiées*) :

```
DJANGO_DEBUG=True
DJANGO_MANDRILL_KEY=DuoSaklG9G8pE8
DJANGO_SEND_MAIL=True
```

Le fichier Django `settings.py` (qui lui est identique dans chaque environnement) est adapté en conséquence pour récupérer les valeurs dans ce fichier :

```python
DEBUG = (os.environ['DJANGO_DEBUG'] in ['True', 'true'])
MANDRILL_KEY = os.environ.get('DJANGO_MANDRILL_KEY')
SEND_MAIL = (os.environ['DJANGO_SEND_MAIL'] in ['True', 'true'])
```

### Description de l'API

l'API permet aux applications mobiles de communiquer avec le serveur HTTP. Il s'agit d'une "programming API" et non pas d'une "public API", c'est à dire qu'elle n'est pas exposée publiquement.
Le framework utilisé est [Restless](https://restless.readthedocs.org/en/latest/). C'est un framework minimaliste qui permet d'arriver à créer une API simplement et rapidement.

Même si ce dernier permet une architecture REST, ce n'est pas le principe qui a été utilisé ici. En effet, les web services ne contenant aucune logique (ou un minimum), leur but est simplement d'appeler les méthodes nécessaires dans le fichier `models.py`. L'utilisation des "verbes" HTTP n'avaient donc ici pas de grande valeur ajoutée.

L'authentification est [Basic](https://fr.wikipedia.org/wiki/Authentification_HTTP#M.C3.A9thode_.C2.AB_Basic_.C2.BB). C'est peut-être la plus simple à mettre en place mais aussi la moins sécurisée. Idéalement, les web services doivent être en HTTPS pour ajouter un minimum de sécurité et éviter les interceptions de type man-in-the-middle.

### Gestion du versionning

Le versionning est une problématique complexe lorsque l'application contient plusieurs composants, notamment dans le cas de Magnetiz car nous devons nous assurer que les applications mobiles (peu importe leur version) soient toutes compatibles avec le back. Or, il est presque impossible de forcer un utilisateur à mettre à jour son application mobile.

Fonctionnellement, il a été décidé de permettre que plusieurs versions d'applications soient compatibles avec le back-end, tout en nous autorisant à "déprécier" certaines versions trop vieilles, et ainsi demander à l'utilisateur de faire une MAJ au lancement d'une application deprecated.

Techniquement, voici comment est agencé le versionning :

+ Chaque application mobile a un fichier contenant entre autres son propre numéro de version :

```javascript
var cfg = {
  backend_host: 'www.magnetiz.fr',
  api_name: 'api_t',
  app_version: '2-1-0',
  google_play_url: 'http://play.google.com/store/apps/details?id=com.ionicframework.magnetiztablet479260'
}
```

+ Côté back-end, le fichier `urls.py` redirige vers les bons web-services en fonction des URLs :

```python
url(r'api_t/2-[0-2]-[0-9]/get_owner_infos/$', GetOwnerInfos_1.as_detail(), name='get_owner_infos')
```

Ici, on voit que si la version de l'application tablette est entre 2.0.X et 2.2.X, alors la version 1 de GetOwnerInfos est appelée. En effet côté APIs, chaque web service dispose d'une version propre. Les web services sont nommés sous la forme `MyWebService_X` où `X` est un entier correspondant à la version du service.

Que se passe t-il si le client a une version d'application mobile inférieure ?
Afin de gérer les versions d'appli mobiles deprecated, il suffit de créer un web_service dédié, qui une fois appelé, aura pour seul rôle de lever une exception invitant l'utilisateur à mettre à jour l'application.

```python
url(r'api_t/1-[0-9]-[0-9]/get_owner_infos/$', Deprecated_T.as_detail()),
```

### Utilisation des tests unitaires

Les tests unitaires sont la 1ère étape pour garantir un code de qualité. Dans Magnetiz, les tests ont été développés pour les parties les plus critiques du code, et notamment l'API tablette.
Petite particularité, un [LiveServer](https://docs.djangoproject.com/en/dev/topics/testing/tools/#django.test.LiveServerTestCase) a été utilisé dans le but de pouvoir utiliser la lib [Requests](http://docs.python-requests.org/en/latest/) pour effectuer les requêtes, très pratique dans le cas des tests de l'API.

Voici un exemple de test simple qui vérifie la récupération d'un client à partir d'un numéro de carte de fidélité :

```python
class GetClientFromCard_1_Test(LiveServerTestCase):
    ''' Test du WS GetClientFromCard_1 '''

    def setUp(self):
        api_test_scenario()
        self.url = BASE_T_URL + 'get_client_from_card/'

    def test_scan_invalid_card(self):
        ''' Scan d'un numéro de carte inconnu '''
        resp = requests.get(self.url + '999', auth=OWNER_AUTH)
        self.assertEqual(resp.status_code, 404)

    def test_scan_weird_stuff(self):
        ''' Scan d'un string étrange (ex : autre qr code) '''
        resp = requests.get(self.url + '&$?_&!', auth=OWNER_AUTH)
        self.assertEqual(resp.status_code, 404)

    def test_scan_unlinked_card(self):
        ''' Scan d'une carte vierge, pas liée à un shop group '''
        resp = requests.get(self.url + '30303030303', auth=OWNER_AUTH)
        self.assertEqual(resp.json()['client_id'], None)
        self.assertEqual(resp.status_code, 200)
```

### Utilisation des messages

Django vient avec un [système de message](https://docs.djangoproject.com/en/dev/ref/contrib/messages/) très pratique.

C'est ce système qui a utilisé pour tous les messages dans les dashboards. Voici le fonctionnement très simple :

Les fichiers messages sont classés dans un fichier séparé nommé `strings.py`. Ceci permet d'éviter de polluer le fichier `views.py` avec trop de texte.
Il suffit ensuite dans la view de lancer le message, par exemple : 

```python
messages.error(request, ERR_NON_PAY) # ERR_NON_PAY est une variable dans strings.py.
```

Enfin pour l'affichage, il suffit d'associer une classe particulière Bootstrap en fonction du tag du message associé. Par exemple :

```html
{{ "{% if msg.tags == 'error' " }}%}
    <div class='alert alert-danger alert-dismissable text-center'>
        <a class="close" data-dismiss="alert" href="#">&times;</a>
        <i class='icon-remove-sign'></i>
        {{ "{{ msg "}}}}
    </div>
{{ "{% endif " }}%}
```

### Jeux de données

Afin de pouvoir remplir rapidement la base de données,
Django propose l'utilisation d'un système de [fixtures](https://docs.djangoproject.com/en/dev/howto/initial-data/). Celui-ci est très limité car les données sont statiques, et la maintenance du fichier JSON est compliquée à partir d'une certaine quantité de données.

Dans Magnetiz il a été nécessaire de créer 2 jeux de données différents :

+ 1 jeu de données servant au développement
+ 1 jeu de données pour avoir un environnement de démo représentatif, afin de présenter l'application Magnetiz avec des données pertinentes aux commerçants.

Pour cela, la lib [FactoryBoy](https://factoryboy.readthedocs.org/en/latest/) a été utilisée. Elle permet de créer des factories (des usines) qui utilisent un modèle Django et le reproduisent en remplissant les objets avec des données prédéfinies, mais dynamiques. Voici un exemple d'utilisation pour un utilisateur :

```python
# factories.py
class UserFactory(factory.DjangoModelFactory):
    class Meta:
        model = User # Le modèle à Utiliser 
 
    first_name = factory.Sequence(lambda n: 'Prenom%d' % n) # itération automatique
    last_name = factory.Sequence(lambda n: 'Nom%d' % n)
    email = factory.LazyAttribute(lambda obj: '%s.%s@yopmail.com' \
        % (obj.first_name.lower(), obj.last_name.lower()))
    password = factory.PostGenerationMethodCall('set_password', 'azerty42')
    birth_date = FuzzyDate(date(1920, 1, 1), date(2002, 1, 1))
    gender = FuzzyChoice(choices=(GENDER_CHOICES[0][0], GENDER_CHOICES[1][0]))
    email_verified = True

# ...
```

Ici, en choisissant par exemple de créer 10 utilisateurs d'un coup, le 7e s'appelera "Prenom7 Nom7", aura prenom7.nom7@yopmail.com comme adresse e-mail, aura une date de naissance au hasard, un genre au hasard, et une adresse e-mail déjà vérifiée.

Les différents scénarios sont ensuite de simples fonctions écrites dans un fichier à part, qui utilisent ces factories. A ce moment, le comportement par défaut des factories peut-être surchargé pour certains attributs uniquement :

```python
# scenarios.py
def demo_scenario():
    ''' Création des données initiales pour l'environnement de démo. '''

    user01 = UserFactory.create(first_name="Gerard", last_name="Philippe", birth_date=date(1945, 4, 30), gender="M")
    user02 = UserFactory.create(first_name="Sonia", last_name="Roland", birth_date=date(1979, 4, 8), gender="F")
    # ...
```


### Utilisation des commandes Django

Django permet de créer [nos propres commandes](https://docs.djangoproject.com/en/dev/howto/custom-management-commands/), qui seront par la suite utilisables en tapant manuellement `python manage.py my-command`, ou plus intéressant, via un batch.

Magnetiz utilise ce mécanisme pour par exemple : générer de nouvelles cartes vierges, remplacer la carte perdue d'un client, remettre les crédits d'envois de mails à une valeur définie en début de mois, envoyer des reminders par mail.

Voici un exemple simple de commande permettant de réinitialiser les crédits d'un commerçant en début de mois :

```python
class Command(BaseCommand):
    ''' Commande permettant de réinitialiser les crédits des owners à leur
    quantité initiale le 1er jour du mois.
    A exécuter dans une tâche planifiée le 1er jour du mois (ou tous les jours le cas échéant) '''

    help = ("Reinitialise les crédits des owners à leur quantité initiale le"
            "1er jour du mois")

    def handle(self, *args, **options):
        if is_first_day_of_month():
            for c in Credits.objects.all():
                c.restart_credits()
                self.stdout.write(
                    "Les crédits du owner {0} ({1}) ont été réinitialisés."\
                    .format(c.owner.user.mail_prefix, c.owner.pk))
        else:
            self.stdout.write("Ce n'est pas le début du mois, aucun crédit"
                              " n'a été mis à jour.")
```

### Problématiques de code diverses

Voici diverses problématiques de code rencontrées pendant Magnetiz, dont la solution peut présenter un certain interêt.

##### Comment parcourir dans un template, des données provenant d'objets hétérogènes, pour ensuite itérer dessus pour les afficher ?

**Example** : dans une vue, je souhaite afficher les informations d'un client (nom, sexe, nombre de points, date du 1er scan, etc.) Or, une partie de ces infos ne se trouve pas dans l'objet correspondant au client, mais est au contraire éparpillée dans d'autres objets. Je ne peux donc pas passer les objets directement au template et espérer pouvoir afficher ces données correctement ensuite.
De plus, c'est le rôle de la vue d'organiser les données avant leur passage au template.

La (ou plutôt une) solution ? Les [namedtuples](https://docs.python.org/3/library/collections.html#collections.namedtuple) :

```python
# views.py

# Création du 'modèle' de tuple
ClientInfosTuple = namedtuple( 
        'ClientInfosTuple', # Nom du tuple
        ['client_str', 'birth_date', 'gender', 'nb_points', 'first_scan',
         'last_scan', 'nb_scans', 'nb_redeemdeals'])

# ...

# Création du tuple lui-même :
client_infos = [ClientInfosTuple(
    client_str=c.user.mail_prefix,
    birth_date=c.user.birth_date,
    gender=c.user.gender,
    nb_points=Points.get_client_points(c, shop_group).nb_current_points,
    first_scan=ScanEvent.get_client_first_scan_sg(c, shop_group),
    last_scan=ScanEvent.get_client_last_scan_sg(c, shop_group),
    nb_scans=ScanEvent.get_client_scan_events_sg(
        c, shop_group, True).count(),
    nb_redeemdeals=DealRecipients.get_client_redeem_events(
        c, shop_group).count()) \
    for c in clients]
```

De cette manière, je n'ai plus qu'à passer le tuple `client_infos` à mon template, et itérer dessus exactement de la même manière qu'un objet classique. Sauf que cette fois-ci, les données sont organisées exactement comme je le souhaite.

<hr>

##### Comment générer des cartes vierges ayant des IDs tous différents, sans que ces IDs soient trop longs à taper (compromis entre sécurité et utilisabilité) ?

```python
    @classmethod
    def gen_id(cls):
        ''' Retourne un ID (unique donc) généré aléatoirement entre first_id et
        last_id. Est appelé à chaque fois qu'une carte est générée. WARN : les
        performances de cette méthode se dégraderont au fil du nombre d'objets.Les bornes pourront être changées si cela devient un problème.'''

        first_id = 10000000000 # min = 10000000000 (11 digits)
        last_id = 100000000000 # max = 99999999999 (11 digits)

        while True: # WARN : attention aux risques de boucles infinies
            # TODO : ajouter un timeout
            the_id = random.randrange(first_id, last_id) # inclus / exclu
            queryset = cls.objects.filter(pk=the_id)
            if not queryset.exists(): # ID non utilisé, on peut le prendre
                return the_id
```

<hr>

##### Comment éviter une duplication d'attributs dans des classes ayant des attributs communs ?

Dans le cas de Magnetiz, il y a 2 types d'événements :

1. Lorsqu'un client scanne sa carte, un événément de type `ScanEvent` est créé.
2. Lorsqu'un client récupère un cadeau, un événément de type `UseRewardEvent` est créé.

Ces événements ont des attributs communs (date de création, annulation) mais aussi des attributs distincts. Pour éviter la répétition des attributs communs dans des modèles similaires, il peut être intéressant d'utiliser une classe abstraite `Event`.

```python
class Event(TimeStampedModel):
    ''' Classe abstraite servant de base aux modèles ScanEvent et
    UseRewardEvent. N'est pas représenté en base puisqu'abstrait. '''

    is_cancelled = models.BooleanField(default=False)
    # ... autres attributs communs

    class Meta:
        abstract = True
```

<hr>

##### Comment modifier le comportement par défaut lors de la création d'un objet ?

Une des solutions est de surcharger la méthode `save()` de la classe, de détecter s'il s'agit d'une création ou d'une modification, et de changer le comportement par défaut.

Dans le cas de Magnetiz, un utilisateur ne doit avoir qu'une seule ligne en base indiquant ses points par commerce. Ainsi, si la ligne n'existe pas, elle doit être créée. Si elle existe, elle doit être modifiée, plutôt que d'en recréer une par erreur. Afin de forcer ce comportement, la méthode `save()` de la classe `Points` est surchargée :

```python
# models.py
class Points(TimeStampedModel):
    
    class Meta:
        # Empêche les doublons au niveau DB (sécurité)
        unique_together = ('benef_card', 'spendable_at')

    # ...

    def save(self, *args, **kwargs):
        ''' Permet de ne pas dupliquer deux entrées de Points ayant le même
        couple (owner, card) '''

        if self.pk: # save() appelée dans un update, on garde le cpt normal
            super(Points, self).save(*args, **kwargs)

        else: # save() appelée dans un create, on modifie le cpt
            if Points.objects.filter(benef_card=self.benef_card,
                                     spendable_at=self.spendable_at).exists():
                pass # On ne fait rien si le couple existe déjà
            else:
                super(Points, self).save(*args, **kwargs)

```

Une autre solution aurait pu être l'utilisation des [signals](https://docs.djangoproject.com/en/dev/topics/signals/) Django.

<hr>

##### Comment avoir un mini-système de permissions ?

Django offre une gestion des permissions très poussée. Cependant, pour l'utilisation de Magnetiz, il n'existe que 2 types d'utilisateurs qui ont les mêmes permissions : les clients, et les owners (commerçants).

Il est donc suffisant d'utiliser une simple méthode qui retourne `True` ou `False` :

```python
# models.py
class User(AbstractBaseUser, PermissionsMixin):
    
    # ...

    def is_owner(self):
    ''' Retourne True si cet utilisateur est lié à un BizProfile '''

        try:
            self.bizprofile
            return True
        except:
            return False
```

Dans une vue, il suffit ensuite d'utiliser cette methode avec le décorateur `@user_passes_test` :

```python
@user_passes_test(User.is_owner)
def viewClients(request):
    # ....
```

Si l'utilisateur n'est ici pas un owner (BizProfile), il ne pourra pas accéder à la vue.

<hr>

##### Comment identifier un utilisateur de manière sécurisée sans qu'il doive entrer ses crédentials ?

Exemple : on souhaite authentifier un utilisateur simplement après que celui-ci ait cliqué sur un lien envoyé par mail. Pour cela, chaque utilisateur dispose d'un token privé qui lui est propre :

```python
class User(AbstractBaseUser, PermissionsMixin):
    
    #...
    private_token = UUIDField(blank=True, null=True)
```

L'URL cliquable contenue dans le mail envoyée matche le REGEX suivant :

```python
url(r'^email_verification/(?P<token>[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12})$', views.email_verification, name='email_verification')
```

La vue récupère le token en tant qu'argument, et récupère l'utilisateur simplement :

```python
def email_verification(request, token):

    # On vérifie la validité du token et raise une erreur si invalide
    u = User.get_user_from_token(str_token=token)

    # ...
```

Note : pour améliorer la sécurité, on pourrait aussi utiliser un système de tokens uniques (des tokens utilisables une seule fois).

<hr>

##### Comment générer un robot.txt différent en fonction des environnements, lorsqu'on n'a pas accès au serveur ?

Réponse : le générer dynamiquement, directement via une vue Django. La liste des éléments à ne pas indexer est contenue dans une variable d'environnement, qui elle-même est récupérée dans la variable `DISALLOW_LIST` dans `settings.py`.

```python
# views.py
def generateRobots(request):
    ''' Génère la page de robots.txt en fonction de l'environnement (dynamique)
    '''

    # Récupère les arguments dans une liste
    donotvisit_list = settings.DISALLOW_LIST.split(sep=",")

    robots_str = "User-agent: *\n" # Présent dans chaque env
    for elt in donotvisit_list:
        robots_str += "Disallow: {0}\n".format(elt)

    return render_to_response(
        'dynamic_robots.html',
        {
            'robots_str': robots_str
        },
        RequestContext(request)
    )
```

```python
# urls.py
url(r'^robots.txt$', views.generateRobots, name='generateRobots'),
```

### Conclusion

Voilà donc un premier aperçu d'éléments techniques divers dans Magnetiz. Il y a de nombreux autres aspects qui pourraient être abordés. Cet article est amené à évoluer. Il sera peut être restructuré davantage aussi.

