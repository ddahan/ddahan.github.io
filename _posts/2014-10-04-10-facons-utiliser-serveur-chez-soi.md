---
layout: post
comments: true
title: 10 façons d'utiliser un serveur chez soi
tags: [home-server, virtualization, unix]
---

Il y a quelques années, j'ai fait l'acquisition d'un <a href="http://www.amazon.fr/HP-658553-421-MicroServer-processeur-Contr%C3%B4leur/dp/B005LRCASM">HP MicroServer</a> pour environ 250€. Quésako ? C'est un serveur comme un autre, mais sa taille, son prix et sa fiabilité font de lui un compagnon parfait pour une petite start-up ou un particulier.


Avec tous les différents usages qu'on peut en faire, vous allez vous rendre compte que ce n'est pas qu'un achat impulsif de geek qui souhaite à tout prix transformer son salon en datacenter ! Ce n'est tout de même pas le meilleur argument de drague d'une jolie fille (_"Hey mate mon serveur, il en jette hein ?"_), je vous le concède.

Allez c'est parti pour une liste non exaustive d'idées sympatiques, certaines étant plus évidentes que d'autres. Par contre, on n'expliquera pas dans ce post **comment** les mettre en place, vous voilà prévenus !

####1. Serveur de fichier (NAS)

La tendance est aux laptops munis de SSD, avec en général une capacité limitée Si comme moi, vous stockez de nombreuses photos ou vidéos, vous aurez besoin de stockage supplémentaire. Et bien sûr, vous voulez que ce stockage soit accessible depuis n'importe où dans la maison (laptop, desktop, smartphone), voire de l'exérieur. Le bon vieux NAS est votre sauveur !

Pour que ce système soit viable, il faudra tout de même vous assurer d'avoir un réseau local de bonne qualité. Privilégiez tant que possible une connexion Ethernet au Wi-Fi si vous voulez une meilleure vitesse de transfert.

Si ces données on de valeur pour vous, vous pourrez facilement mettre en place du [RAID1](http://fr.wikipedia.org/wiki/RAID_%28informatique%29) pour les protéger. On verra plus bas un autre moyen pour ça (suspens..). A noter que le micro serveut dont je vous parlais peut accueillir jusqu'à 6 disques, ça aide.

####2. Serveur FTP

Vous avez besoin de partager 10 Go de vidéos de vacances filmées avec votre toute nouvelle GoPro ? L'upload risque d'être long non ? Vous n'avez non plus envie qu'une société externe puisse accéder à vos photos un peu limites ? Une solution simple consiste à créer un serveur FTP. Il suffit ensuite de créer un ou plusieurs accès pour les personnes supposées y accéder. Avec les bons outils, il y en a pour 5mn chrono.

**Warning** : ce protocole est un peu obsolète d'un point de vue sécurité, donc lorsqu'il n'est pas utilisé, pensez à le désactiver, ou mieux, utilisez un protocole du même genre mais sécurité (FTPS par exemple).

####3. Client BitTorrent

Vous êtes accroc au téléchargement ? Vous êtes du genre à laisser votre ordi bruyant et brûlant tourner la nuit pour télécharger ? Pourquoi ne pas utiliser votre serveur pour jouer ce rôle ? Lui au moins, il tourne 24/24 par défaut, et si vous avez un NAS, les fichiers seront déjà dessus.

De plus, certains programmes/scripts permettent même de télécharger automatiquement certains fichiers nouvellement disponibles, via des flux RSS. En gros, vous vous reveillez, et les fichiers que vous vouliez vous attendent tranquilement, la belle vie non ?

####4. Serveur de sauvegarde (Rsync)

Imaginez votre maman qui vient tout juste d'apprendre à transférer ses photos dans le dossier "photo" de son ordinateur. Vous espérez quand-même pas qu'elle pense d'elle-même à les sauvegarder dans le claoud ?

L'astuce, vous lui installez un client <a href="http://fr.wikipedia.org/wiki/Rsync">Rsync</a> sur son ordinateur, et vous, vous vous occupez de la partie serveur. Concrètement, à des horaires définis, le contenu de son dossier "photo" sera synchronisé sur votre serveur. Et comme Rsync est intelligent, il va synchroniser juste le différentiel entre les deux dossiers, pour ne pas dévorer toute la bande passante. Evidemment, ça peut fonctionner en local, avec votre maman chez vous, mais aussi avec votre grand mère qui habite à 50km, en passant cette fois-ci par Internet. Dans ce cas, on pensera à utiliser un VPN pour assurer un minimum de confidentialité sur les données transférées. Notez que Rsync est un logiciel libre, gratuit, multi OS et fiable.

####5. Hébergeur de site web

La fausse bonne idée. Pour être clair, il faut **vraiment** avoir une bonne raison pour héberger un site chez soi plutôt que sur via un hébergeur professionnel.
Tout simplement parce que vous ne pourrez pas atteindre une qualité de services, notamment en terme de disponibilités, équivalente à celle d'un hébergeur, dont c'est le métier. Une prise débranchée en passant l'aspirateur, ça arrive ! Une coupure de courant EDF ou une erreur de manip aussi. Si c'est pour un site sans traffic, pourquoi pas, mais sinon, on oublie !

####6. Serveur d'impression (CUPS)

Vous avez une bonne vieille imprimante réseau et vous voudriez qu'elle puisse être utilisée par toute la famille ? Un petit serveur d'impression [CUPS](http://fr.wikipedia.org/wiki/Common_Unix_Printing_System) est bien pratique. Ce dernier va permettre à n'importe qui connecté au réseau local d'installer l'imprimante facilement. Si vous avez une imprimante Wi-Fi, je vous avoue que ce protocole ne sera d'aucune utilité.

####7. Serveur Git

Vous développez en équipe et vous voulez votre propre serveur Git ? Pourquoi pas ! Là encore, le choix de ne pas prendre un serveur dans le cloud, à l'heure où des services commes [Github](https://github.com/) ou [Bitbucket](https://bitbucket.org/) répondent parfaitement à ce besoin, doit être mûrement réfléchi.

####8. Hyperviseur (ESX, Xen, ...)

Imaginez votre serveur comme un terrain de jeu géant, où vous pouvez ajouter des machines virtuelles avec leur OS, faire tous les tests que vous voulez, puis les supprimer ou redimensionner à souhait ? N'est-ce pas un gain de fléxibilité énorme ?

Mais ce qu'il faut bien comprendre, c'est que le fait d'installer un hyperviseur va bien vous permettre de faire tous les usages qu'on a vus ci-dessus. Mais tout en permettant de cloisonner proprement les OS en fonction des usages que vous en faites.

**Note** : Si vous voulez mettre un ESX, il faudra 4Go de RAM minimum sur votre serveur.

####9. VPN

Vous avez décidé de devenir agents secrets avec votre girl friend (oui je pars du principe que vous, cher lecteur, êtes masculin), et vous décidez que tous vos échanges ne devront jamais être interceptés par les méchants américains de la NSA ? Pourtant, vous habitez à 300 Km l'un de l'autre et votre seul moyen de communication est Internet ? Ou vous êtes tout simplement paranoïaque ? Pas de soucis, vous allez utiliser un VPN, et donc créer un tunnel qui simulera une connexion locale avec votre partenaire, alors que le flux sera encapsulé dans un protocole sécurisé. Si des données sont interceptées, elles seront indéchiffrables. Alors oui, ça pourrait être géré directement côté client, mais si on peut alléger sa machine en déportant ça sur le serveur, pourquoi se gêner ?

Le logiciel libre le plus connu est <a href="https://openvpn.net/">OpenVPN</a>

####10. Serveur TeamSpeak

Vous êtes un gamer régulier et vous avez une connexion internet sans faille ? Pourquoi ne pas créer votre propre serveur TeamSpeak ? Ca vous évitera de payer, et contrairement à l'hébergement, c'est à priori beaucoup moins critique !

#####Et vous, comment utilisez-vous votre home server ?
