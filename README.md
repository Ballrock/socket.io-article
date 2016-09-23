# Socket.io

Suite à la réalisation d'un projet de jeu sous forme de webapp en Angular 2, j'ai eu l'occasion/la possibilité de pousser le concept du dit jeu vers le multijoueur. Cela afin d'amener de l'interactivité entre les joueurs connectés à un instant T, du "Temps réél" en somme.
Afin de réaliser cela sur une web-app je me suis tourné vers les [WebSocket](https://fr.wikipedia.org/wiki/WebSocket) et plus particulièrement vers `socket.io` dans mon cas. 

## Présentation
[socket.io](http://socket.io/) est donc une des implémentations du protocole WebSocket utilisant du NodeJS coté serveur et du Javascript coté Client. Elle possède deux "librairies" distinctes pour chacun des deux côtés. 
Le site internet du framework étant très complet je vous invite à y faire une tour.
Tout le principe de la communication repose sur l'échange de "messages" possédant un identifiant et un contenu sur un socket donné. 
L'exemple type d'un échange entre un client et un serveur peut être représenté de la façon suivante :

1. Serveur déploie le websocket 
  * Ouverture du websocket coté serveur
2. Client 1 se connecte au serveur WebSocket 
  * Création du socket
3. Client 1 envoie le message "doAttack" avec comme paramètre le joueur ainsi que son adversaire l'adversaire
  * Envoi du message `["doAttack" moi adversaire]`
4. Serveur réceptionne le message et le traite par exemple en enregistrant le coup et en notifiant l'"attaqué"
  * Traitement du message `["doAttack" attaquant attaqué]`
  * Enregistrement du coup
  * Envoi du message `["takeAttack" attaquant résultat]` au socket de l'"attaqué"
5. Client 2 (déjà connecté) reçoit le message de l'attaque et le traite
  * Traitement du message `["takeAttack" attaquant résultat]`
  *  …
6. …

Et ainsi de suite, le tout se gère via un système callback JS qui sont exécutés à la réception des messages.

Un fonctionnement simple mais pas simpliste qui permet de réaliser beaucoup de choses. 

Il est à noter qu'il existe dans le framework un système de "room" donnant la possibilité de réunir plusieurs sockets ensembles. Ce qui permet entre autre d'envoyer directement un message à tous les sockets de cette "room".
Dernière petite chose, socket.io client intègre un système de "heartbeat" permettant au client de savoir si le serveur est toujours là et en cas de déconnexion, de s'y reconnecter automatiquement.

## Coté Serveur
### Prérequis 
* http.server déjà présent (via express par exemple)

``` bash
npm install --save socket.io
```
Une fois que la dépendance est installée c'est assez simple pour le démarrage du serveur, à savoir :

```js
import socketIO = require("socket.io");
import express = require("express");
import http = require("http");

let app = express();
logger.debug("Lancement du serveur HTTP");
let server = http.createServer(app);
[…]
logger.debug("Lancement du serveur WebSocket");
let io = socketIO(server);
[…]
```

> Petit aparté, les plus fins d'entre vous aurons surement remarqué que la syntaxe utilisée n'est pas très conventionnelle. En effet, le serveur de ce projet a été codé en [Typescript](https://www.typescriptlang.org/). Si vous voulez intégrer `socket.io` en Typescript je vous conseille d'utiliser [typings](https://github.com/typings/typings) et plus particulièrement la définition disponible sur  [DefinitelyTyped](https://github.com/DefinitelyTyped/DefinitelyTyped/tree/cc3d223a946f661eff871787edeb0fcb8f0db156/socket.io) (c'est celle qui est utilisée ici). Même pour les éventuels utilisateurs de TypeScript 2.0, en effet à l'heure où j'écris ces lignes le [@types](https://www.npmjs.com/~types) de socket.io n'est pas encore disponible sur npm.

Et voilà notre serveur est prêt à recevoir des requêtes WebSocket. Quant à les traiter c'est une autre affaire, il va falloir que nous déclarions les fonctions de callback pour chaque message.
Démarrons par la connexion d'un client, action de base mais qui au final, suit la même logique qu'un message :

```js
let self = this;
this.io.on("connection", function (socket: SocketIO.Socket) {
	self._logger.info("Un joueur s\'est connecté ! ")
});
```

Quelques mots sur ce bout de code avant d'enchainer sur la gestion des messages:
* On utilise la variable self pour continuer à appeler le contexte de cet objet (du service en l'occurrence ici) même depuis la fonction de callback. [Piqure de rappel](https://developer.mozilla.org/fr/docs/Web/JavaScript/Reference/Fonctions)
* Ce service est instancié par mon serveur (index.ts) qui lui passe la référence de `socket.io` (d’où le this.io)
	
Revenons donc à socket.io. Ici on initialise la fonction de callback lors de la connexion d'un joueur à notre serveur,  cette fonction prend en paramètre le socket du dit joueur. Cette dernière information est importante pour la suite.

De là nous allons commencer à pouvoir définir les comportements lors de la réception de messages spécifiques. Après un peu de développement nous obtenons quelque chose qui ressemble à ça :

```js
let self = this;
this.io.on("connection", function (socket: SocketIO.Socket) {
	let nbrJoueurs = self.getNbrJoueurs();
	// Tout le monde - Mise à jour du nombre de joueurs
	socket.emit("updateNbJoueurs", nbrJoueurs);
	socket.broadcast.emit("updateNbJoueurs", nbrJoueurs);
	self._LOGGER.info("Un joueur s\'est connecté ! " + nbrJoueurs + " joueur(s) connecté(s).");
				
	socket.on("disconnect", function() {
		nbrJoueurs = self.getNbrJoueurs();
		self._LOGGER.info("Un joueur s\'est déconnecté ! " + nbrJoueurs + " joueur(s) connecté(s).");
		// Tout le monde - Mise à jour du nombre de joueurs
		socket.emit("updateNbJoueurs", nbrJoueurs);
		socket.broadcast.emit("updateNbJoueurs", nbrJoueurs);
	});
	
	socket.on("postTentative", function(equipe, joueurParam){
		try {
			let joueur = new Joueur().deserialize(joueurParam);
			self.handleTentative(new Equipe().deserialize(equipe), joueur, socket);
			[…]
			socket.emit("updateEssaisRestants", nbRestant);
		} catch (e) {
			self.notifyClientMessage(e.message, "error", "Tentative", "Une erreur est survenue lors de l\'analyse de la tentative, veuillez reesayer plus tard", socket);
		}
	});
	
	socket.on("getHistorique", function(){
		// Actuel - Mise à jour historique
		socket.emit("updateFullHisto", self.getClassement());
	});
	
	[…]
});
```
On peut voir ici que la déclaration des messages est faite pour le socket qui est récupéré à la connexion du joueur.
Plusieurs actions sont réalisées ici :
* `disconnect` (qui est un message) / connection :  met à jour le compte de joueurs pour tous les autres joueurs.
* message `postTentative` : traite la tentative émise par un joueur.
* message `getHistorique` : émet un message updateFullHisto avec un classement actuel en paramètre

Un petit élément à savoir concernant disconnect, ce message est appelé automatiquement à la déconnexion (fermeture du navigateur, départ du site, etc…) ce qui peut être très pratique pour faire un système de "destructeur" ou notifier les autres joueurs (comme ici)

Au final, trois méthodes de socket.io sont utilisées ici, elles sont les "piliers" du framework: 
* `Socket.on` qui permet de definir le callback pour un message donné,
* `Socket.emit` qui émet un message au socket actuel,
* `Socket.broadcast` qui émet un message à tous les sockets sauf l'actuel.
* 
Et c'est tout… on peut déjà aller très loin avec cette approche.

> Concernant Typescript, vous avez peut-être remarqué que je "deserialize" les objets arrivant d'un message, en effet ceux-ci sont des plain-object, et la conversion vers un objet Typescript n'est pas native, c'est le petit côté négatif de l'utilisation de TS.

Aller, une dernière petite astuce, on peut récupérer la liste des joueurs connectés via cette petite fonction simple :

```js
private getNbrJoueurs() {
	let srvSockets = this.io.sockets.sockets;
	return Object.keys(srvSockets).length;
}
```

## Coté Client

Je vais survoler cette partie car au final, on retrouve exactement le même principe de réception et d'émission que coté serveur. La seul différence est que cette fois-ci on communique uniquement avec un acteur, le serveur.

``` bash
npm install --save socket.io-client
```

Une fois installé et inclus la librairie dans nos js/ts/html et on initialise la connexion :

```js
this._socket = socketIO(window.location.protocol + "//" + window.location.host);
```

Alors oui c'est un peu facile dans mon cas, mais cette initialisation à pour mérite de marcher la plupart du temps (en somme si le serveur socket.io cible est également le serveur qui déploie l'application.).
 
Une fois cela réalisé on peut définir comme d'habitude nos chers callback :

```js
// Mise à jour du nombre de connectés.
this._socket.on("updateNbJoueurs", function(nbJoueurs) {
	self._nbJoueurs = nbJoueurs;
});
```

Je pense que vous commencez à comprendre le concept pas besoin d'insister… ^^

## Conclusion
Bien sûr il est possible d'aller beaucoup plus loin avec `socket.io.` Cependant, grâce à son fonctionnement très simple il est possible très rapidement de réaliser une communication multi-parties en pseudo temps réel (modulo la latence bien sûr).
`socket.io` m'a permis d'intégrer du multijoueur dans mon projet d'une façon très aisée et, que je trouve pour ma part, assez élégante. 
Coté performance je n'ai pas pu énormément pousser l'application mais, de ce que j'ai vu, ça tiens très bien la route (notre cher NodeJS y étant pour beaucoup). Avec 45 joueurs simultanés sur la web-app, mon tout petit [Kimsufi](https://www.kimsufi.com/fr/serveurs.xml) ne dépassait pas les 10% CPU, et 200mo de RAM OS inclus (CentOS 7).

Cependant, son côté facile d'accès peut également être son défaut, les messages pouvant rapidement se multiplier et se complexifier aux dépens de la maintenabilité de l'application.

Dernier point, il est très important de faire attention à la quantité d'informations envoyées au client, notamment pour des questions de performances côté client. En effet, et c'est encore plus vrai pour un web-app, un message envoyant par exemple l'historique des coups joués, peut vite devenir énorme si beaucoup de monde est connecté et actif. Mais cette dernière remarque tient plus de la logique que du framework, j'en conviens :)
