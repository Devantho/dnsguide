1 - Le protocole DNS
====================

> **Note du traducteur — À quoi sert le DNS ?**
>
> Sur Internet, les machines se reconnaissent par des **adresses IP** (par
> exemple `216.58.211.142`), un peu comme des coordonnées cadastrales. Les
> humains, eux, retiennent des **noms** (`google.com`). Le DNS (*Domain Name
> System*, « système de noms de domaine ») est l'annuaire qui fait la
> correspondance entre les deux. Chaque fois que vous tapez une adresse dans un
> navigateur, votre ordinateur pose une question à un serveur DNS (« quelle est
> l'adresse IP de google.com ? ») et reçoit une réponse. Ce chapitre décortique
> le format exact de ces questions et de ces réponses ; c'est le plus technique
> du guide, ne vous découragez pas.

Nous commencerons par étudier le protocole DNS, puis nous utiliserons ces
connaissances pour programmer un client simple.

Par convention, les paquets DNS sont transportés par UDP et limités à 512
octets. Comme nous le verrons plus tard, ces deux règles ont des exceptions : le
DNS peut aussi fonctionner sur TCP, et un mécanisme appelé eDNS permet
d'augmenter la taille des paquets. Pour l'instant, nous nous en tiendrons
toutefois à la spécification d'origine.

> **Note du traducteur — Paquet, octet, UDP, TCP**
>
> - Un **paquet** est un message envoyé d'une machine à une autre sur le
>   réseau : une suite de valeurs numériques.
> - Un **octet** (*byte* en anglais) est l'unité de base de ces messages : un
>   nombre compris entre 0 et 255, codé sur 8 **bits** (8 chiffres binaires,
>   valant 0 ou 1). 512 octets, c'est très peu : environ une demi-page de texte.
> - **UDP** et **TCP** sont deux façons d'acheminer ces paquets. UDP fonctionne
>   comme une carte postale : on l'envoie sans connexion préalable et sans
>   garantie de réception. TCP ressemble plutôt à un appel téléphonique : on
>   établit une connexion, puis l'échange est fiable et ordonné. Le DNS utilise
>   UDP par défaut, car c'est plus rapide pour un échange court
>   question/réponse.

Le DNS a l'avantage d'utiliser le même format pour les requêtes et pour les
réponses. Cela signifie qu'une fois écrits un module qui lit les paquets (un
*parser*, ou analyseur) et un module qui les écrit, notre travail sur le
protocole est terminé. C'est différent de la plupart des protocoles Internet,
qui utilisent généralement des structures distinctes pour la requête et pour la
réponse. Vu de haut, un paquet DNS se présente ainsi :

| Section                                       | Taille    | Type                    | Rôle                                                                                                                     |
| --------------------------------------------- | --------- | ----------------------- | ------------------------------------------------------------------------------------------------------------------------ |
| Header (en-tête)                              | 12 octets | En-tête                 | Informations sur la requête/réponse.                                                                                     |
| Question Section (section Question)           | Variable  | Liste de questions      | En pratique, une seule question, qui indique le nom demandé (le domaine) et le type d'enregistrement recherché.          |
| Answer Section (section Réponse)              | Variable  | Liste d'enregistrements | Les enregistrements correspondant au type demandé.                                                                       |
| Authority Section (section Autorité)          | Variable  | Liste d'enregistrements | Une liste de serveurs de noms (enregistrements NS), utilisée pour résoudre les requêtes de manière récursive.            |
| Additional Section (section Additionnelle)    | Variable  | Liste d'enregistrements | Des enregistrements supplémentaires, potentiellement utiles. Par exemple, les enregistrements A correspondant aux NS.    |

> **Note du traducteur — « Enregistrement »**
>
> Un **enregistrement** (*record*, ou *resource record*) est une ligne de
> l'annuaire DNS : « tel nom → telle information ». Il en existe plusieurs
> types : le type **A** associe un nom à une adresse IP, le type **NS** indique
> quel serveur gère un domaine, etc. Le chapitre 3 en présente plusieurs.

Concrètement, nous devons gérer trois objets différents : l'en-tête (*Header*),
la question (*Question*) et l'enregistrement (*Record*). Fort heureusement, les
listes d'enregistrements et de questions sont simplement des éléments
individuels placés les uns à la suite des autres, sans rien de plus. Le nombre
d'enregistrements de chaque section est indiqué dans l'en-tête. La structure de
l'en-tête est la suivante :

| Nom RFC  | Nom descriptif                                  | Longueur | Description                                                                                                                                                                              |
| -------- | ----------------------------------------------- | -------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| ID       | Packet Identifier (identifiant du paquet)       | 16 bits  | Un identifiant aléatoire est attribué aux paquets de requête. Les paquets de réponse doivent reprendre le même identifiant. C'est nécessaire pour distinguer les réponses, car UDP est sans état (il ne garde aucune trace des échanges). |
| QR       | Query Response (requête/réponse)                | 1 bit    | 0 pour une requête, 1 pour une réponse.                                                                                                                                                  |
| OPCODE   | Operation Code (code d'opération)               | 4 bits   | En pratique toujours 0 ; voir la RFC 1035 pour le détail.                                                                                                                                |
| AA       | Authoritative Answer (réponse faisant autorité) | 1 bit    | Vaut 1 si le serveur qui répond fait autorité pour le domaine demandé, c'est-à-dire qu'il en est « propriétaire ».                                                                       |
| TC       | Truncated Message (message tronqué)             | 1 bit    | Vaut 1 si la taille du message dépasse 512 octets. Indique traditionnellement que la requête peut être renvoyée en TCP, où cette limite ne s'applique pas.                               |
| RD       | Recursion Desired (récursion souhaitée)         | 1 bit    | Positionné par l'émetteur de la requête s'il souhaite que le serveur tente une résolution récursive lorsqu'il ne dispose pas directement de la réponse.                                  |
| RA       | Recursion Available (récursion disponible)      | 1 bit    | Positionné par le serveur pour indiquer s'il accepte ou non les requêtes récursives.                                                                                                     |
| Z        | Reserved (réservé)                              | 3 bits   | Réservé à l'origine pour un usage futur, désormais utilisé pour les requêtes DNSSEC.                                                                                                     |
| RCODE    | Response Code (code de réponse)                 | 4 bits   | Positionné par le serveur pour indiquer le statut de la réponse : succès ou échec, et dans ce dernier cas, la cause de l'échec.                                                           |
| QDCOUNT  | Question Count (nombre de questions)            | 16 bits  | Nombre d'entrées dans la section Question.                                                                                                                                               |
| ANCOUNT  | Answer Count (nombre de réponses)               | 16 bits  | Nombre d'entrées dans la section Réponse.                                                                                                                                                |
| NSCOUNT  | Authority Count (nombre d'autorités)            | 16 bits  | Nombre d'entrées dans la section Autorité.                                                                                                                                               |
| ARCOUNT  | Additional Count (nombre d'additionnels)        | 16 bits  | Nombre d'entrées dans la section Additionnelle.                                                                                                                                          |

> **Note du traducteur — Lire ce tableau**
>
> - Une **RFC** (*Request For Comments*) est un document de normalisation
>   d'Internet, l'équivalent d'une norme dans le bâtiment. La RFC 1035 (1987)
>   est le texte de référence du DNS.
> - Les champs d'1 bit sont des **drapeaux** (*flags*) : de simples cases à
>   cocher, 0 = non, 1 = oui.
> - L'en-tête fait 12 octets, soit 96 bits. Le total de la colonne
>   « Longueur » donne bien 16 + 1 + 4 + 1 + 1 + 1 + 1 + 3 + 4 + 4 × 16 = 96 :
>   tout est rangé bit à bit, sans aucun espace perdu.
> - **Récursion** : un serveur qui accepte la récursion se charge d'aller
>   chercher lui-même la réponse auprès d'autres serveurs si nécessaire, au lieu
>   de vous renvoyer vers eux. Le chapitre 5 détaille ce mécanisme.
> - **DNSSEC** est une extension de sécurité du DNS (signature des réponses),
>   qui n'est pas traitée dans ce guide.

La question est nettement moins intimidante :

| Champ          | Type                | Description                                                                         |
| -------------- | ------------------- | ----------------------------------------------------------------------------------- |
| Name (nom)     | Séquence de labels  | Le nom de domaine, encodé sous forme de séquence de labels, comme décrit plus bas.  |
| Type           | Entier sur 2 octets | Le type d'enregistrement.                                                           |
| Class (classe) | Entier sur 2 octets | La classe ; en pratique toujours égale à 1.                                         |

La difficulté réside dans l'encodage du nom de domaine, sur lequel nous
reviendrons plus loin.

Viennent enfin les enregistrements, qui constituent le cœur du protocole. Il
existe de nombreux types d'enregistrements, mais pour l'instant nous n'en
étudierons que quelques-uns, essentiels. Tous les enregistrements commencent par
le préambule suivant :

| Champ          | Type                | Description                                                                                                       |
| -------------- | ------------------- | ----------------------------------------------------------------------------------------------------------------- |
| Name (nom)     | Séquence de labels  | Le nom de domaine, encodé sous forme de séquence de labels, comme décrit plus bas.                                |
| Type           | Entier sur 2 octets | Le type d'enregistrement.                                                                                         |
| Class (classe) | Entier sur 2 octets | La classe ; en pratique toujours égale à 1.                                                                       |
| TTL            | Entier sur 4 octets | *Time-To-Live* (durée de vie) : durée pendant laquelle un enregistrement peut être gardé en cache avant d'être redemandé. |
| Len            | Entier sur 2 octets | Longueur des données propres au type d'enregistrement.                                                            |

> **Note du traducteur — Cache et TTL**
>
> Un **cache** est une mémoire temporaire : plutôt que de reposer la même
> question à chaque fois, un serveur garde la réponse sous le coude pendant une
> durée limitée, le **TTL**, exprimé en secondes. Un TTL de 300 signifie
> « cette information reste valable 5 minutes ».

Nous voilà prêts à examiner des types d'enregistrement précis, en commençant par
le plus essentiel : l'enregistrement A, qui associe un nom à une adresse IP.

| Champ                 | Type                         | Description                                                            |
| --------------------- | ---------------------------- | ---------------------------------------------------------------------- |
| Preamble (préambule)  | Préambule d'enregistrement   | Le préambule décrit ci-dessus, avec le champ de longueur égal à 4.     |
| IP                    | Entier sur 4 octets          | Une adresse IP encodée sous forme d'entier sur quatre octets.          |

> **N.d.T.** — Une adresse IPv4 comme `216.58.211.142` est justement composée
> de quatre nombres de 0 à 255 : chacun tient dans un octet, d'où les 4 octets.

Arrivés à ce stade, voyons concrètement comment cela fonctionne en effectuant
une recherche avec l'outil `dig` :

> **Note du traducteur — Les blocs de commandes**
>
> Dans les blocs ci-dessous, la ligne qui commence par `#` est la commande
> tapée dans un terminal (le `#` symbolise l'invite de commande, il ne faut pas
> le recopier) ; les lignes suivantes sont ce que la commande affiche. `dig` est
> un outil en ligne de commande qui interroge un serveur DNS et affiche la
> réponse en détail ; les lignes qui commencent par `;` sont des commentaires
> ajoutés par `dig` pour la lisibilité. Ces sorties sont laissées telles
> quelles, en anglais, puisque c'est ce que vous verrez à l'écran.
>
> Les adresses IP, TTL et dates affichées datent de la rédaction originale du
> guide (2016) : vous obtiendrez des valeurs différentes aujourd'hui.

```text
# dig +noedns google.com

; <<>> DiG 9.10.3-P4-Ubuntu <<>> +noedns google.com
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 36383
;; flags: qr rd ra ad; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 0

;; QUESTION SECTION:
;google.com.                    IN      A

;; ANSWER SECTION:
google.com.             204     IN      A       172.217.18.142

;; Query time: 0 msec
;; SERVER: 192.168.1.1#53(192.168.1.1)
;; WHEN: Wed Jul 06 13:24:19 CEST 2016
;; MSG SIZE  rcvd: 44
```

Nous utilisons l'option `+noedns` pour nous assurer de rester sur le format
d'origine. Plusieurs points sont à remarquer dans la sortie ci-dessus :

 * On voit que `dig` décrit explicitement l'en-tête (`HEADER`), la section
   Question (`QUESTION SECTION`) et la section Réponse (`ANSWER SECTION`) du
   paquet de réponse.
 * L'en-tête utilise l'OPCODE QUERY, qui correspond à 0. Le statut (RESCODE)
   vaut NOERROR, soit 0 numériquement. L'identifiant (id) est 36383 et changera
   aléatoirement d'une requête à l'autre. Les drapeaux Query Response (qr),
   Recursion Desired (rd) et Recursion Available (ra) sont activés,
   c'est-à-dire égaux à 1. Nous pouvons ignorer `ad` pour l'instant, car il
   concerne DNSSEC. Enfin, l'en-tête nous indique qu'il y a une question et un
   enregistrement de réponse.
 * La section Question nous montre notre question : `IN` indique la classe, et
   A nous dit que nous demandons des enregistrements de type A.
 * La section Réponse contient l'enregistrement de réponse, avec l'adresse IP de
   Google. `204` est le TTL, IN est à nouveau la classe et A le type
   d'enregistrement. On trouve enfin l'adresse IP de google.com.
 * La dernière ligne nous apprend que la taille totale du paquet était de 44
   octets.

> **N.d.T.** — `IN` signifie *Internet* : c'est la classe n° 1, la seule utilisée
> en pratique. Le « RESCODE » mentionné par l'auteur désigne le champ RCODE de
> l'en-tête (le code de réponse).

Certains détails restent toutefois cachés ; plongeons donc plus profondément et
examinons un vidage hexadécimal (*hexdump*) des paquets. Nous pouvons utiliser
`netcat` pour écouter sur un port, puis demander à `dig` d'y envoyer sa
requête. Dans une première fenêtre de terminal, on lance :

> **Note du traducteur — Port, netcat, hexdump**
>
> - Un **port** est un numéro de « guichet » sur une machine : une même machine
>   peut héberger plusieurs services, chacun à l'écoute sur son propre port. Le
>   DNS utilise le port 53.
> - `netcat` (commande `nc`) permet d'envoyer ou de recevoir des données brutes
>   sur le réseau. Ici, on l'utilise comme un faux serveur qui enregistre dans
>   un fichier tout ce qu'il reçoit.
> - `hexdump` affiche le contenu brut d'un fichier, octet par octet, en
>   hexadécimal (voir plus bas).
> - Ces outils sont disponibles nativement sous Linux et macOS. Sous Windows, le
>   plus simple est d'utiliser WSL (le sous-système Linux de Windows). La
>   syntaxe exacte de `nc` peut varier légèrement selon la version installée.

```text
# nc -u -l 1053 > query_packet.txt
```

Puis, dans une autre fenêtre :

```text
# dig +retry=0 -p 1053 @127.0.0.1 +noedns google.com

; <<>> DiG 9.10.3-P4-Ubuntu <<>> +retry=0 -p 1053 @127.0.0.1 +noedns google.com
; (1 server found)
;; global options: +cmd
;; connection timed out; no servers could be reached
```

L'échec est normal dans ce cas : `dig` abandonne au bout d'un certain temps
(*timeout*) faute de recevoir une réponse, puis se termine. On peut alors
quitter `netcat` avec Ctrl+C. Il nous reste un paquet de requête dans
`query_packet.txt`. Nous pouvons nous servir de ce paquet pour enregistrer
également un paquet de réponse :

```text
# nc -u 8.8.8.8 53 < query_packet.txt > response_packet.txt
```

> **N.d.T.** — Cette commande envoie le contenu de `query_packet.txt` au serveur
> DNS public de Google (`8.8.8.8`, port 53) et enregistre sa réponse dans
> `response_packet.txt`.

Patientez une seconde, puis interrompez avec Ctrl+C. Nous sommes maintenant
prêts à examiner nos paquets :

```text
# hexdump -C query_packet.txt
00000000  86 2a 01 20 00 01 00 00  00 00 00 00 06 67 6f 6f  |.*. .........goo|
00000010  67 6c 65 03 63 6f 6d 00  00 01 00 01              |gle.com.....|
0000001c
# hexdump -C response_packet.txt
00000000  86 2a 81 80 00 01 00 01  00 00 00 00 06 67 6f 6f  |.*...........goo|
00000010  67 6c 65 03 63 6f 6d 00  00 01 00 01 c0 0c 00 01  |gle.com.........|
00000020  00 01 00 00 01 25 00 04  d8 3a d3 8e              |.....%...:..|
0000002c
```

> **Note du traducteur — Lire de l'hexadécimal**
>
> L'**hexadécimal** (base 16) utilise 16 chiffres : `0` à `9`, puis `a` (10),
> `b` (11), `c` (12), `d` (13), `e` (14) et `f` (15). Deux chiffres
> hexadécimaux décrivent exactement un octet, de `00` (0) à `ff` (255). Par
> exemple, `2a` = 2 × 16 + 10 = 42. On écrit souvent `0x` devant un nombre pour
> signaler qu'il est en hexadécimal : `0x2a`.
>
> Chaque ligne d'un *hexdump* se lit ainsi :
>
> - la première colonne (`00000000`, `00000010`…) est la position du premier
>   octet de la ligne, elle-même en hexadécimal (`10` = 16) ;
> - viennent ensuite 16 octets, en deux groupes de 8 ;
> - la colonne entre `|…|` montre les mêmes octets interprétés comme du texte
>   (un `.` quand l'octet ne correspond à aucun caractère imprimable). C'est
>   pour cela qu'on y voit apparaître `google` et `com`.

Essayons d'y voir clair. Nous savons que l'en-tête fait 12 octets. Pour le
paquet de requête, les octets de l'en-tête sont :
`86 2a 01 20 00 01 00 00 00 00 00 00`. On constate que les huit derniers octets
correspondent aux nombres d'entrées des différentes sections ; la seule à avoir
un contenu est la section Question, qui contient une seule entrée. La partie la
plus intéressante, ce sont les quatre premiers octets, qui correspondent aux
différents champs de l'en-tête. Tout d'abord, nous savons que nous avons un
identifiant sur 2 octets, censé rester le même dans la requête et dans la
réponse. On voit effectivement que dans cet exemple, il vaut `86 2a` dans les
deux *hexdumps*. Le plus difficile à décoder, ce sont les deux octets restants.
Pour les interpréter, il faut les convertir en binaire. En commençant par le
`01 20` du paquet de requête, on obtient (bit de poids fort en premier) :

> **Note du traducteur — De l'hexadécimal au binaire**
>
> Chaque chiffre hexadécimal correspond exactement à 4 bits : `0` = `0000`,
> `1` = `0001`, `2` = `0010`, … `8` = `1000`, … `c` = `1100`, `f` = `1111`. Il
> suffit donc de remplacer chaque chiffre par ses 4 bits :
> `01 20` → `0000 0001  0010 0000`. Le **bit de poids fort** est le bit le plus
> à gauche, celui qui « pèse » le plus dans la valeur du nombre (comme le chiffre
> des milliers dans un nombre décimal).
>
> Dans le schéma ci-dessous, chaque colonne est un bit ; les lettres écrites
> verticalement sous les tirets donnent le nom du champ concerné (QR, OPCODE,
> AA, TC, RD, RA, Z, RCODE).

```text
0 0 0 0 0 0 0 1  0 0 1 0 0 0 0 0
- -+-+-+- - - -  - -+-+- -+-+-+-
Q    O    A T R  R   Z      R
R    P    A C D  A          C
     C                      O
     O                      D
     D                      E
     E
```

Mis à part le bit lié à DNSSEC dans la zone `Z`, c'est conforme aux attentes.
`QR` vaut 0 puisqu'il s'agit d'une requête ; `OPCODE` vaut aussi 0, puisqu'il
s'agit d'une recherche standard ; les drapeaux `AA`, `TC` et `RA` ne concernent
pas les requêtes, tandis que `RD` est activé, car `dig` demande par défaut une
résolution récursive. Enfin, `RCODE` n'est pas non plus utilisé dans les
requêtes.

Passons aux octets de drapeaux du paquet de réponse, `81 80` :

```text
1 0 0 0 0 0 0 1  1 0 0 0 0 0 0 0
- -+-+-+- - - -  - -+-+- -+-+-+-
Q    O    A T R  R   Z      R
R    P    A C D  A          C
     C                      O
     O                      D
     D                      E
     E
```

Comme il s'agit d'une réponse, `QR` est activé, de même que `RA`, pour indiquer
que le serveur prend bien en charge la récursion. En examinant les huit octets
restants de la réponse, on constate qu'en plus d'une question, nous avons aussi
un enregistrement de réponse.

Juste après l'en-tête vient la question. Décortiquons-la octet par octet :

```text
                   nom demandé              type   classe
       -----------------------------------  -----  -----
HEX    06 67 6f 6f 67 6c 65 03 63 6f 6d 00  00 01  00 01
ASCII     g  o  o  g  l  e     c  o  m
DEC    6                    3           0       1      1
```

> **N.d.T.** — Ligne `HEX` : les octets en hexadécimal. Ligne `ASCII` : le
> caractère correspondant à chaque octet (l'ASCII est la table standard de
> correspondance entre nombres et caractères : `67` = `g`, `6f` = `o`…). Ligne
> `DEC` : la valeur décimale des octets qui représentent des nombres.

Comme l'indiquait le tableau plus haut, la question se compose de trois
parties : le nom demandé, le type et la classe. La façon dont le nom est encodé
présente toutefois une particularité : il ne contient aucun point. Le DNS encode
en effet chaque nom sous la forme d'une séquence de `labels`, chaque label étant
précédé d'un octet unique qui indique sa longueur. Dans l'exemple ci-dessus,
« google » fait 6 octets et est donc précédé de `0x06`, tandis que « com » fait
3 octets et est précédé de `0x03`. Enfin, tous les noms se terminent par un
label de longueur nulle, c'est-à-dire un octet nul. Cela paraît assez simple,
non ? Eh bien, comme nous allons le voir bientôt, il y a une subtilité
supplémentaire.

> **N.d.T.** — Un **label** est un morceau du nom de domaine compris entre deux
> points : `www.google.com` contient les trois labels `www`, `google` et `com`.
> Le nom est donc stocké sous la forme `[3]www[6]google[3]com[0]`, chaque
> nombre entre crochets étant un octet de longueur.

Nous avons atteint la fin de notre paquet de requête, mais il reste des données
à décoder dans le paquet de réponse. Ces données sont un unique enregistrement A
contenant l'adresse IP de google.com :

```text
      nom      type   classe        ttl        long.    ip
      ------  ------  ------  --------------  ------  --------------
HEX   c0  0c  00  01  00  01  00  00  01  25  00  04  d8  3a  d3  8e
DEC   192 12    1       1           293         4     216 58  211 142
```

L'essentiel est conforme aux attentes : le type vaut 1 pour un `enregistrement
A`, la classe vaut 1 pour `IN`, le TTL vaut ici 293, ce qui semble raisonnable,
la longueur des données vaut 4, comme il se doit, et l'on apprend enfin que
l'adresse IP de Google est `216.58.211.142`. Mais alors, que se passe-t-il avec
le champ du nom ? Où sont passés les labels que nous venons de découvrir ?

> **N.d.T.** — Vérification : `01 25` en hexadécimal = 1 × 256 + 37 = 293 ;
> `d8` = 216, `3a` = 58, `d3` = 211, `8e` = 142.

En raison de la contrainte de taille d'origine du DNS (512 octets par paquet),
une forme de compression était nécessaire. Comme l'essentiel de la place est
occupé par les noms de domaine, et que des morceaux d'un même nom ont tendance à
se répéter, il y a là une occasion évidente de gagner de la place. Prenons par
exemple la requête DNS suivante :

```text
# dig @a.root-servers.net com

- (coupé) -

;; AUTHORITY SECTION:
com.                172800  IN  NS      e.gtld-servers.net.
com.                172800  IN  NS      b.gtld-servers.net.
com.                172800  IN  NS      j.gtld-servers.net.
com.                172800  IN  NS      m.gtld-servers.net.
com.                172800  IN  NS      i.gtld-servers.net.
com.                172800  IN  NS      f.gtld-servers.net.
com.                172800  IN  NS      a.gtld-servers.net.
com.                172800  IN  NS      g.gtld-servers.net.
com.                172800  IN  NS      h.gtld-servers.net.
com.                172800  IN  NS      l.gtld-servers.net.
com.                172800  IN  NS      k.gtld-servers.net.
com.                172800  IN  NS      c.gtld-servers.net.
com.                172800  IN  NS      d.gtld-servers.net.

;; ADDITIONAL SECTION:
e.gtld-servers.net. 172800  IN  A       192.12.94.30
b.gtld-servers.net. 172800  IN  A       192.33.14.30
b.gtld-servers.net. 172800  IN  AAAA    2001:503:231d::2:30
j.gtld-servers.net. 172800  IN  A       192.48.79.30
m.gtld-servers.net. 172800  IN  A       192.55.83.30
i.gtld-servers.net. 172800  IN  A       192.43.172.30
f.gtld-servers.net. 172800  IN  A       192.35.51.30
a.gtld-servers.net. 172800  IN  A       192.5.6.30
a.gtld-servers.net. 172800  IN  AAAA    2001:503:a83e::2:30
g.gtld-servers.net. 172800  IN  A       192.42.93.30
h.gtld-servers.net. 172800  IN  A       192.54.112.30
l.gtld-servers.net. 172800  IN  A       192.41.162.30
k.gtld-servers.net. 172800  IN  A       192.52.178.30
c.gtld-servers.net. 172800  IN  A       192.26.92.30
d.gtld-servers.net. 172800  IN  A       192.31.80.30

- (coupé) -
```

Ici, nous interrogeons l'un des serveurs racine d'Internet pour connaître les
serveurs de noms qui gèrent le TLD .com. Remarquez comme `gtld-servers.net.`
revient sans cesse : ne serait-il pas pratique de ne l'inclure qu'une seule
fois ? Une façon d'y parvenir est d'insérer une « instruction de saut », qui
indique à l'analyseur de paquets d'aller à une autre position et d'y terminer la
lecture du nom. Et c'est justement ce que nous avons sous les yeux dans notre
paquet de réponse.

> **Note du traducteur — Serveurs racine, TLD et instruction de saut**
>
> Le DNS est organisé comme une arborescence. Tout en haut, les **serveurs
> racine** (*root servers*) connaissent les serveurs chargés de chaque **TLD**
> (*Top-Level Domain*, domaine de premier niveau : `.com`, `.fr`, `.org`…).
> Ceux-ci connaissent à leur tour les serveurs de chaque domaine
> (`google.com`), et ainsi de suite. Le chapitre 5 parcourt cette arborescence
> pas à pas.
>
> L'« instruction de saut » fonctionne comme un renvoi dans un document (« voir
> page 12 ») : au lieu de répéter un texte déjà écrit, on indique où le relire.

J'ai mentionné plus haut que chaque label est précédé d'un octet de longueur. Il
faut en plus tenir compte du fait suivant : si les deux bits de poids fort de
cet octet de longueur sont à 1, il faut s'attendre à ce qu'il soit suivi d'un
second octet. Ces deux octets pris ensemble, une fois leurs deux bits de poids
fort retirés, indiquent la position du saut. Dans l'exemple ci-dessus, nous
avons `0xC00C`. Le motif binaire des deux bits de poids fort, exprimé en
hexadécimal, est `0xC000` (en binaire `11000000 00000000`) ; on peut donc
trouver la position du saut en appliquant un « ou exclusif » (*XOR*) entre nos
deux octets et ce masque, pour remettre ces deux bits à zéro :
`0xC00C ^ 0xC000 = 12`. Il faut donc sauter à l'octet 12 du paquet et lire à
partir de là. En se rappelant que l'en-tête DNS fait justement 12 octets, on
comprend que l'instruction nous demande de lire à partir du début de la partie
Question du paquet. C'est logique, puisque la question commence par le domaine
demandé, ici « google.com ». Une fois la lecture du nom terminée, on reprend
l'analyse là où on s'était arrêté, et on passe au type d'enregistrement.

> **Note du traducteur — Le « ou exclusif » (XOR, noté `^`)**
>
> Le XOR compare deux nombres bit à bit : le résultat vaut 1 quand les deux bits
> sont différents, 0 quand ils sont identiques. Avec un masque dont les seuls
> bits à 1 sont ceux qu'on veut effacer (et dont on sait qu'ils sont à 1), on
> les remet à 0 sans toucher au reste :
>
> ```text
> 0xC00C = 1100 0000 0000 1100
> 0xC000 = 1100 0000 0000 0000
> XOR    = 0000 0000 0000 1100  = 12
> ```

### BytePacketBuffer

Nous en savons enfin assez pour commencer à programmer ! La première chose à
faire est de nous doter d'un moyen pratique de manipuler les paquets. Pour cela,
nous utiliserons une `struct` (structure) appelée `BytePacketBuffer`.

> **Note du traducteur — Lire le code Rust**
>
> À partir d'ici, le guide contient beaucoup de code écrit en **Rust**, un
> langage de programmation. Les commentaires du code (lignes qui commencent par
> `//` ou `///`) ont été traduits ; les noms de variables et de fonctions sont
> restés en anglais, car ils font partie du programme. Quelques repères :
>
> - `struct` définit une **structure** : une fiche comportant des champs nommés
>   (ici `buf`, le contenu du paquet, et `pos`, la position de lecture).
> - `impl` regroupe les **fonctions** (`fn`) qui agissent sur cette structure,
>   appelées aussi *méthodes*. `self` désigne la structure elle-même.
> - `u8`, `u16` et `u32` désignent des nombres entiers positifs sur 8, 16 ou 32
>   bits (1, 2 ou 4 octets). `usize` est un entier qui sert à compter ou à
>   repérer une position. `[u8; 512]` est un tableau de 512 octets.
> - `Result<...>` signifie « soit un résultat, soit une erreur ». `Ok(...)`
>   renvoie le résultat, `Err(...)` renvoie une erreur, et le point
>   d'interrogation `?` placé après un appel veut dire : « si cet appel a
>   échoué, on s'arrête là et on transmet l'erreur ».
> - `<<` et `>>` décalent les bits vers la gauche ou vers la droite ; `|` (ou)
>   et `&` (et) combinent des bits. Lire les deux octets `0x86` et `0x2a` pour
>   en faire le nombre `0x862a`, c'est décaler le premier de 8 bits vers la
>   gauche, puis l'assembler au second avec `|` : c'est exactement ce que fait
>   `read_u16`.
> - Le [glossaire](GLOSSAIRE.md) reprend ces notions et les autres termes du
>   guide.

> **Note du traducteur — Lignes à placer en tête du fichier**
>
> Le texte du guide ne montre pas les premières lignes du programme, pourtant
> indispensables : elles figurent seulement dans `examples/sample1.rs`. Si vous
> recopiez le code des chapitres dans votre propre fichier, commencez-le par :
>
> ```rust
> use std::fs::File;
> use std::io::Read;
> use std::net::Ipv4Addr;
>
> type Error = Box<dyn std::error::Error>;
> type Result<T> = std::result::Result<T, Error>;
> ```
>
> - Les trois lignes `use` importent des éléments de la bibliothèque standard
>   de Rust : `File` (ouvrir un fichier), `Read` (lire son contenu) et
>   `Ipv4Addr` (adresse IPv4).
> - Les deux lignes `type` définissent le `Result<...>` utilisé partout dans
>   le guide. Le `Result` standard de Rust attend deux informations : le type
>   du résultat et le type de l'erreur (`Result<T, E>`). Ce raccourci fixe une
>   fois pour toutes le type d'erreur (`Box<dyn std::error::Error>`, c'est-à-dire
>   « n'importe quelle erreur »), ce qui permet d'écrire simplement
>   `Result<u8>`. Sans ces lignes, chaque `Result<...>` provoque l'erreur
>   *enum takes 2 generic arguments but 1 generic argument was supplied*
>   (« l'énumération attend 2 paramètres, 1 seul a été fourni »).
>
> Ces lignes évoluent légèrement d'un chapitre à l'autre ; en cas de doute,
> reportez-vous au début du fichier `examples/sampleN.rs` correspondant.

```rust
pub struct BytePacketBuffer {
    pub buf: [u8; 512],
    pub pos: usize,
}

impl BytePacketBuffer {

    /// Crée un tampon (buffer) vierge destiné à contenir le paquet, ainsi qu'un
    /// champ qui mémorise la position où l'on se trouve.
    pub fn new() -> BytePacketBuffer {
        BytePacketBuffer {
            buf: [0; 512],
            pos: 0,
        }
    }

    /// Position courante dans le tampon
    fn pos(&self) -> usize {
        self.pos
    }

    /// Avance la position dans le tampon d'un nombre de pas donné
    fn step(&mut self, steps: usize) -> Result<()> {
        self.pos += steps;

        Ok(())
    }

    /// Modifie la position dans le tampon
    fn seek(&mut self, pos: usize) -> Result<()> {
        self.pos = pos;

        Ok(())
    }

    /// Lit un seul octet et avance la position d'un pas
    fn read(&mut self) -> Result<u8> {
        if self.pos >= 512 {
            return Err("End of buffer".into());
        }
        let res = self.buf[self.pos];
        self.pos += 1;

        Ok(res)
    }

    /// Récupère un seul octet, sans modifier la position dans le tampon
    fn get(&mut self, pos: usize) -> Result<u8> {
        if pos >= 512 {
            return Err("End of buffer".into());
        }
        Ok(self.buf[pos])
    }

    /// Récupère une plage d'octets
    fn get_range(&mut self, start: usize, len: usize) -> Result<&[u8]> {
        if start + len >= 512 {
            return Err("End of buffer".into());
        }
        Ok(&self.buf[start..start + len as usize])
    }

    /// Lit deux octets et avance de deux pas
    fn read_u16(&mut self) -> Result<u16> {
        let res = ((self.read()? as u16) << 8) | (self.read()? as u16);

        Ok(res)
    }

    /// Lit quatre octets et avance de quatre pas
    fn read_u32(&mut self) -> Result<u32> {
        let res = ((self.read()? as u32) << 24)
            | ((self.read()? as u32) << 16)
            | ((self.read()? as u32) << 8)
            | ((self.read()? as u32) << 0);

        Ok(res)
    }


    /// Lit un qname (le nom de domaine contenu dans le paquet)
    ///
    /// La partie délicate : lire les noms de domaine en tenant compte des
    /// labels. Transforme par exemple [3]www[6]google[3]com[0] en
    /// www.google.com, qui est ajouté à la fin de outstr.
    fn read_qname(&mut self, outstr: &mut String) -> Result<()> {
        // Comme nous pouvons rencontrer des sauts, nous suivons notre position
        // dans une variable locale plutôt que d'utiliser celle de la structure.
        // Cela permet de placer la position partagée juste après le qname en
        // cours, tout en continuant à suivre notre progression dans ce qname
        // grâce à cette variable.
        let mut pos = self.pos();

        // indique si nous avons effectué un saut ou non
        let mut jumped = false;
        let max_jumps = 5;
        let mut jumps_performed = 0;

        // Le séparateur ajouté avant chaque label. Comme on ne veut pas de
        // point au début du nom de domaine, on le laisse vide pour l'instant et
        // on le passera à "." à la fin du premier tour de boucle.
        let mut delim = "";
        loop {
            // Les paquets DNS sont des données non fiables : il faut donc être
            // paranoïaque. Quelqu'un pourrait fabriquer un paquet dont les
            // instructions de saut forment une boucle sans fin. Ce test protège
            // contre ce type de paquet.
            if jumps_performed > max_jumps {
                return Err(format!("Limit of {} jumps exceeded", max_jumps).into());
            }

            // À ce stade, on se trouve toujours au début d'un label. Pour
            // rappel, un label commence par un octet qui indique sa longueur.
            let len = self.get(pos)?;

            // Si les deux bits de poids fort de len sont à 1, il s'agit d'un
            // saut vers une autre position (offset) du paquet :
            if (len & 0xC0) == 0xC0 {
                // On place la position du tampon juste après le label en cours.
                // Nous n'aurons plus besoin d'y toucher ensuite.
                if !jumped {
                    self.seek(pos + 2)?;
                }

                // On lit l'octet suivant, on calcule la position cible (offset)
                // et on effectue le saut en mettant à jour notre variable de
                // position locale
                let b2 = self.get(pos + 1)? as u16;
                let offset = (((len as u16) ^ 0xC0) << 8) | b2;
                pos = offset as usize;

                // On note qu'un saut a été effectué.
                jumped = true;
                jumps_performed += 1;

                continue;
            }
            // Le cas de base : on lit un seul label et on l'ajoute au
            // résultat :
            else {
                // On avance d'un octet pour dépasser l'octet de longueur.
                pos += 1;

                // Les noms de domaine se terminent par un label vide de
                // longueur 0 : si la longueur vaut zéro, on a terminé.
                if len == 0 {
                    break;
                }

                // On ajoute d'abord le séparateur au résultat.
                outstr.push_str(delim);

                // On extrait les octets ASCII (les caractères) de ce label et
                // on les ajoute au résultat.
                let str_buffer = self.get_range(pos, len as usize)?;
                outstr.push_str(&String::from_utf8_lossy(str_buffer).to_lowercase());

                delim = ".";

                // On avance de toute la longueur du label.
                pos += len as usize;
            }
        }

        if !jumped {
            self.seek(pos)?;
        }

        Ok(())
    }
}
```

> **N.d.T.** — Le « tampon » (*buffer*) est simplement une zone mémoire de 512
> octets qui contient le paquet, avec un curseur (`pos`) qui avance au fil de la
> lecture, comme un doigt qui suit les lignes d'un texte. La fonction
> `read_qname` est la plus délicate : elle reconstitue un nom de domaine à
> partir de ses labels, en suivant les éventuels sauts décrits plus haut, et se
> protège contre les paquets piégés dont les sauts boucleraient à l'infini.

### ResultCode

Avant de passer à l'en-tête, nous ajoutons une `enum` (énumération) pour les
valeurs du champ `rescode` :

```rust
#[derive(Copy, Clone, Debug, PartialEq, Eq)]
pub enum ResultCode {
    NOERROR = 0,
    FORMERR = 1,
    SERVFAIL = 2,
    NXDOMAIN = 3,
    NOTIMP = 4,
    REFUSED = 5,
}

impl ResultCode {
    pub fn from_num(num: u8) -> ResultCode {
        match num {
            1 => ResultCode::FORMERR,
            2 => ResultCode::SERVFAIL,
            3 => ResultCode::NXDOMAIN,
            4 => ResultCode::NOTIMP,
            5 => ResultCode::REFUSED,
            0 | _ => ResultCode::NOERROR,
        }
    }
}
```

> **N.d.T.** — Une `enum` (énumération) est une liste fermée de valeurs
> possibles, comme une liste déroulante. Ici, ce sont les codes de réponse du
> DNS : `NOERROR` (pas d'erreur), `FORMERR` (requête mal formée), `SERVFAIL`
> (échec du serveur), `NXDOMAIN` (le domaine n'existe pas), `NOTIMP` (opération
> non prise en charge) et `REFUSED` (requête refusée). La fonction `from_num`
> convertit le nombre lu dans le paquet en l'une de ces valeurs, à l'aide d'un
> `match`, c'est-à-dire d'un aiguillage (« si c'est 1, alors… ; si c'est 2,
> alors… »). La ligne `#[derive(...)]` demande au compilateur de générer
> automatiquement quelques fonctions standard (copie, comparaison, affichage).

### DnsHeader

Nous pouvons maintenant nous attaquer à l'en-tête. Nous le représenterons
ainsi :

```rust
#[derive(Clone, Debug)]
pub struct DnsHeader {
    pub id: u16, // 16 bits

    pub recursion_desired: bool,    // 1 bit
    pub truncated_message: bool,    // 1 bit
    pub authoritative_answer: bool, // 1 bit
    pub opcode: u8,                 // 4 bits
    pub response: bool,             // 1 bit

    pub rescode: ResultCode,       // 4 bits
    pub checking_disabled: bool,   // 1 bit
    pub authed_data: bool,         // 1 bit
    pub z: bool,                   // 1 bit
    pub recursion_available: bool, // 1 bit

    pub questions: u16,             // 16 bits
    pub answers: u16,               // 16 bits
    pub authoritative_entries: u16, // 16 bits
    pub resource_entries: u16,      // 16 bits
}
```

> **N.d.T.** — `bool` (booléen) est une valeur vrai/faux (`true`/`false`),
> idéale pour représenter un drapeau d'un bit. Les commentaires `// 1 bit`,
> `// 4 bits`… rappellent la place occupée par chaque champ dans le paquet. Les
> noms anglais correspondent aux champs du tableau vu plus haut :
> `recursion_desired` = RD, `truncated_message` = TC, `authoritative_answer` =
> AA, `opcode` = OPCODE, `response` = QR, `rescode` = RCODE,
> `recursion_available` = RA ; `checking_disabled`, `authed_data` et `z` se
> partagent les 3 bits de la zone Z (DNSSEC). Les quatre compteurs de la fin
> correspondent à QDCOUNT, ANCOUNT, NSCOUNT et ARCOUNT.

L'implémentation comporte beaucoup de manipulations de bits :

```rust
impl DnsHeader {
    pub fn new() -> DnsHeader {
        DnsHeader {
            id: 0,

            recursion_desired: false,
            truncated_message: false,
            authoritative_answer: false,
            opcode: 0,
            response: false,

            rescode: ResultCode::NOERROR,
            checking_disabled: false,
            authed_data: false,
            z: false,
            recursion_available: false,

            questions: 0,
            answers: 0,
            authoritative_entries: 0,
            resource_entries: 0,
        }
    }

    pub fn read(&mut self, buffer: &mut BytePacketBuffer) -> Result<()> {
        self.id = buffer.read_u16()?;

        let flags = buffer.read_u16()?;
        let a = (flags >> 8) as u8;
        let b = (flags & 0xFF) as u8;
        self.recursion_desired = (a & (1 << 0)) > 0;
        self.truncated_message = (a & (1 << 1)) > 0;
        self.authoritative_answer = (a & (1 << 2)) > 0;
        self.opcode = (a >> 3) & 0x0F;
        self.response = (a & (1 << 7)) > 0;

        self.rescode = ResultCode::from_num(b & 0x0F);
        self.checking_disabled = (b & (1 << 4)) > 0;
        self.authed_data = (b & (1 << 5)) > 0;
        self.z = (b & (1 << 6)) > 0;
        self.recursion_available = (b & (1 << 7)) > 0;

        self.questions = buffer.read_u16()?;
        self.answers = buffer.read_u16()?;
        self.authoritative_entries = buffer.read_u16()?;
        self.resource_entries = buffer.read_u16()?;

        Ok(())
    }
}
```

> **Note du traducteur — Ce que fait `read`**
>
> La fonction lit les 2 octets de drapeaux d'un seul coup (`flags`), puis les
> sépare en deux octets : `a` (le premier) et `b` (le second). Pour chaque
> drapeau, l'expression `(a & (1 << n)) > 0` teste si le bit n° *n* vaut 1 :
> `1 << n` fabrique un nombre dont seul le bit *n* est à 1, et `&` ne conserve
> que ce bit. L'OPCODE, qui fait 4 bits, est extrait par décalage (`>> 3`) puis
> par masquage (`& 0x0F`, qui ne garde que les 4 bits de droite).

### QueryType

Avant de passer à la partie Question du paquet, il nous faut un moyen de
représenter le type d'enregistrement demandé :

```rust
#[derive(PartialEq, Eq, Debug, Clone, Hash, Copy)]
pub enum QueryType {
    UNKNOWN(u16),
    A, // 1
}

impl QueryType {
    pub fn to_num(&self) -> u16 {
        match *self {
            QueryType::UNKNOWN(x) => x,
            QueryType::A => 1,
        }
    }

    pub fn from_num(num: u16) -> QueryType {
        match num {
            1 => QueryType::A,
            _ => QueryType::UNKNOWN(num),
        }
    }
}
```

> **N.d.T.** — `UNKNOWN(u16)` (« inconnu ») permet de conserver le numéro d'un
> type que le programme ne sait pas encore traiter, au lieu de s'arrêter sur une
> erreur. Pour l'instant, seul le type A (numéro 1) est reconnu. `to_num` et
> `from_num` convertissent dans les deux sens entre le type et son numéro.

### DnsQuestion

L'énumération nous permettra d'ajouter facilement d'autres types
d'enregistrement par la suite. Passons maintenant aux entrées de la section
Question :

```rust
#[derive(Debug, Clone, PartialEq, Eq)]
pub struct DnsQuestion {
    pub name: String,
    pub qtype: QueryType,
}

impl DnsQuestion {
    pub fn new(name: String, qtype: QueryType) -> DnsQuestion {
        DnsQuestion {
            name: name,
            qtype: qtype,
        }
    }

    pub fn read(&mut self, buffer: &mut BytePacketBuffer) -> Result<()> {
        buffer.read_qname(&mut self.name)?;
        self.qtype = QueryType::from_num(buffer.read_u16()?); // qtype
        let _ = buffer.read_u16()?; // classe

        Ok(())
    }
}
```

La partie difficile (la lecture des noms de domaine) étant déjà réglée dans
notre structure `BytePacketBuffer`, ce code s'avère plutôt compact.

### DnsRecord

Il nous faut évidemment aussi un moyen de représenter les enregistrements DNS
eux-mêmes, et là encore nous utilisons une énumération pour pouvoir l'étendre
facilement :

```rust
#[derive(Debug, Clone, PartialEq, Eq, Hash, PartialOrd, Ord)]
#[allow(dead_code)]
pub enum DnsRecord {
    UNKNOWN {
        domain: String,
        qtype: u16,
        data_len: u16,
        ttl: u32,
    }, // 0
    A {
        domain: String,
        addr: Ipv4Addr,
        ttl: u32,
    }, // 1
}
```

Comme il existe de nombreux types d'enregistrements, nous ajoutons la
possibilité de garder trace de ceux que nous ne connaissons pas encore.
L'énumération nous permettra aussi d'ajouter facilement de nouveaux types
d'enregistrements plus tard. L'implémentation de `DnsRecord` se présente
ainsi :

```rust
impl DnsRecord {
    pub fn read(buffer: &mut BytePacketBuffer) -> Result<DnsRecord> {
        let mut domain = String::new();
        buffer.read_qname(&mut domain)?;

        let qtype_num = buffer.read_u16()?;
        let qtype = QueryType::from_num(qtype_num);
        let _ = buffer.read_u16()?;
        let ttl = buffer.read_u32()?;
        let data_len = buffer.read_u16()?;

        match qtype {
            QueryType::A => {
                let raw_addr = buffer.read_u32()?;
                let addr = Ipv4Addr::new(
                    ((raw_addr >> 24) & 0xFF) as u8,
                    ((raw_addr >> 16) & 0xFF) as u8,
                    ((raw_addr >> 8) & 0xFF) as u8,
                    ((raw_addr >> 0) & 0xFF) as u8,
                );

                Ok(DnsRecord::A {
                    domain: domain,
                    addr: addr,
                    ttl: ttl,
                })
            }
            QueryType::UNKNOWN(_) => {
                buffer.step(data_len as usize)?;

                Ok(DnsRecord::UNKNOWN {
                    domain: domain,
                    qtype: qtype_num,
                    data_len: data_len,
                    ttl: ttl,
                })
            }
        }
    }
}
```

> **N.d.T.** — La fonction lit d'abord le préambule commun (nom, type, classe,
> TTL, longueur des données), puis agit selon le type : pour un enregistrement
> A, elle lit les 4 octets de l'adresse IP et les découpe en quatre nombres ;
> pour un type inconnu, elle saute simplement par-dessus les données
> (`buffer.step(data_len)`), dont elle connaît la longueur grâce au préambule.
> `let _ = ...` signifie qu'on lit une valeur (ici la classe) sans la conserver.

### DnsPacket

Enfin, rassemblons le tout dans une structure appelée `DnsPacket` :

```rust
#[derive(Clone, Debug)]
pub struct DnsPacket {
    pub header: DnsHeader,
    pub questions: Vec<DnsQuestion>,
    pub answers: Vec<DnsRecord>,
    pub authorities: Vec<DnsRecord>,
    pub resources: Vec<DnsRecord>,
}

impl DnsPacket {
    pub fn new() -> DnsPacket {
        DnsPacket {
            header: DnsHeader::new(),
            questions: Vec::new(),
            answers: Vec::new(),
            authorities: Vec::new(),
            resources: Vec::new(),
        }
    }

    pub fn from_buffer(buffer: &mut BytePacketBuffer) -> Result<DnsPacket> {
        let mut result = DnsPacket::new();
        result.header.read(buffer)?;

        for _ in 0..result.header.questions {
            let mut question = DnsQuestion::new("".to_string(), QueryType::UNKNOWN(0));
            question.read(buffer)?;
            result.questions.push(question);
        }

        for _ in 0..result.header.answers {
            let rec = DnsRecord::read(buffer)?;
            result.answers.push(rec);
        }
        for _ in 0..result.header.authoritative_entries {
            let rec = DnsRecord::read(buffer)?;
            result.authorities.push(rec);
        }
        for _ in 0..result.header.resource_entries {
            let rec = DnsRecord::read(buffer)?;
            result.resources.push(rec);
        }

        Ok(result)
    }
}
```

> **N.d.T.** — `Vec<...>` est une liste de longueur variable. `from_buffer` lit
> l'en-tête, puis exactement le nombre de questions et d'enregistrements annoncé
> par celui-ci, section par section. Les noms des champs correspondent aux
> sections du paquet : `questions` (Question), `answers` (Réponse),
> `authorities` (Autorité) et `resources` (Additionnelle).

### Assemblage final

Utilisons le fichier `response_packet.txt` généré plus haut pour essayer tout
cela !

```rust
fn main() -> Result<()> {
    let mut f = File::open("response_packet.txt")?;
    let mut buffer = BytePacketBuffer::new();
    f.read(&mut buffer.buf)?;

    let packet = DnsPacket::from_buffer(&mut buffer)?;
    println!("{:#?}", packet.header);

    for q in packet.questions {
        println!("{:#?}", q);
    }
    for rec in packet.answers {
        println!("{:#?}", rec);
    }
    for rec in packet.authorities {
        println!("{:#?}", rec);
    }
    for rec in packet.resources {
        println!("{:#?}", rec);
    }

    Ok(())
}
```

> **N.d.T.** — `main` est la fonction principale, le point d'entrée du
> programme. Elle ouvre le fichier, le copie dans le tampon, le décode, puis
> affiche chaque partie avec `println!`. Le format `{:#?}` produit un affichage
> détaillé et indenté, pratique pour vérifier le contenu d'une structure.

L'exécution affiche :

```text
DnsHeader {
    id: 34346,
    recursion_desired: true,
    truncated_message: false,
    authoritative_answer: false,
    opcode: 0,
    response: true,
    rescode: NOERROR,
    checking_disabled: false,
    authed_data: false,
    z: false,
    recursion_available: true,
    questions: 1,
    answers: 1,
    authoritative_entries: 0,
    resource_entries: 0
}
DnsQuestion {
    name: "google.com",
    qtype: A
}
A {
    domain: "google.com",
    addr: 216.58.211.142,
    ttl: 293
}
```

> **Note du traducteur — Lancer cet exemple**
>
> Le programme complet de ce chapitre se trouve dans `examples/sample1.rs`.
> Après avoir généré `response_packet.txt` comme expliqué plus haut (le fichier
> doit se trouver dans le dossier depuis lequel vous lancez la commande),
> exécutez `cargo run --example sample1` (voir le [README](README.md) pour
> l'installation de Rust). L'identifiant `34346` affiché correspond bien à
> `0x862a`, les deux premiers octets du paquet. Les versions récentes de Rust
> ajoutent une virgule à la fin de chaque ligne de cet affichage ; c'est sans
> incidence.

Au chapitre suivant, nous ajouterons la connexion au réseau :
[Chapitre 2 - Construire un résolveur minimal (*stub resolver*)](chapter2.md)
