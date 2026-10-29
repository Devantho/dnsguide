Construire un serveur DNS en Rust
=================================

> **À propos de cette traduction**
>
> Traduction française du guide *Building a DNS server in Rust* d'Emil
> Hernvall, publié sous licence MIT (voir [LICENSE](LICENSE)). La version
> originale en anglais est disponible sur le dépôt de l'auteur,
> [EmilHernvall/dnsguide](https://github.com/EmilHernvall/dnsguide), ainsi
> que dans l'historique git de ce dépôt.
>
> La traduction ajoute des **notes du traducteur** (« N.d.T. ») pour expliquer
> les notions techniques, un [glossaire](GLOSSAIRE.md) et la section
> [Avant de commencer](#avant-de-commencer-ajout-du-traducteur) ci-dessous.

Internet repose sur des fondations conceptuelles riches, faites de nombreuses
idées passionnantes qui lui permettent de fonctionner tel que nous le
connaissons. L'une des plus astucieuses est le DNS. Avant son invention, tout le
monde sur Internet (ce qui, il faut bien l'admettre, ne représentait pas grand
monde à l'époque) s'appuyait sur un fichier partagé appelé HOSTS.TXT, tenu à
jour par le Stanford Research Institute. Ce fichier était synchronisé
manuellement par FTP, et à mesure que le nombre de machines augmentait, le
rythme des modifications augmentait lui aussi, rendant le système intenable. En
1983, Paul Mockapetris s'est attelé à trouver une solution durable au problème
et a fini par concevoir et mettre en œuvre le DNS. Que sa création ait pu passer
de quelques milliers d'ordinateurs à l'Internet que nous connaissons aujourd'hui
témoigne de son génie.

Avec le triple objectif de comprendre le DNS en profondeur, de réaliser quelque
chose d'intéressant en Rust et de répondre à quelques envies personnelles, je me
suis lancé à l'origine dans l'écriture de mon propre serveur DNS. Ce document
n'est pas une chronique fidèle de ce parcours, mais plutôt une version idéalisée,
débarrassée de tous les détours que j'ai fini par faire. Nous allons
progressivement construire un serveur DNS complet, en partant des principes de
base.

 * [Chapitre 1 - Le protocole DNS](chapter1.md)
 * [Chapitre 2 - Construire un résolveur minimal (*stub resolver*)](chapter2.md)
 * [Chapitre 3 - Ajouter d'autres types d'enregistrements](chapter3.md)
 * [Chapitre 4 - Mon premier serveur DNS](chapter4.md)
 * [Chapitre 5 - Résolution récursive](chapter5.md)
 * [Glossaire](GLOSSAIRE.md) *(ajout du traducteur)*

Exemples
--------

Chaque chapitre est accompagné d'un exemple qui contient le code complet
jusqu'à ce stade du guide ; ils sont nommés de `sample1.rs` à `sample5.rs`. Pour
le premier chapitre, par exemple, on le lance avec
`cargo run --example sample1`.

Historique des révisions
------------------------

 * Juin 2020 - Correction d'une faille de sécurité dans `read_qname`, qui
   permettait à un paquet malveillant de déclencher une boucle infinie.
   Modernisation du code pour le conformer aux pratiques actuelles de Rust, et
   correction de diverses inefficacités disgracieuses.
 * Juillet 2016 - Version initiale

Avant de commencer (ajout du traducteur)
----------------------------------------

### Ce que vous allez apprendre

Le guide explique, en construisant pas à pas un programme qui fonctionne,
comment un ordinateur trouve l'adresse d'un site web à partir de son nom :

| Chapitre | Ce qu'on y apprend sur le DNS                                                                      | Ce que fait le programme à la fin du chapitre                                  |
| -------- | -------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------ |
| 1        | Le format exact d'une question et d'une réponse DNS, octet par octet.                              | Il lit une réponse DNS enregistrée dans un fichier et l'affiche en clair.      |
| 2        | Comment envoyer soi-même une question à un serveur DNS.                                            | Il interroge le serveur DNS public de Google et affiche la réponse.            |
| 3        | Les principaux types d'enregistrements (adresse, alias, messagerie, IPv6…).                        | Il comprend ces nouveaux types de réponses.                                    |
| 4        | La différence entre serveur « faisant autorité » et serveur « cache ».                             | Il devient un serveur DNS relais, qui transmet les questions à Google.         |
| 5        | Le parcours complet d'une résolution, des serveurs racine jusqu'au serveur du domaine.            | Il devient un serveur DNS autonome, qui trouve les réponses par lui-même.      |

### Parcours de lecture conseillé

Le guide s'adresse d'abord à des développeurs, mais il peut se lire sans rien
installer : les explications, les commandes et leurs résultats figurent dans le
texte.

- **Pour comprendre le fonctionnement du DNS sans entrer dans le code** : le
  chapitre 1 jusqu'à la section *BytePacketBuffer*, le tableau des types
  d'enregistrements au début du chapitre 3, le début du chapitre 4 (les deux
  types de serveurs) et le chapitre 5 jusqu'à la section *Étendre DnsPacket*.
  Les notes du traducteur suffisent pour suivre ces passages.
- **Pour suivre aussi le programme** : tout lire dans l'ordre. Les notes du
  traducteur donnent les repères nécessaires pour lire le code Rust, et le
  [glossaire](GLOSSAIRE.md) récapitule les termes.

### Exécuter les exemples

1. **Installer Rust** depuis [rustup.rs](https://rustup.rs) en suivant les
   instructions de la page (un installateur sous Windows, une ligne de commande
   sous macOS et Linux). Cela installe aussi `cargo`, l'outil qui compile et
   lance les programmes Rust.
2. **Récupérer ce dépôt** : sur GitHub, bouton *Code* puis *Download ZIP*, ou
   avec `git clone`.
3. **Ouvrir un terminal dans le dossier du dépôt** et lancer l'exemple voulu,
   par exemple : `cargo run --example sample2`. La première exécution compile le
   programme, ce qui prend quelques secondes.
4. Les chapitres 1, 4 et 5 utilisent aussi les outils en ligne de commande
   `dig`, `nc` (netcat) et `hexdump`. Ils sont disponibles sous Linux et macOS ;
   sous Windows, le plus simple est de passer par WSL (le sous-système Linux de
   Windows).

Les exemples 2 à 5 ont besoin d'un accès à Internet. Certains réseaux
d'entreprise bloquent les requêtes DNS directes vers l'extérieur (port 53) : si
un programme semble attendre indéfiniment, essayez depuis une autre connexion.

### Conventions de la traduction

- Les encadrés **« Note du traducteur »** et **« N.d.T. »** sont des ajouts :
  ils ne figurent pas dans l'original. Quand l'original contient une imprécision
  ou un passage daté, la note le signale sans modifier le texte de l'auteur.
- Dans le code, **les commentaires sont traduits** ; les noms de variables, de
  fonctions et les messages affichés par les programmes restent en anglais, car
  ils font partie du programme. Le [glossaire](GLOSSAIRE.md) en donne la
  signification.
- Les sorties des commandes (`dig`, `hexdump`…) sont reproduites telles
  quelles. Les adresses IP, TTL et dates qu'elles contiennent datent de la
  rédaction originale (2016) : vous obtiendrez d'autres valeurs aujourd'hui.
- Les termes anglais d'usage courant sont donnés en italique entre parenthèses
  lors de leur première apparition (par exemple : tampon (*buffer*)).
