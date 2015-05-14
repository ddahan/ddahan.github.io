---
layout: post
comments: true
title: Générer des cartes à QR code unique avec Python
tags: [python, scripting, tutorial]
---

Dans le cadre de [Magnetiz](http://www.magnetiz.fr), j'ai été amené à fabriquer des cartes uniques pour chacun des différents clients. Je vous propose aujourd'hui de refaire ça ensemble, pas à pas, avec un simple script Python, et les libs associées. [Le code complet est disponible sur GitHub](https://github.com/ddahan/qrcode-cards-generator).


Voici le résultat auquel nous devrions arriver :

<img src="/res/misc/qr-result.png" class="img-responsive">

Ici, nous avons généré 10 cartes avec un identifiant unique, le QR code correspondant, et un texte avec l'identifiant juste en dessous du QR code.

#### Création d'un fond de carte

La 1ère étape n'a pas grand chose à voir avec de la programmation, puisqu'il vous faudra sélectionner un fond de carte. Voilà à quoi ressemble le fond en question pour moi :

<img src="/res/misc/qr-background.png" class="img-responsive">

Notez que j'utilise un format TIFF, mais pillow (la lib Python de gestion des images de Python) vous laissera utiliser quasiment n'importe quoi.

#### Installation de l'environnement

Préparons l'environnement de travail :

``` bash
mkdir qr_card_generator
cd qr_card_generator
virtualenv -p /le/chemin/de/python3 venv # Init du virtualenv en spécifiant le chemin de Python3 (sur mon système, Python 2.7 est par défaut)
source venv/bin/activate # Activation du virtualenv
pip install qrcode pillow # Installation des deux libs nécessaires
touch card-generator.py
```

#### L'algorithme principal

Voici comment va s'articuler l'algorithme nous permettant de générer autant de cartes que souhaitées. Il est vraiment très simple :

```
- Création d'un dossier permettant d'accueillir les cartes créées
- Récupérations des dimensions de l'image de fond
- Création d'un objet de type Font, pour écrire du texte sur la carte 
- Pour chaque carte à générer :
    - copie de l'image de fond dans cette nouvelle image
    - création du QR code contenant l'UUID comme données
    - dessin du QR code, placé au bon endroit sur la carte
    - écriture textuelle de l'ID, juste en dessous du QR code
    - sauvegarde du fichier image dans le dossier précédemment créé
    - suppression de l'image de la RAM (inutile de la garder)
```

Maintenant, jetons un coup d'oeil au code des étapes principales. N'hésitez pas à suivre ce post avec le [code complet](https://github.com/ddahan/qrcode-cards-generator) ouvert à côté.

#### Génération des identifiants de cartes

Nous voulons que chaque carte soit unique, pour les associer par exemple à une personne. Dans mon cas, mon application Django génère des identifiants entiers de 11 chiffres en s'assurant qu'ils sont uniques on les comparant avec ceux existant dans une base de données. 

Mais puisqu'ici nous n'avons pas forcément ce besoin, nous allons utiliser les UUID (Universally Unique IDentifiers). Je ne vais pas rééexpliquer ici comment ils fonctionnent puisque [d'autres l'ont déjà parfaitement fait](http://sametmax.com/quest-ce-quun-uuid-et-a-quoi-ca-sert/). Retenez juste qu'un UUID est une suite de chiffres et de lettres suffisamment longue pour qu'il soit statistiquement impossible de générer 2 fois le même. Pratique !

Voici la fonction simplissime dont nous nous servirons :

``` python
def create_UUIDs(qty):
    ''' Return a list of `qty` UUIDs '''
    return [str(uuid4()) for _ in range(qty)]
```

#### Injection des données dans un QR code

Comment injecter notre UUID dans un QR code ?

``` python
def data_to_qrcode(data):
    ''' Return a qrcode image from data '''

    qrc = qrcode.QRCode(error_correction=qrcode.constants.ERROR_CORRECT_Q,
                        box_size=8,
                        border=0)
    qrc.add_data(data)
    qrc.make(fit=True)
    img = qrc.make_image()

    return img
```

Quelques points à noter :

- Le paramètre `error_correction` peut prendre 4 valeurs et sert à ajouter de la redondance dans le QR code. `Q` est le 3e niveau et ajoute 25% de redondance, ce qui permettra à nos QR codes d'être lisibles même si une partie est invisible. Vous ferez le test ensuite, c'est assez fun. 
- Le paramètre `box_size` est comme son nom l'indique, lié à la taille de l'image générée. Adaptez en fonction de vos besoins.
- Le paramètre `border` est très important puisqu'il est désigne la taille du "cadre de blanc" à dessiner autour du QR code. C'est primordial puisque l'algorithme de détection d'un QR code risque d'être paumé s'il n'y a pas de blanc autour. Par défaut, `border` vaut 4. Je l'ai mis à 0 parce que j'avais déjà un cadre blanc assez large sur mon fond de carte, mais ce n'est pas forcément quelque chose à faire !

#### La boucle principale

La boucle principale du script est la suivante :

``` python
for cpt, elt_id in enumerate(input_ids):

    # Copy the background to the new image, then draw it
    new_bg = copy(bg_image)
    draw = ImageDraw.Draw(new_bg)

    # Generate a QR code as image
    qr_image = data_to_qrcode(elt_id)
    qr_x, qr_y = qr_image.size # Get width/height of QR code image

    # Paste the QR code image to the background
    # DELTA are used to change image placement (default is centered)
    DELTA_X = 0 # Keep the QR code horizontally centered
    DELTA_Y = 107 # Place the QR code according to the background
    new_bg.paste(qr_image,
                 (bg_x//2 - qr_x//2 + DELTA_X, bg_y//2 - qr_y//2 + DELTA_Y))

    # (Optinnal) Draw the ID on the card, just above the QR code
    draw.text((bg_x/2 - 240, 810), # No rule here, just try and adapt
              text=elt_id,
              font=fnt,
              fill=0)

    # Save the picture
    full_path = os.path.join(output_path, elt_id + '.' + OUTPUT_FORMAT)
    new_bg.save(full_path, OUTPUT_FORMAT)

    # Delete in-memory objects (they're not necessary anymore)
    del new_bg
    del qr_image
```

il n'y a rien de très complexe ici, notons tout de même que :

- A chaque tour de boucle, on prend le fond d'image pour le copier dans une nouvelle image à créer
- Au niveau du placement du QR code sur le fond de carte, vous pourriez faire simple en spécifiant directement les coordonnées, mais ce n'est pas forcément une bonne idée puisque si la taille de votre fond de carte change, tout votre placement ne fonctionnera plus. Il faut donc ici essayer tant que possible de raisonner en relatif plutôt qu'en absolu. Pour cela, on place l'image au centre par défaut (X vaut alors `bg_x//2 - qr_x//2`, et Y vaut `bg_y//2 - qr_y//2`) (Attention à la division entière qui s'écrit bien `//` et non pas `/` avec Python 3). Ensuite, on paramètre un décalage en largeur `DELTA_X` et en hauteur `DELTA_Y` permettant d'effectuer des décalages.
- Notez que je me contredis juste après en dessinant le texte puisque je le place de manière totalement absolue. Je n'ai pas vraiment le choix parce que je ne sais pas à l'avance quelle va être la largeur du texte en question. D'ailleurs, en écrivant un UUID à la place d'un ID de 11 caractères (ce que j'avais fait pour Magnetiz), j'ai évidemment dû changer manuellement le placement. Bon, c'est pas la mer à boire non plus...
- Après avoir sauvegardé l'image sur le disque au bon endroit, on n'oublie pas de la supprimer dans la mémoire, parce que la garder pourrait être gênant si il y avait 100000 cartes de 10 Mo chacune.

#### Le mot de la fin

Maintenant que votre script vous permet de générer des cartes à QR code unique, vous pouvez vous amuser à les imprimer, puis les scanner avec votre smartphone pour voir le résultat. Ca fonctionne parfaitement !

Et vous, pour quel usage pourriez vous avoir besoin des QR codes en python ?
A bientôt !
