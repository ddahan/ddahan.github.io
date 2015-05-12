---
layout: post
comments: true
title: Web scraping de Yelp avec Python et BeautifulSoup
tags: [python, scripting, tutorial]
---

Aujourd’hui, nous allons expliquer et réaliser ensemble un script de scraping (_explications ci-dessous_) en utilisant le site de Yelp comme exemple.
[Le code complet de ce tutoriel est disponible sur GitHub](https://github.com/ddahan/yelp-scraper).


##Qu’est-ce que le web scraping ?
Il s’agit d’une technique d’extraction du contenu d’un site web via un programme. Une fois les données extraites (dans un fichier Excel, XML, ou une base de données par exemple), on peut s’en servir comme on le souhaite.

![alt text](/res/misc/scrap.png "scraping picture")


##Un exemple avec Yelp
Jetons un coup d’oeil au site de Yelp : ce dernier permet de parcourir les commerces locaux des grandes villes, en les filtrant de manière assez poussée (notation des utilisateurs, ville, arrondissement, type de commerce, sous-type de commerce, etc.)

Pour notre part, nous allons essayer de récupérer : _Tous les commerces de telles catégories dans tels quartiers/arrondissements de Paris_`. Pour chacun des commerces, nous réupérerons le nom, l’adresse, le numéro de téléphone, l'URL, et l’ensemble des catégories auxquelles il appartient (1 commerce appartient à 1->N catégories. C’est logique : il peut vendre des bagels, mais aussi des sandwichs, ou des desserts).

**Avertissement** : les conditions générales d’utilisation de Yelp interdisent ce genre de pratique, cet exercice est donc à visée pédagogique uniquement. S'il vous arrive malheur (IP bannie, etc.), tant pis ^^

A noter que Yelp propose une API laissant penser à première vue que le scraping n’a aucun interêt supplémentaire. Cependant, cette API reste limitée, d’une part par le type de requête qu’elle permet d’effectuer, mais aussi par le nombre de résultats.

Vous êtes toujours motivés ? Alors c’est parti !

##1ère étape : analyser le fonctionnement du site à scraper

Avant de commencer la moindre ligne de code, vérifions si ce que nous souhaitons réaliser est possible (ce qui n’est pas garanti). Pour cela, direction [le site de Yelp](http://www.yelp.fr/search) !

####1e constat : le contenu

Chaque page de recherche affiche les infos principales sur les commerces, ce qui sera suffisant pour notre tutoriel, sans que nous ayons besoin de rentrer dans les pages détaillées de chaque commerce.

####2e constat : les filtres

Chaque application d’un filtre (ex : choix du type de commerce, choix des arrondissement parisiens) modifie l’URL en y ajoutant des variables de type GET. Par exemple lorsque je sélectionne la catégorie “Alimentation”, j'obtiens une URL du genre : `http://www.yelp.fr/search#cflt=food`

C’est une bonne nouvelle puisque ça permettra à notre script d’être paramétrable, en générant les bonnes URL en fonction des filtres désirés.

####3e constat : les catégories

Il semble qu'on ne puisse pas sélectionner plusieurs catégories en même temps. Donc si on veut faire une requête à la fois sur les bagels et les vétérinaires, il va falloir faire 2 recherches distinctes. C'est pas la mer à boire mais ça va nous rajouter une boucle dans notre programme

####4e et dernier constat : la pagination

Chaque page affiche 10 résultats au maximum, avant de devoir nous rendre sur la page suivante. En allant en page 2, on se rend compte que l’URL a changé avec une nouvelle variable : `start=10`. On comprend donc facilement que la page 1 contient les résultats de 0 à 9, la page 2 les résultats de 10 à 19, et la page 67 les résultats de 660 à 669.

Maintenant que ça sent plutôt bon puisque tout est paramétrable au niveau URL, il va falloir vérifier que le site fonctionne de la même manière sans JavaScript. En effet, notre script n’est pas notre browser, et il n’est pas capable d’exécuter du code côté client.

Pour cela, j’ai ouvert Firefox, désactivé le Javascript, et réessayé les étapes ci-dessus, en entrant cette-fois-ci directement les URLs complètes (avec les variables correspondant aux filtres), pour vérifier que les bons résultats s’affichent toujours correctement dans notre HTML. RAS, on va pouvoir commencer !

##2e étape : le script

L’algorithme dans sa globalité va se présenter sous la forme d’une double boucle, suivi d'une écriture dans un fichier Excel :

```
- Pour chacune des catégories :
   - Pour chaque page de résultats :
      - Générer l’URL de la page à aller scraper
      - Ouvrir cette page et passer le résultat HTML au parseur.
      - Pour chaque commerce (10 max par page) :
        - Récupèrer chacune des données souhaitées (nom, téléphone, etc.)
        - Si c'est un doublon ou une publicité, skipper ce tour de boucle.
        - Nettoyer les données (ex : retirer les trailing spaces)
        - Les placer dans un objet (de classe YelpShop par exemple)
        - Ajouter l’objet créé à une liste d’objets pour le sauver
- Pour chaque objet YelpShop créé :
  - Ecrire son contenu dans le fichier Excel
```

Et voilà !

Maintenant, nous allons étudier un peu plus en détail quelques parties intéressantes du code. Je ne vais pas coller les 200 lignes ici (ça ne serait pas intéressant), mais le script complet est disponible sur Github.

Si jamais vous souhaitez refaire le projet de votre côté, vous devrez créer un virtualenv sous Python 3 et y installer les packages suivants :

```
XlsxWriter==0.7.2
beautifulsoup4==4.3.2
requests==2.7.0
```

####Génération des URLs à scraper
Puisqu'on a vu qu'il y avait un lien direct entre le numéro de la page parcourue, et l'index à écrire en tant que variable dans l'URL, une petite fonction toute simple permet de passer de l'un à l'autre :

``` python
def page_to_index(page_num):
    ''' Transforms page number into start index to be written in Yelp URL '''
    return (page_num - 1)*10
```

Cela nous permet ensuite de construire simplement nos URLs, où vous pouvez voir qu'il s'agit simplement de concatener des variables à notre URL :

``` python
def build_yelp_url(page, c):
    ''' Builds Yelp URL for the given page and cflt to be parsed according to
    config variables '''

    url = "http://www.yelp.fr/search?&start={0}".format(page_to_index(page))
    if CITY: url += "&find_loc={0}".format(CITY)
    url += "&cflt={0}".format(c) # We assume that CFLTS list is not empty
    if PARIS_DISTRICTS: url += "&l=p:FR-75:Paris::{0}".format(
        build_arglist(PARIS_DISTRICTS))
```

####Parsing HTML

Nous utilisons ici la lib [BeautifulSoup](http://www.crummy.com/software/BeautifulSoup/bs4/doc/) qui permet de faire du parsing HTML très simplement.
Chaque commerce issu d'une page de résultat de recherche, est contenu dans une `<div class="search-result">`. Ceci ne peut pas être deviné autrement qu'en regardant le code source de Yelp (l'inspecteur Chrome est très pratique pour ça). Pour récupérer chacune des portions HTML liées, il suffit donc de créer une liste :

```python
search_results = soup.find_all('div', attrs={"class":u"search-result"}):
```

Au sein de chacune de ces portions de HTML récupérées, récupérer le nom est très similaire, à la différence ici que le nom du commerce est situé dans un lien html :

```python
for sr in search_results:
  shop_name = sr.find('a', attrs={"class":u"biz-name"}).get_text()
```
Le mécanisme est du même genre pour les autres informations à récupérer (adresse, téléphone, url, catégories, etc.)

####Retrait des annonces Yelp
Certaines pages contiennent des annonces publicitaires qui viennent polluer nos résultats, puisque ces dernières ne correspondent en rien à nos filtres de recherche. Pour les supprimer, il suffit d'observer la classe qui leur est associée, et d'écrire une fonction associée :

```python
def is_advertisement(search_result):
    ''' Return True is the search result is an add '''

    if search_result.find('span', attrs={"class":u"yloca-tip"}):
        return True
    return False
```


####Un point sur le contournement de la sécurité

Scraper des sites web n’est généralement pas apprécié par les sites en questions (ce qui est tout à fait compréhensible). Partez donc du principe que ces derniers feront tout leur possible pour vous compliquer la vie. Voici deux mesures très simples qui vont réduire les chances d’être identifié comme un “robot” :

#####Durée d’attente aléatoire entre chaque ouverture de page

Si votre script ouvre 30 pages en 5 secondes, vous donnez le baton pour vous faire battre puisqu’un humain n’agirait à priori jamais de la sorte. L’idée est donc de générer un sleep aléatoire à la fin de chaque tour de la boucle principale :

``` python
def r_sleep():
    ''' generates a random sleep between 2.000 and 10.000 seconds '''

    length = float(randint(2000, 10000)) / 1000
    mylog("Safety Random Sleep has started for {0} sec".format(length))
    sleep(length)
    mylog("Safety Random Sleep is over")
```

#####Passage d’en-têtes à la requête HTTP

Comme vous le savez probablement, une requête HTTP créée par votre browser contient des en-têtes permettant au serveur de récupérer des informations telles que le navigateur utilisé, la version de votre OS, et beaucoup d’autres.
Dans notre script, nous allons donc spécifier manuellement ces en-têtes afin que le serveur pense qu’il s’agisse d’un navigateur classique, avant d'effectuer la requête en question :

``` python
fake_headers = {
    # Headers taken from Chrome spy mode
    'Connection': 'keep-alive',
    'Accept': 'text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,*/*;q=0.8',
    'User-Agent': 'Mozilla/5.0 (Macintosh; Intel Mac OS X 10_9_5) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/42.0.2311.135 Safari/537.36',
    'Accept-Encoding': 'gzip, deflate, sdch',
    'Accept-Language': 'fr,en-US;q=0.8,en;q=0.6'}
r = requests.get(cur_url, headers=fake_headers)
```

Notez qu’un site qui voudrait vraiment vous empêcher de scraper à tout prix, pourrait trouver d’autres moyens plus filous, comme un script JS qui analyserait les mouvements du pointeur de votre souris. Il y a d’ailleurs sûrement de nombreux autres méthodes que je ne connais pas.

## Conclusion

C'est terminé. N'hésitez pas à jeter un coup d'oeil au [code complet](https://github.com/ddahan/yelp-scraper), car les bouts de code expliqués ne sont pas suffisants pour tout comprendre.

Pour améliorer ce script, on pourrait par exemple :

- rendre les fichiers xlsx créés évolutifs (on peut lancer le scrap pour mettre à jour uniquement les résultats qui ont changé, sans réécrire le fichier)
- écrire dans le fichier Excel au fur et à mesure plutôt qu'à la fin

Avez vous d’autres idées d’amélioration ?

Notez aussi que ce genre de script n'a aucune garantie de fonctionner à travers le temps, puisque la moindre modif du côté de Yelp casserait son fonctionnement. N'hésitez pas à me le signaler si c'était le cas.

J'espère que ce 1er tutoriel vous aura été utile, à bientôt !

