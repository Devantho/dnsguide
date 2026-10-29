Glossaire
=========

*Ajout du traducteur : ce glossaire ne fait pas partie du guide original. Il
regroupe les termes rencontrés au fil des chapitres, avec leur équivalent
anglais quand il est utile.*

- [1. Réseau et DNS](#1-réseau-et-dns)
- [2. Représentation des données](#2-représentation-des-données)
- [3. Lire le code Rust](#3-lire-le-code-rust)
- [4. Les identifiants du code](#4-les-identifiants-du-code)
- [5. Les codes de réponse DNS](#5-les-codes-de-réponse-dns)
- [6. Les outils en ligne de commande](#6-les-outils-en-ligne-de-commande)

1. Réseau et DNS
----------------

| Terme (anglais)                                 | Définition                                                                                                                                                                                  |
| ----------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Adresse IP** (*IP address*)                   | Numéro qui identifie une machine sur Internet. En IPv4 : quatre nombres de 0 à 255 (`216.58.211.142`). En IPv6 : huit groupes hexadécimaux (`2001:503:231d::2:30`, les `::` remplaçant une suite de zéros). |
| **127.0.0.1**                                   | Adresse spéciale qui désigne « cette machine-ci » (*localhost*). Utile pour tester un serveur qui tourne sur son propre ordinateur.                                                        |
| **8.8.8.8, 8.8.4.4**                            | Adresses des serveurs DNS publics de Google, utilisables par tous.                                                                                                                          |
| **DNS** (*Domain Name System*)                  | Système de noms de domaine : l'annuaire d'Internet, qui associe des noms (`google.com`) à des adresses IP et à d'autres informations.                                                       |
| **Nom de domaine** (*domain name*)              | Nom hiérarchique, lu de droite à gauche du plus général au plus précis : dans `www.google.com`, `com` est le TLD, `google.com` le domaine, `www` un sous-domaine.                          |
| **Label**                                       | Chacun des morceaux d'un nom de domaine compris entre deux points (`www`, `google`, `com`). 63 caractères au maximum.                                                                     |
| **TLD** (*Top-Level Domain*)                    | Domaine de premier niveau : `.com`, `.fr`, `.org`…                                                                                                                                          |
| **Zone**                                        | Portion de l'annuaire DNS dont un serveur a la charge (par exemple, tout ce qui se termine par `google.com`).                                                                              |
| **Serveur racine** (*root server*)              | Serveur situé au sommet de l'arborescence DNS ; il connaît les serveurs de chaque TLD. Il en existe 13 « logiques » (`a` à `m.root-servers.net`), chacun répliqué dans le monde entier.    |
| **Serveur de noms** (*name server*, *NS*)       | Serveur DNS qui répond aux questions portant sur une zone.                                                                                                                                  |
| **Serveur faisant autorité** (*authoritative*)  | Serveur qui détient les informations officielles d'une zone. Sa réponse fait foi (drapeau `AA`).                                                                                            |
| **Serveur cache** (*caching server*)            | Serveur qui va chercher les réponses auprès des serveurs faisant autorité pour le compte de ses clients, et les garde en mémoire pendant leur TTL. Synonyme : **résolveur récursif**.        |
| **Résolveur** (*resolver*)                      | Programme qui « résout » un nom, c'est-à-dire qui trouve l'information associée (souvent l'adresse IP).                                                                                     |
| **Résolveur minimal** (*stub resolver*)         | Résolveur qui ne fait pas la recherche lui-même et la délègue à un serveur cache. C'est ce qu'utilise votre ordinateur au quotidien (chapitre 2).                                           |
| **Serveur mandataire DNS** (*DNS proxy*)        | Serveur qui se contente de relayer les questions vers un autre serveur et de renvoyer les réponses (chapitre 4).                                                                           |
| **Récursion, résolution récursive**             | Démarche qui consiste à interroger successivement les serveurs racine, puis ceux du TLD, puis ceux du domaine, jusqu'à obtenir la réponse (chapitre 5).                                    |
| **Requête / réponse** (*query / response*)      | La question envoyée à un serveur DNS, et ce qu'il renvoie. En DNS, les deux ont le même format.                                                                                             |
| **Paquet** (*packet*)                           | Message envoyé sur le réseau. Un paquet DNS classique fait au plus 512 octets.                                                                                                              |
| **En-tête** (*header*)                          | Les 12 premiers octets d'un paquet DNS : identifiant, drapeaux et nombre d'éléments de chaque section.                                                                                     |
| **Sections** d'un paquet                        | **Question** (ce qu'on demande), **Réponse** (*Answer*, la réponse directe), **Autorité** (*Authority*, les serveurs compétents vers lesquels on est renvoyé), **Additionnelle** (*Additional*, informations complémentaires, souvent les adresses IP de ces serveurs). |
| **Enregistrement** (*record*, *resource record*) | Une entrée de l'annuaire DNS : un nom, un type, une durée de validité (TTL) et des données.                                                                                                |
| **A**                                           | Enregistrement qui donne l'adresse IPv4 d'un nom (*Address*). Type n° 1.                                                                                                                    |
| **AAAA**                                        | Enregistrement qui donne l'adresse IPv6 d'un nom. Type n° 28.                                                                                                                               |
| **NS** (*Name Server*)                          | Enregistrement qui indique le nom d'un serveur faisant autorité pour un domaine. Type n° 2.                                                                                                 |
| **CNAME** (*Canonical Name*)                    | Alias : indique que ce nom est un autre nom pour un nom « canonique » (officiel). Type n° 5.                                                                                                |
| **MX** (*Mail eXchange*)                        | Enregistrement qui indique le serveur de messagerie d'un domaine, avec une priorité (la plus petite valeur est essayée en premier). Type n° 15.                                             |
| **Classe** (*class*), **IN**                    | Famille de réseau concernée. En pratique, toujours `IN` (*Internet*), de valeur 1.                                                                                                          |
| **TTL** (*Time-To-Live*)                        | Durée de validité d'un enregistrement, en secondes, pendant laquelle il peut être gardé en cache.                                                                                          |
| **Cache**                                       | Mémoire temporaire où l'on garde des réponses déjà obtenues pour ne pas reposer la même question.                                                                                          |
| **RFC** (*Request For Comments*)                | Document de normalisation d'Internet. La RFC 1035 définit le format des paquets DNS.                                                                                                        |
| **UDP**                                         | Mode de transport sans connexion, rapide, sans garantie de livraison (comme une carte postale). Mode par défaut du DNS.                                                                    |
| **TCP**                                         | Mode de transport avec connexion, fiable et ordonné (comme un appel téléphonique).                                                                                                         |
| **Port**                                        | Numéro de « guichet » sur une machine, qui permet à plusieurs services de cohabiter. Le DNS utilise le port 53 ; les exemples du guide utilisent 2053 et 43210.                            |
| **Socket**                                      | Point de branchement d'un programme sur le réseau, ouvert sur un port, par lequel il envoie et reçoit des paquets.                                                                         |
| **DHCP**                                        | Protocole qui configure automatiquement un ordinateur à sa connexion au réseau (adresse IP, serveur DNS…).                                                                                 |
| **FAI** (*ISP, Internet Service Provider*)      | Fournisseur d'accès à Internet.                                                                                                                                                             |
| **eDNS**                                        | Extension du DNS qui permet notamment des paquets de plus de 512 octets. Non traitée dans le guide.                                                                                         |
| **DNSSEC**                                      | Extension de sécurité du DNS : les réponses sont signées, ce qui permet de vérifier leur authenticité. Non traitée dans le guide.                                                          |
| **Empoisonnement DNS** (*DNS poisoning*)        | Attaque qui consiste à faire accepter de fausses réponses à un serveur DNS, pour détourner un nom vers un serveur malveillant.                                                             |
| **Anycast**                                     | Technique qui permet à de nombreuses machines réparties dans le monde de partager une même adresse IP ; utilisée pour les serveurs racine.                                                 |

2. Représentation des données
-----------------------------

| Terme (anglais)                                   | Définition                                                                                                                                                         |
| ------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| **Bit**                                           | Chiffre binaire : 0 ou 1. La plus petite unité d'information.                                                                                                      |
| **Octet** (*byte*)                                | Groupe de 8 bits, qui peut représenter un nombre de 0 à 255. Unité de base des paquets.                                                                            |
| **Binaire**                                       | Écriture des nombres en base 2, avec seulement des 0 et des 1. `0010 0000` = 32.                                                                                   |
| **Hexadécimal**, `0x`                             | Écriture en base 16, avec les chiffres `0`–`9` puis `a`–`f`. Deux chiffres hexadécimaux = un octet (`ff` = 255). Le préfixe `0x` signale un nombre hexadécimal.    |
| **Décimal**                                       | Écriture habituelle, en base 10.                                                                                                                                   |
| **ASCII**                                         | Table standard de correspondance entre nombres et caractères (`0x67` = `g`).                                                                                       |
| **Bit de poids fort** (*Most Significant Bit*, MSB) | Le bit le plus à gauche d'un nombre, celui qui pèse le plus dans sa valeur.                                                                                      |
| **Drapeau** (*flag*)                              | Information oui/non stockée sur un seul bit, comme une case à cocher.                                                                                              |
| **Masque** (*mask*)                               | Nombre dont certains bits sont à 1, utilisé pour isoler ou modifier ces bits dans un autre nombre.                                                                 |
| **ET binaire** `&`, **OU binaire** `\|`           | Opérations bit à bit : `&` ne garde que les bits à 1 dans les deux nombres (sert à isoler des bits) ; `\|` combine les bits à 1 des deux nombres (sert à assembler). |
| **OU exclusif** (*XOR*) `^`                       | Opération bit à bit qui donne 1 quand les deux bits diffèrent. Sert dans le guide à effacer les deux bits de poids fort d'un pointeur de saut.                     |
| **Décalage** (*shift*) `<<`, `>>`                 | Déplace tous les bits d'un nombre vers la gauche ou la droite. `1 << 3` = `1000` en binaire = 8.                                                                   |
| **Entier sur 2 / 4 octets**                       | Nombre stocké sur 16 / 32 bits, octet de poids fort en premier (ordre dit « gros-boutiste » ou *big-endian*, utilisé par les protocoles réseau).                   |
| **Position, décalage** (*offset*)                 | Emplacement d'un octet dans le paquet, compté depuis le début (le premier octet est à la position 0).                                                              |
| **Compression, saut** (*jump*)                    | Mécanisme du DNS qui évite de répéter un nom : deux octets commençant par les bits `11` indiquent la position où lire la suite du nom.                              |
| **Tampon** (*buffer*)                             | Zone mémoire qui contient les octets d'un paquet, avec un curseur de lecture/écriture.                                                                            |
| **Analyseur** (*parser*), **décoder** (*parse*)   | Programme ou fonction qui transforme des octets bruts en données structurées.                                                                                       |
| ***Hexdump***                                     | Affichage du contenu brut d'un fichier, octet par octet, en hexadécimal.                                                                                           |

3. Lire le code Rust
--------------------

| Élément                               | Signification                                                                                                                                                      |
| ------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| **Rust**                              | Langage de programmation réputé pour sa rapidité et sa sûreté.                                                                                                    |
| **cargo**                             | Outil qui compile et lance les programmes Rust (`cargo run --example sample1`).                                                                                    |
| **Compiler**                          | Traduire le code source en programme exécutable. Le compilateur Rust refuse de produire un programme s'il détecte certaines erreurs.                              |
| `// ...`, `/// ...`                   | Commentaires : texte destiné au lecteur, ignoré par l'ordinateur. `///` documente l'élément qui suit.                                                             |
| `fn`                                  | Définit une **fonction** : une suite d'instructions qui reçoit des paramètres et renvoie un résultat (`-> Type`).                                                 |
| `main`                                | Fonction principale, point d'entrée du programme.                                                                                                                  |
| `struct`                              | **Structure** : une fiche avec des champs nommés (`DnsHeader` a les champs `id`, `opcode`…).                                                                      |
| `enum`                                | **Énumération** : une liste fermée de variantes possibles (`QueryType` vaut `A`, `NS`, `CNAME`…).                                                                 |
| `impl`                                | Regroupe les fonctions (**méthodes**) associées à une structure ou une énumération.                                                                               |
| `self`                                | Désigne, dans une méthode, la structure sur laquelle elle agit.                                                                                                    |
| `pub`                                 | Rend un élément accessible depuis le reste du programme (« public »).                                                                                             |
| `let`, `let mut`                      | Déclare une variable ; `mut` (*mutable*) indique qu'elle pourra être modifiée.                                                                                    |
| `u8`, `u16`, `u32`                    | Entiers positifs sur 8, 16 ou 32 bits (1, 2 ou 4 octets).                                                                                                          |
| `usize`                               | Entier positif utilisé pour compter ou repérer une position.                                                                                                       |
| `bool`, `true`, `false`               | Booléen : vrai ou faux.                                                                                                                                            |
| `String`, `&str`                      | Texte (chaîne de caractères).                                                                                                                                      |
| `[u8; 512]`                           | Tableau de 512 octets.                                                                                                                                             |
| `Vec<...>`                            | Liste de longueur variable. `push` ajoute un élément, `pop` retire le dernier, `len()` donne la longueur.                                                         |
| `Ipv4Addr`, `Ipv6Addr`                | Types fournis par Rust pour représenter une adresse IPv4 ou IPv6.                                                                                                 |
| `Result<...>`, `Ok(...)`, `Err(...)`  | Résultat d'une opération qui peut échouer : soit `Ok` avec la valeur, soit `Err` avec l'erreur.                                                                    |
| `?`                                   | Placé après un appel : « si cet appel a échoué, on s'arrête et on transmet l'erreur à l'appelant ».                                                                |
| `Option<...>`, `Some(...)`, `None`    | Valeur facultative : soit `Some` avec une valeur, soit `None` (rien).                                                                                              |
| `match`                               | Aiguillage : exécute un traitement différent selon la valeur ou la variante rencontrée. `_` signifie « tous les autres cas ».                                     |
| `if let Some(x) = ...`                | « S'il y a bien une valeur, appelons-la `x` et faisons… ».                                                                                                        |
| `loop`, `for ... in`, `continue`, `break` | Boucles : `loop` répète indéfiniment, `for` parcourt une série d'éléments ; `continue` passe au tour suivant, `break` sort de la boucle.                       |
| `&`, `&mut` (devant un nom ou un type) | **Référence** : on prête l'accès à une donnée sans la copier ; `&mut` autorise sa modification. À ne pas confondre avec le `&` binaire placé entre deux nombres. |
| `as u8`, `as u16`...                  | Conversion d'un nombre vers un autre type d'entier.                                                                                                                |
| `.iter()`, `.filter`, `.map`, `.next()` | Chaîne de traitement sur une série d'éléments : parcourir, filtrer, transformer, prendre le premier (chapitre 5).                                               |
| `\|x\| ...`                           | Petite fonction anonyme (*closure*) appliquée à chaque élément.                                                                                                    |
| `'a`                                  | Annotation de « durée de vie » propre à Rust ; peut être ignorée en première lecture.                                                                             |
| `println!`, `eprintln!`               | Affichent un message à l'écran (`eprintln!` pour les erreurs). `{:?}` affiche une donnée sur une ligne, `{:#?}` de façon détaillée et indentée.                  |
| `#[derive(...)]`                      | Demande au compilateur de générer automatiquement des fonctions standard (copie, comparaison, affichage…).                                                        |
| `use`                                 | Importe un élément de la bibliothèque standard de Rust (par exemple `UdpSocket`).                                                                                 |
| `type Result<T> = ...`                | Définit un raccourci de nom de type ; en tête de chaque exemple, il fixe le type d'erreur utilisé partout.                                                        |
| **Remaniement** (*refactoring*)       | Réorganisation du code sans changer ce qu'il fait.                                                                                                                 |

4. Les identifiants du code
---------------------------

Les noms utilisés dans le code sont restés en anglais. Voici leur signification.

### Structures et énumérations

| Identifiant         | Signification                                                                 |
| ------------------- | ----------------------------------------------------------------------------- |
| `BytePacketBuffer`  | Tampon d'octets d'un paquet, avec sa position de lecture/écriture (`pos`).   |
| `DnsHeader`         | En-tête DNS.                                                                  |
| `DnsQuestion`       | Une question DNS (nom `name` et type `qtype`).                                |
| `DnsRecord`         | Un enregistrement DNS.                                                        |
| `DnsPacket`         | Un paquet DNS complet : en-tête et quatre sections.                           |
| `QueryType`         | Type d'enregistrement demandé (A, NS, CNAME, MX, AAAA, ou `UNKNOWN` = inconnu). |
| `ResultCode`        | Code de réponse (voir la section 5).                                          |

### Champs

| Identifiant                          | Signification                                                                                      |
| ------------------------------------ | -------------------------------------------------------------------------------------------------- |
| `buf`, `pos`                         | Contenu du tampon, position courante.                                                              |
| `id`                                 | Identifiant du paquet.                                                                             |
| `recursion_desired`                  | Récursion souhaitée (drapeau RD).                                                                  |
| `truncated_message`                  | Message tronqué (drapeau TC).                                                                      |
| `authoritative_answer`               | Réponse faisant autorité (drapeau AA).                                                             |
| `opcode`                             | Code d'opération (OPCODE).                                                                         |
| `response`                           | Vrai pour une réponse, faux pour une requête (drapeau QR).                                         |
| `rescode`                            | Code de réponse (RCODE).                                                                           |
| `checking_disabled`, `authed_data`, `z` | Les trois bits de la zone Z (liés à DNSSEC) : vérification désactivée, données authentifiées, bit réservé. |
| `recursion_available`                | Récursion disponible (drapeau RA).                                                                 |
| `questions`, `answers`, `authoritative_entries`, `resource_entries` | Dans l'en-tête : nombre d'entrées des sections Question, Réponse, Autorité, Additionnelle. |
| `questions`, `answers`, `authorities`, `resources` | Dans `DnsPacket` : les listes des sections Question, Réponse, Autorité, Additionnelle. |
| `name`, `domain`                     | Nom de domaine concerné.                                                                           |
| `qname`, `qtype`                     | Nom demandé, type demandé (*q* pour *query*).                                                      |
| `addr`                               | Adresse IP.                                                                                        |
| `host`                               | Nom d'hôte vers lequel pointe l'enregistrement (NS, CNAME, MX).                                    |
| `priority`                           | Priorité (MX).                                                                                     |
| `ttl`                                | Durée de validité.                                                                                 |
| `data_len`                           | Longueur des données de l'enregistrement.                                                          |

### Fonctions

| Identifiant                                  | Signification                                                                                  |
| -------------------------------------------- | ---------------------------------------------------------------------------------------------- |
| `new`                                        | Crée un élément vierge.                                                                        |
| `read`, `write`                              | Lire depuis le tampon, écrire dans le tampon.                                                  |
| `read_u16`, `read_u32`, `write_u8`…          | Lire / écrire un entier de 1, 2 ou 4 octets.                                                   |
| `get`, `get_range`, `set`, `set_u16`         | Lire / modifier un octet (ou une plage) à une position donnée, sans déplacer le curseur.       |
| `step`, `seek`                               | Avancer le curseur de *n* octets ; placer le curseur à une position donnée.                   |
| `read_qname`, `write_qname`                  | Lire / écrire un nom de domaine sous forme de labels.                                          |
| `from_num`, `to_num`                         | Convertir un numéro en valeur d'énumération, et inversement.                                   |
| `from_buffer`                                | Décoder un paquet complet à partir d'un tampon.                                                |
| `lookup`                                     | Rechercher : envoyer une question à un serveur et récupérer sa réponse.                        |
| `recursive_lookup`                           | Recherche récursive, en partant des serveurs racine (chapitre 5).                              |
| `handle_query`                               | Traiter une requête reçue par notre serveur.                                                   |
| `get_random_a`                               | Prendre une adresse IP (la première) parmi les enregistrements A de la réponse.                |
| `get_ns`                                     | Lister les serveurs de noms compétents de la section Autorité.                                 |
| `get_resolved_ns`, `get_unresolved_ns`       | Trouver un serveur de noms dont l'adresse IP est fournie / dont seul le nom est connu.         |
| `bind`, `send_to`, `recv_from`               | Ouvrir un socket sur un port, envoyer un paquet, recevoir un paquet.                           |

### Messages affichés par les programmes

| Message                         | Traduction                                                  |
| ------------------------------- | ----------------------------------------------------------- |
| `End of buffer`                 | Fin du tampon (lecture ou écriture au-delà de 512 octets).  |
| `Limit of 5 jumps exceeded`     | Limite de 5 sauts dépassée (paquet suspect).                |
| `Single label exceeds 63 characters of length` | Un label dépasse 63 caractères.              |
| `Skipping record`               | Enregistrement ignoré.                                      |
| `Received query`                | Requête reçue.                                              |
| `Answer`, `Authority`, `Resource` | Réponse, Autorité, Ressource (additionnelle).             |
| `attempting lookup of ... with ns ...` | Tentative de recherche de … auprès du serveur de noms … |
| `An error occurred`             | Une erreur s'est produite.                                  |

5. Les codes de réponse DNS
---------------------------

| Code       | Valeur | Signification                                               |
| ---------- | ------ | ----------------------------------------------------------- |
| `NOERROR`  | 0      | Pas d'erreur.                                               |
| `FORMERR`  | 1      | Erreur de format : la requête est mal formée.               |
| `SERVFAIL` | 2      | Échec du serveur : il n'a pas pu traiter la requête.        |
| `NXDOMAIN` | 3      | Le domaine demandé n'existe pas (*Non-eXistent Domain*).    |
| `NOTIMP`   | 4      | Opération non prise en charge (*Not Implemented*).          |
| `REFUSED`  | 5      | Requête refusée (par exemple, zone non gérée par ce serveur). |

6. Les outils en ligne de commande
----------------------------------

| Outil               | Usage dans le guide                                                                                                      |
| ------------------- | ------------------------------------------------------------------------------------------------------------------------ |
| **Terminal**        | Fenêtre dans laquelle on tape des commandes. Dans le guide, le `#` en début de ligne représente l'invite de commande.    |
| `dig`               | Interroge un serveur DNS et affiche la réponse en détail. `@serveur` choisit le serveur, `-p` le port, `+norecurse` désactive la demande de récursion, `+noedns` désactive l'extension eDNS. |
| `nc` (netcat)       | Envoie ou reçoit des données brutes sur le réseau (`-u` : en UDP, `-l` : en écoute).                                    |
| `hexdump -C`        | Affiche le contenu d'un fichier en hexadécimal, avec la traduction en caractères à droite.                               |
| `cargo run --example sampleN` | Compile et lance l'exemple du chapitre N.                                                                     |
| Ctrl+C              | Interrompt le programme en cours dans le terminal.                                                                       |
