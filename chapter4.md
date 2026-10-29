4 - Mon premier serveur DNS
===========================

Arrivés jusqu'ici, nous sommes prêts à faire une première tentative d'écriture
d'un véritable serveur. Dans la réalité, les serveurs DNS se déclinent en deux
variétés :

 * Serveur faisant autorité (*Authoritative Server*) - Un serveur DNS qui
   héberge une ou plusieurs « zones ». Par exemple, les serveurs faisant
   autorité pour la zone google.com sont ns1.google.com, ns2.google.com,
   ns3.google.com et ns4.google.com.
 * Serveur cache (*Caching Server*) - Un serveur DNS qui traite les recherches
   en vérifiant d'abord dans son cache s'il connaît déjà l'enregistrement
   demandé, et sinon, en effectuant une recherche récursive pour le trouver.
   C'est le cas du serveur DNS qui tourne probablement sur votre box ou votre
   routeur, du serveur DNS que votre fournisseur d'accès vous attribue via
   DHCP, ainsi que des serveurs DNS publics de Google, 8.8.8.8 et 8.8.4.4.

> **Note du traducteur — Deux métiers différents**
>
> - Une **zone** est la portion de l'annuaire DNS dont un serveur a la charge
>   (par exemple, tout ce qui se termine par `google.com`).
> - Le **serveur faisant autorité** détient le registre original de sa zone,
>   comme le service du cadastre détient le plan officiel de sa commune : il ne
>   répond que sur son territoire, mais sa réponse fait foi.
> - Le **serveur cache** (ou résolveur récursif) est l'intermédiaire qui va
>   chercher l'information pour vous auprès des services compétents, puis en
>   garde une copie pendant la durée du TTL pour répondre plus vite la fois
>   suivante.
> - **DHCP** est le protocole qui configure automatiquement votre ordinateur
>   lorsqu'il se connecte à un réseau (adresse IP, serveur DNS à utiliser…).

Strictement parlant, rien n'empêche un serveur de faire les deux, mais en
pratique, ces deux rôles sont généralement exclusifs l'un de l'autre. Cela
explique aussi l'importance des drapeaux `RD` (Recursion Desired) et `RA`
(Recursion Available) dans l'en-tête du paquet : un résolveur minimal qui
interroge un serveur cache active le drapeau `RD`, et comme le serveur accepte
ce type de requête, il effectue la recherche et envoie une réponse avec le
drapeau `RA` activé. Cela ne fonctionne pas avec un serveur faisant autorité,
qui ne répond qu'aux requêtes portant sur les zones qu'il héberge et renvoie donc
une réponse d'erreur à toute requête dont le drapeau `RD` est activé.

> **N.d.T.** — Les exemples qui suivent nuancent cette dernière phrase : un
> serveur faisant autorité refuse les requêtes portant sur des zones qu'il
> n'héberge pas, mais il répond normalement pour ses propres zones, même si
> `RD` est activé ; il se contente alors de ne pas activer `RA`.

Mais ne me croyez pas sur parole : vérifions-le ! Commençons par utiliser
`8.8.8.8` pour rechercher *yahoo.com* :

> **N.d.T.** — Dans la commande `dig @8.8.8.8 yahoo.com`, la partie `@8.8.8.8`
> désigne le serveur à interroger. Les lignes `OPT PSEUDOSECTION` et `EDNS`
> concernent l'extension eDNS évoquée au chapitre 1 ; on peut les ignorer.

```text
# dig @8.8.8.8 yahoo.com

; <<>> DiG 9.10.3-P4-Ubuntu <<>> +recurse @8.8.8.8 yahoo.com
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 53231
;; flags: qr rd ra; QUERY: 1, ANSWER: 3, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 512
;; QUESTION SECTION:
;yahoo.com.			IN	A

;; ANSWER SECTION:
yahoo.com.		1051	IN	A	98.138.253.109
yahoo.com.		1051	IN	A	98.139.183.24
yahoo.com.		1051	IN	A	206.190.36.45

;; Query time: 1 msec
;; SERVER: 8.8.8.8#53(8.8.8.8)
;; WHEN: Fri Jul 08 11:43:55 CEST 2016
;; MSG SIZE  rcvd: 86
```

Cela fonctionne comme prévu. Essayons maintenant d'envoyer la même requête à
l'un des serveurs qui hébergent la zone *google.com* :

```text
# dig @ns1.google.com yahoo.com

; <<>> DiG 9.10.3-P4-Ubuntu <<>> +recurse @ns1.google.com yahoo.com
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: REFUSED, id: 12034
;; flags: qr rd; QUERY: 1, ANSWER: 0, AUTHORITY: 0, ADDITIONAL: 0
;; WARNING: recursion requested but not available

;; QUESTION SECTION:
;yahoo.com.			IN	A

;; Query time: 10 msec
;; SERVER: 216.239.32.10#53(216.239.32.10)
;; WHEN: Fri Jul 08 11:44:07 CEST 2016
;; MSG SIZE  rcvd: 27
```

Remarquez que le statut de la réponse indique `REFUSED` (refusé) ! `dig` nous
avertit aussi que, si le drapeau `RD` était bien activé dans la requête, le
serveur n'a pas activé le drapeau `RA` dans la réponse (*recursion requested but
not available* : « récursion demandée mais non disponible »). Nous pouvons
toutefois utiliser ce même serveur pour *google.com* :

```text
dig @ns1.google.com google.com                                                                                                                                                                                                                                 <<<

; <<>> DiG 9.10.3-P4-Ubuntu <<>> +recurse @ns1.google.com google.com
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 28058
;; flags: qr aa rd; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 0
;; WARNING: recursion requested but not available

;; QUESTION SECTION:
;google.com.			IN	A

;; ANSWER SECTION:
google.com.		300	IN	A	216.58.211.142

;; Query time: 10 msec
;; SERVER: 216.239.32.10#53(216.239.32.10)
;; WHEN: Fri Jul 08 11:46:27 CEST 2016
;; MSG SIZE  rcvd: 44
```

Pas d'erreur cette fois ; `dig` nous avertit cependant toujours que la récursion
n'est pas disponible. Nous pouvons désactiver explicitement la demande de
récursion avec `+norecurse`, ce qui fait disparaître l'avertissement :

```text
# dig +norecurse @ns1.google.com google.com

; <<>> DiG 9.10.3-P4-Ubuntu <<>> +norecurse @ns1.google.com google.com
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 15850
;; flags: qr aa; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 0

;; QUESTION SECTION:
;google.com.			IN	A

;; ANSWER SECTION:
google.com.		300	IN	A	216.58.211.142

;; Query time: 10 msec
;; SERVER: 216.239.32.10#53(216.239.32.10)
;; WHEN: Fri Jul 08 11:47:52 CEST 2016
;; MSG SIZE  rcvd: 44
```

Cette dernière requête est exactement le type de requête qu'un serveur cache est
censé envoyer lorsqu'il résout un nom de manière récursive.

> **N.d.T.** — Remarquez le drapeau `aa` (*Authoritative Answer*) dans les deux
> dernières réponses : le serveur signale que sa réponse fait autorité, puisque
> `google.com` fait partie de sa zone.

Pour notre première incursion dans l'écriture de notre propre serveur, nous
ferons quelque chose d'encore plus simple : un serveur qui se contente de
transmettre les requêtes à un autre serveur cache, autrement dit un « serveur
mandataire DNS » (*DNS proxy server*). Comme nous avons déjà fait le plus dur,
ce sera plutôt rapide !

> **N.d.T.** — Notre serveur jouera donc le rôle d'un relais : il reçoit une
> question, la retransmet telle quelle à Google (`8.8.8.8`), puis renvoie la
> réponse obtenue à celui qui l'a interrogé.

### Isoler la recherche dans une fonction séparée

Nous commençons par un petit remaniement (*refactoring*) : nous déplaçons notre
code de recherche dans une fonction séparée. C'est pour l'essentiel le même code
que celui de notre fonction `main` du chapitre précédent.

```rust
fn lookup(qname: &str, qtype: QueryType) -> Result<DnsPacket> {
    // Transmet les requêtes au DNS public de Google
    let server = ("8.8.8.8", 53);

    let socket = UdpSocket::bind(("0.0.0.0", 43210))?;

    let mut packet = DnsPacket::new();

    packet.header.id = 6666;
    packet.header.questions = 1;
    packet.header.recursion_desired = true;
    packet
        .questions
        .push(DnsQuestion::new(qname.to_string(), qtype));

    let mut req_buffer = BytePacketBuffer::new();
    packet.write(&mut req_buffer)?;
    socket.send_to(&req_buffer.buf[0..req_buffer.pos], server)?;

    let mut res_buffer = BytePacketBuffer::new();
    socket.recv_from(&mut res_buffer.buf)?;

    DnsPacket::from_buffer(&mut res_buffer)
}

```

> **N.d.T.** — `lookup` (« recherche ») prend en entrée le nom et le type
> recherchés, et renvoie le paquet de réponse (`Result<DnsPacket>` : soit un
> paquet, soit une erreur).

### Implémenter notre premier serveur

Écrivons maintenant le code de notre serveur. Il faut d'abord mettre quelques
éléments en place.

```rust
/// Traite un seul paquet entrant
fn handle_query(socket: &UdpSocket) -> Result<()> {
    // Le socket étant prêt, on peut lire un paquet. Cet appel est bloquant : le
    // programme attend jusqu'à ce qu'un paquet arrive.
    let mut req_buffer = BytePacketBuffer::new();

    // La fonction `recv_from` écrit les données dans le tampon fourni et
    // renvoie la longueur des données lues ainsi que l'adresse de l'expéditeur.
    // La longueur ne nous intéresse pas, mais il faut conserver l'adresse de
    // l'expéditeur pour pouvoir lui répondre ensuite.
    let (_, src) = socket.recv_from(&mut req_buffer.buf)?;

    // Ensuite, `DnsPacket::from_buffer` sert à convertir les octets bruts en un
    // `DnsPacket`.
    let mut request = DnsPacket::from_buffer(&mut req_buffer)?;

    // Crée et initialise le paquet de réponse
    let mut packet = DnsPacket::new();
    packet.header.id = request.header.id;
    packet.header.recursion_desired = true;
    packet.header.recursion_available = true;
    packet.header.response = true;

    // Dans le cas normal, le paquet contient exactement une question
    if let Some(question) = request.questions.pop() {
        println!("Received query: {:?}", question);

        // Tout étant en place et conforme, la requête peut être transmise au
        // serveur cible. Il est toujours possible que la requête échoue : dans
        // ce cas, on renvoie le code de réponse `SERVFAIL` pour en informer le
        // client. Si au contraire tout se passe comme prévu, la question et les
        // enregistrements de réponse sont recopiés dans notre paquet de
        // réponse.
        if let Ok(result) = lookup(&question.name, question.qtype) {
            packet.questions.push(question);
            packet.header.rescode = result.header.rescode;

            for rec in result.answers {
                println!("Answer: {:?}", rec);
                packet.answers.push(rec);
            }
            for rec in result.authorities {
                println!("Authority: {:?}", rec);
                packet.authorities.push(rec);
            }
            for rec in result.resources {
                println!("Resource: {:?}", rec);
                packet.resources.push(rec);
            }
        } else {
            packet.header.rescode = ResultCode::SERVFAIL;
        }
    }
    // Les données reçues d'expéditeurs quelconques étant peu fiables, il faut
    // vérifier qu'une question est bien présente. Sinon, on renvoie `FORMERR`
    // pour signaler que l'expéditeur a envoyé une requête mal formée.
    else {
        packet.header.rescode = ResultCode::FORMERR;
    }

    // Il ne reste plus qu'à encoder notre réponse et à l'envoyer !
    let mut res_buffer = BytePacketBuffer::new();
    packet.write(&mut res_buffer)?;

    let len = res_buffer.pos();
    let data = res_buffer.get_range(0, len)?;

    socket.send_to(data, src)?;

    Ok(())
}

fn main() -> Result<()> {
    // Ouvre un socket UDP sur le port 2053
    let socket = UdpSocket::bind(("0.0.0.0", 2053))?;

    // Pour l'instant, les requêtes sont traitées l'une après l'autre : on lance
    // donc une boucle infinie qui les sert successivement.
    loop {
        match handle_query(&socket) {
            Ok(_) => {},
            Err(e) => eprintln!("An error occurred: {}", e),
        }
    }
}
```

> **Note du traducteur — Le fonctionnement du serveur en résumé**
>
> 1. `main` ouvre un socket sur le port 2053, puis tourne en boucle (`loop`)
>    indéfiniment, en appelant `handle_query` pour chaque paquet reçu.
> 2. `handle_query` attend un paquet, le décode, prépare un paquet de réponse
>    portant le même identifiant, puis :
>    - s'il y a une question, la transmet à Google via `lookup` et recopie la
>      réponse obtenue (ou renvoie `SERVFAIL` en cas d'échec) ;
>    - s'il n'y a pas de question, renvoie `FORMERR` (requête mal formée).
> 3. La réponse est encodée puis renvoyée à l'expéditeur (`src`).
>
> `if let Some(question) = ...` signifie « s'il y a bien une question, alors… » ;
> `pop()` retire et renvoie le dernier élément d'une liste. Les messages
> *Received query*, *Answer*, *Authority*, *Resource* et *An error occurred*
> signifient respectivement « requête reçue », « réponse », « autorité »,
> « ressource (additionnelle) » et « une erreur s'est produite ».

L'idiome `match` pour la gestion des erreurs est utilisé à répétition ici, car
nous voulons à tout prix éviter d'interrompre notre boucle de traitement des
requêtes. C'est un peu verbeux, et nous aimerions normalement utiliser `try!` à
la place. Malheureusement, ce n'est pas possible ici, puisque nous sommes dans
la fonction `main`, qui ne renvoie pas de `Result`.

> **N.d.T.** — Ce paragraphe date de la première version du guide (2016) et ne
> correspond plus exactement au code. `try!` est l'ancêtre de l'opérateur `?`
> présenté au chapitre 1, et la fonction `main` actuelle renvoie bien un
> `Result`. La vraie raison d'utiliser `match` est la suivante : avec `?`, la
> première erreur (par exemple un paquet malformé envoyé par n'importe qui)
> ferait s'arrêter tout le serveur ; avec `match`, l'erreur est simplement
> affichée (`eprintln!`) et la boucle continue de servir les requêtes
> suivantes.

Terminé ! Essayons-le ! Nous lançons notre serveur dans un terminal et utilisons
`dig` pour effectuer une recherche dans un second terminal.

> **Note du traducteur — Lancer cet exemple**
>
> - Dans un premier terminal : `cargo run --example sample4`. Le programme
>   n'affiche rien et attend des requêtes : c'est normal.
> - Dans un second terminal : `dig @127.0.0.1 -p 2053 google.com`.
>   `127.0.0.1` est l'adresse spéciale qui désigne « cette machine-ci », et
>   `-p 2053` indique le port de notre serveur.
> - Le serveur utilise le port 2053 plutôt que le port DNS standard (53),
>   probablement parce que sur la plupart des systèmes, notamment Linux, ouvrir
>   un port inférieur à 1024 nécessite des droits d'administrateur.
> - Pour arrêter le serveur : Ctrl+C dans le premier terminal.

```text
# dig @127.0.0.1 -p 2053 google.com

; <<>> DiG 9.10.3-P4-Ubuntu <<>> @127.0.0.1 -p 2053 google.com
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 47200
;; flags: qr rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 0

;; QUESTION SECTION:
;google.com.			IN	A

;; ANSWER SECTION:
google.com.		68	IN	A	216.58.211.142

;; Query time: 1 msec
;; SERVER: 127.0.0.1#2053(127.0.0.1)
;; WHEN: Fri Jul 08 12:07:44 CEST 2016
;; MSG SIZE  rcvd: 54
```

Dans le terminal de notre serveur, on voit :

```text
Received query: DnsQuestion { name: "google.com", qtype: A }
Answer: A { domain: "google.com", addr: 216.58.211.142, ttl: 96 }
```

Réussi ! En moins de 800 lignes de code, nous avons construit un serveur DNS
capable de répondre à des requêtes portant sur plusieurs types d'enregistrements
différents !

Au chapitre suivant, nous nous affranchirons de notre dépendance à un résolveur
existant : [Chapitre 5 - Résolution récursive](chapter5.md)
