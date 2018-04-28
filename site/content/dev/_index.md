---
title: Gem of Wallo comment c'est fait?
---
Pour ceux qui ont joué et qui souhaiteraient savoir comment le jeu a été développé, j'ai décidé d'expliquer le design de programmation utilisé.  

Le <span style="color: red;">jeu</span> repose essentiellement sur le design MVC. Je n'expliquerai pas ici qu'est-ce qu'un design MVC mais comment je l'ai appliqué à ce projet.  

**Modèle:**\
Les données (dialogues, liste des items, vie des personnages, les questions-réponses etc)

**Vue:**\
l'affichage (portion de page html)

**Contrôleur:**\
les actions utilisateurs et la logique du jeu

### Données du jeu:

**La carte du monde** (sous forme de table SQL), il s'agit d'une liste de cases comprenant:  

* la position XY
* le type de case, (foret, montagne, loup, coffre, etc...)
* les points de vie (dans le cas ou c'est une case qui contient un ennemi)
* l'item qui y sera looter
* option de verrouillage de la case (dans le cas où on tombe sur un ennemi, la case est verrouillée on ne peut plus que reculer)

\
**Les dialogues:**\
Ils sont stockés sous format XML, et un peu à la manière d'une vue html, dans un fichier séparé correspondant à une action du contrôleur. Par exemple: quand l'utilisateur parle au loup, c'est le fichier de dialogue _loup\_conversation.xml_ qui sera chargé

**Les questions / réponses:**\
comme la carte du monde, c'est une table SQL comprenant la question posée et la liste des choix/réponses possibles

**Le joueur:**\
C'est une table sql, liée à la carte du monde (pour la partie en cours) et à une session cookies/http qui contient:

* les points de vie de chaque héro dans l'équipe
* la position XY actuelle du joueur sur la carte
* la position XY précédente du joueur (utilisée au cas ou il fait reculer quand il est coincé devant un monstre)
* la question à laquelle le joueur est en train de répondre (si il y en a une)
\
**Les item/objets:**\
table sql, très simple: type de l'objet et un lien vers l'utilisateur

### Implémentation:

Maintenant qu'on a les données et qu'on sait que c'est le design MVC qui est utilisé, je pense qu'il est assez facile de comprendre comment le jeu a été implémenté.  

Admettons que l'utilisateur ait une session cookie valable, qu'il ait lancé une partie et que la carte du monde ait été générée. Dans ce cas il se trouve sur la page _game.php_ qui utilise le contrôleur _gameControlleur_. Dans le design que j'ai utilisé _gameControlleur_ vérifie que la session est valable (cookies, carte créé, etc) si ce n'est pas le cas un message est affiché et l'utilisateur est redirigé vers l'accueil.  

Si tout est bon, mon contrôleur, va chercher dans la base de données la position du joueur sur la carte et selon le type de case sur lequel il se trouve, va charger le contrôleur adapté.\
Par exemple, si il est dans la foret c'est _foretController_ qui sera chargé.\
Ensuite c'est _foretController_ qui va gérer l'action utilisateur, chercher les données et les transmettre à la vue adaptée.\
Comme c'est _gameController_ qui a appelé _foretController_, il récupère le contenu de la vue de _foretController_ pour en faire sa propre vue.  

Alors pourquoi j'ai fait comme ça? D'avoir un contrôleur qui imbrique d'autres contrôleurs et pas simplement 1 page html = 1 contrôleur. Pour la simple raison que je ne voulais pas que le joueur puisse savoir à l'avance (en lisant l'url) sur quel type de case il allait tomber. Par exemple si il choisissait d'aller à gauche, il aurait vu sur l'url de la flêche gauche "foret.php". Et aussi parce que je ne voulais pas avoir un géant fichier php contrôleur impossible à maintenir.  

Pour être sûr de ne pas me mélanger les pinceaux entre _gameController_ et celui de la case, les deux contrôleurs n'utilise pas le même nom de paramètre GET. _gameController_ répond à _**action**_ (ex: _&action=reculer_, _&action=right_) tandis que les contrôleurs des cases répondent à _**decision**_ (_&decision=combattre_, _&decision=discuter_).  

**Donc pour les actions utilisateurs:**  

* via le paramètre GET _**action**_, les déplacements sont gérés par _gameController_ (_action=left, action=right, action=reculer_)
* via le paramètre GET _**decision**_ (ou POST d'ailleurs), ce sont des actions propres à l'endroit où le joueur se trouve et donc c'est le sous contrôleur qui s'en charger (_foretController, loupController_, etc...)
* si il n'y a pas de paramètre GET, c'est la fonction index de _gameController_ qui est appelé (qui charge le "sous contrôleur" adapté qui lui-même appel sa propre fonction _index_)

En ce qui concèrne les déplacements (via _action=left, action=right, action=up, action=down_), la fonction correspondante de _gameControlleur_ est appelé (_right(), left()_, etc...). Cette fonction déplace le joueur sur la map (changement de la position XY dans la table _USER_) et puis ensuite appelle la fonction _index_ de _gameController_ qui regarde où se trouve le joueur sur la carte et puis charge le contrôleur adapté. En clair tout passe et repasse par la fonction index de gameControlleur.  

**Liste des actions de _gameControlleur_:**  

* index (expliquée au dessus)
* right (l'utilisateur demande à aller à droite)
* left (aller à gauche)
* up (en haut)
* down (en bas)
* reculer (retourner à la case précédente quand un ennemis bloque la route)

**Maintenant les contrôleurs correspondants aux cases:**\
Admettons que le joueur soit dans la forêt et ben, j'ai mon fichier _foretController.php_ qui ne contient que la fonction _index_. Tout simplement parce que quand on est dans la forêt on peut juste continuer sa route (pas d'utilisation des items, de combats, etc...).  

**La fonction _index_ de _foretController_:**  

1. charge le dialogue dans fichier _foret.xml_,
2. la vue qui correspond au fichier _foret.php_ reçoit le dialogue, est crée puis stocké dans un des membres de _foretControlleur_,
3. finalement _gameControlleur_ la récupère et en fait sa propre vue à l'intérieur de la fonction _index_ de _gameControlleur_.

**Et si il était tombé face au loup?**\
_loupControlleur_ répond aux actions suivante: index, conversation, combattre, attaquer, donner, epee, torche, guitare:  

* index (quand on arrive sur la case): le dialogue _loup.xml_ est chargé et puis la vue _loup.php_ est créé et pour finir _caseModel_ verrouille la case (l’ennemi bloque la route du joueur)
* conversation: là, c'est _loup\_conversation.xml_ qui est chargé et la vue _loup\_conversation.php_
* combattre: (affiche la liste des personnage et leur pv) même principe mais en plus on transmet à la vue les pv des personnages récupérée via _userModel_
* attaquer: idem, mais cette fois ci le contrôleur met à jour les pv du joueur et de la case (celle du loup)
* donner: _loup\_donner.xml_ pour le dialogue, _loup\_donner.php_ pour la vue et le modèle _userModel_ met à jour l'or que le joueur possède
* epee: _objectModel_ vérifie que l'utilisateur possède bien l'épée, si oui le dialogue _loup\_epee.xml_ est chargé ainsi que la vue _loup\_epee.php_
* guitare: même principe
* torche: même principe, mais en plus la case est déverrouillée (puisque la torche fait fuir le loup) et la vue _loup\_torche.php_ affiche donc les flèches pour choisir où aller ensuite

Voilà comment c'est fait pour les "ennemis". Pour le coffre c'est presque pareil sauf que la case coffre n'est pas verrouillée.  

**Questions / réponses:**\
Dans le cas du coffre, quand le joueur répond correctement à la question réponse le coffre s'ouvre et il loot quelque chose (ou rien si le coffre est vide).  

Pour ce faire dans coffreController, le modèle questionModel est utilisé pour tirer une question au hasard dans la table "question" et la transmettre à la vue. L'identifiant de la question est inscrit dans la table _user_ (pour mémoriser la question et ainsi éviter que l'utilisateur rafraîchisse la page jusqu'à ce qu'il tombe sur une question-réponse qu'il connaisse déjà)  

Quand le joueur répond via l'action _reply_, le contrôleur charge la question correspondante à l'identifiant inscrit dans la table _user_, et si la réponse est bonne objectModel est utilisé pour ajouter l'item looté (Dans le cas de l'ogre par exemple, il n'y a pas de loot mais la case est débloqué du coup l'utilisateur peux repartir)  

Et voilà en gros comment le jeu a été codé! Je ne suis pas parti directement sur ce design. J'ai commencé par programmer les tâches les plus simples, que j'ai implémenté naïvement et j'ai ensuite rapidement pensé au design à utiliser.  

Si ça vous intéresse, je met à disposition le code de librairie MVC que j'ai utilisé pour gemofwallo sur [github](http://github.com/lsmod/mini-mvc).
