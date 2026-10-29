5 - Résolution récursive
========================

Notre serveur fonctionne, mais dépendre d'un autre serveur pour effectuer
réellement la recherche est agaçant et d'un intérêt limité. C'est le bon moment
pour entrer dans le détail de la façon dont un nom est réellement résolu.

En partant du principe qu'on ne sait rien au départ, la question est d'abord
posée à l'un des 13 serveurs racine d'Internet. Pourquoi 13 ? Parce que c'est le
nombre de serveurs qui tiennent dans un paquet DNS de 512 octets (à strictement
parler, il y a de la place pour 14, mais une marge a été conservée). On pourrait
penser que 13, c'est un peu juste pour gérer tout Internet, et on aurait
raison : il y a 13 serveurs logiques, mais beaucoup plus en réalité. Vous pouvez
en savoir plus [ici](http://www.root-servers.org/). Tout résolveur doit
connaître ces 13 serveurs à l'avance. Un fichier qui les recense tous, au format
bind, est disponible sous le nom
[named.root](https://www.internic.net/domain/named.root). Ces serveurs
contiennent tous les mêmes informations ; pour commencer, nous pouvons donc en
choisir un au hasard. En consultant `named.root`, on voit que l'adresse IP de
*a.root-servers.net* est 198.41.0.4 : nous l'utiliserons pour effectuer notre
requête initiale sur *www.google.com*.

> **Note du traducteur — Serveurs racine et format bind**
>
> - Les 13 serveurs racine sont désignés par les lettres `a` à `m`
>   (`a.root-servers.net`, `b.root-servers.net`…). Chacune de ces 13 adresses
>   est en réalité servie par de nombreuses machines réparties dans le monde
>   entier (technique dite *anycast*) : c'est ce que l'auteur appelle « 13
>   serveurs logiques ».
> - **BIND** est le logiciel de serveur DNS le plus répandu historiquement ;
>   son format de fichier sert de référence pour décrire des enregistrements
>   DNS.
> - Ici, on interroge les serveurs avec `+norecurse` : on se comporte comme le
>   ferait un serveur cache, en demandant à chaque serveur ce qu'il sait
>   lui-même, sans lui demander de faire la recherche à notre place.

```text
# dig +norecurse @198.41.0.4 www.google.com

; <<>> DiG 9.10.3-P4-Ubuntu <<>> +norecurse @198.41.0.4 www.google.com
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 64866
;; flags: qr; QUERY: 1, ANSWER: 0, AUTHORITY: 13, ADDITIONAL: 16

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
;; QUESTION SECTION:
;www.google.com.			IN	A

;; AUTHORITY SECTION:
com.			172800	IN	NS	e.gtld-servers.net.
com.			172800	IN	NS	b.gtld-servers.net.
com.			172800	IN	NS	j.gtld-servers.net.
com.			172800	IN	NS	m.gtld-servers.net.
com.			172800	IN	NS	i.gtld-servers.net.
com.			172800	IN	NS	f.gtld-servers.net.
com.			172800	IN	NS	a.gtld-servers.net.
com.			172800	IN	NS	g.gtld-servers.net.
com.			172800	IN	NS	h.gtld-servers.net.
com.			172800	IN	NS	l.gtld-servers.net.
com.			172800	IN	NS	k.gtld-servers.net.
com.			172800	IN	NS	c.gtld-servers.net.
com.			172800	IN	NS	d.gtld-servers.net.

;; ADDITIONAL SECTION:
e.gtld-servers.net.	172800	IN	A	192.12.94.30
b.gtld-servers.net.	172800	IN	A	192.33.14.30
b.gtld-servers.net.	172800	IN	AAAA	2001:503:231d::2:30
j.gtld-servers.net.	172800	IN	A	192.48.79.30
m.gtld-servers.net.	172800	IN	A	192.55.83.30
i.gtld-servers.net.	172800	IN	A	192.43.172.30
f.gtld-servers.net.	172800	IN	A	192.35.51.30
a.gtld-servers.net.	172800	IN	A	192.5.6.30
a.gtld-servers.net.	172800	IN	AAAA	2001:503:a83e::2:30
g.gtld-servers.net.	172800	IN	A	192.42.93.30
h.gtld-servers.net.	172800	IN	A	192.54.112.30
l.gtld-servers.net.	172800	IN	A	192.41.162.30
k.gtld-servers.net.	172800	IN	A	192.52.178.30
c.gtld-servers.net.	172800	IN	A	192.26.92.30
d.gtld-servers.net.	172800	IN	A	192.31.80.30

;; Query time: 24 msec
;; SERVER: 198.41.0.4#53(198.41.0.4)
;; WHEN: Fri Jul 08 14:09:20 CEST 2016
;; MSG SIZE  rcvd: 531
```

Les serveurs racine ne connaissent pas *www.google.com*, mais ils connaissent
*com* ; leur réponse nous indique donc où aller ensuite. Quelques points sont à
noter :

 * On nous fournit un ensemble d'enregistrements NS, qui se trouvent dans la
   section Autorité. Les enregistrements NS nous donnent *le nom* du serveur de
   noms qui gère un domaine.
 * Le serveur se montre serviable en joignant les enregistrements A
   correspondant aux enregistrements NS, ce qui nous évite une seconde
   recherche.
 * Nous n'avons pas réellement demandé *com*, mais *www.google.com*. Pourtant,
   les enregistrements NS font tous référence à *com*.

> **N.d.T.** — Dans l'en-tête de la réponse, `ANSWER: 0` signifie que le
> serveur n'a pas de réponse directe ; `AUTHORITY: 13` et `ADDITIONAL: 16`
> indiquent qu'il nous renvoie vers d'autres serveurs, avec leurs adresses. Un
> NS donne le *nom* d'un serveur ; pour le contacter, il faut aussi son
> *adresse IP*, d'où l'utilité des enregistrements A de la section
> Additionnelle.

Choisissons un serveur dans le résultat et continuons. *192.5.6.30*, pour
*a.gtld-servers.net*, en vaut bien un autre.

```text
# dig +norecurse @192.5.6.30 www.google.com

; <<>> DiG 9.10.3-P4-Ubuntu <<>> +norecurse @192.5.6.30 www.google.com
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 16229
;; flags: qr; QUERY: 1, ANSWER: 0, AUTHORITY: 4, ADDITIONAL: 5

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
;; QUESTION SECTION:
;www.google.com.			IN	A

;; AUTHORITY SECTION:
google.com.		172800	IN	NS	ns2.google.com.
google.com.		172800	IN	NS	ns1.google.com.
google.com.		172800	IN	NS	ns3.google.com.
google.com.		172800	IN	NS	ns4.google.com.

;; ADDITIONAL SECTION:
ns2.google.com.		172800	IN	A	216.239.34.10
ns1.google.com.		172800	IN	A	216.239.32.10
ns3.google.com.		172800	IN	A	216.239.36.10
ns4.google.com.		172800	IN	A	216.239.38.10

;; Query time: 114 msec
;; SERVER: 192.5.6.30#53(192.5.6.30)
;; WHEN: Fri Jul 08 14:13:26 CEST 2016
;; MSG SIZE  rcvd: 179
```

Nous ne sommes toujours pas arrivés à *www.google.com*, mais nous disposons
maintenant d'un ensemble de serveurs qui gèrent le domaine *google.com*. Faisons
une nouvelle tentative en envoyant notre requête à *216.239.32.10*.

```text
# dig +norecurse @216.239.32.10 www.google.com

; <<>> DiG 9.10.3-P4-Ubuntu <<>> +norecurse @216.239.32.10 www.google.com
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 20432
;; flags: qr aa; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 0

;; QUESTION SECTION:
;www.google.com.            IN  A

;; ANSWER SECTION:
www.google.com.     300 IN  A   216.58.211.132

;; Query time: 10 msec
;; SERVER: 216.239.32.10#53(216.239.32.10)
;; WHEN: Fri Jul 08 14:15:11 CEST 2016
;; MSG SIZE  rcvd: 48
```

Et voilà ! L'adresse IP de *www.google.com*, comme nous le souhaitions.
Récapitulons :

 * *a.root-servers.net* nous dit d'interroger *a.gtld-servers.net*, qui gère com
 * *a.gtld-servers.net* nous dit d'interroger *ns1.google.com*, qui gère
   google.com
 * *ns1.google.com* nous donne l'adresse IP de *www.google.com*

> **N.d.T.** — C'est un peu comme retrouver un document d'urbanisme : on
> s'adresse d'abord à un guichet national, qui vous oriente vers le bon service
> régional, qui vous oriente à son tour vers la mairie compétente, laquelle
> détient l'information. Notez le drapeau `aa` dans la dernière réponse : c'est
> le serveur qui fait autorité pour `google.com` qui répond.

C'est assez typique : la plupart des recherches ne nécessitent que trois étapes,
même sans cache. Il est toutefois possible d'avoir des serveurs de noms pour des
sous-domaines, puis d'autres pour des sous-sous-domaines. En pratique, un
serveur DNS tient un cache, et la plupart des TLD y sont déjà connus. La plupart
des requêtes ne nécessitent donc que deux recherches de la part du serveur, et
souvent une seule, voire aucune.

### Étendre DnsPacket pour les recherches récursives

Avant de continuer, il nous faut quelques fonctions utilitaires dans
`DnsPacket`.

> **Note du traducteur — Les chaînes d'itérateurs**
>
> Ces fonctions utilisent un style d'écriture typique de Rust, qu'on peut lire
> comme une chaîne de traitement : `.iter()` parcourt les éléments d'une liste
> un par un ; chaque étape suivante filtre (`.filter`), transforme (`.map`) ou
> fait les deux à la fois (`.filter_map`) ; `.flat_map` remplace chaque élément
> par une nouvelle série d'éléments ; enfin, `.next()` prend le premier élément
> qui sort de la chaîne. `Option<...>` signifie « une valeur ou rien » :
> `Some(x)` quand on a trouvé quelque chose, `None` sinon. `|record| ...`
> désigne une petite fonction anonyme appliquée à chaque élément. Les `'a` sont
> des annotations techniques de Rust (durées de vie), que l'on peut ignorer en
> première lecture.
>
> En clair :
>
> - `get_random_a` renvoie la première adresse IP trouvée dans la section
>   Réponse ;
> - `get_ns` liste les serveurs de noms de la section Autorité qui sont
>   compétents pour le nom recherché (leur domaine est la fin du nom
>   recherché : `com` pour `www.google.com`) ;
> - `get_resolved_ns` cherche, pour ces serveurs, une adresse IP déjà fournie
>   dans la section Additionnelle ;
> - `get_unresolved_ns` renvoie simplement le nom d'un de ces serveurs, quand
>   aucune adresse n'est fournie.

```rust
impl DnsPacket {

    - (code déjà présenté, omis ici) -

    /// Il est utile de pouvoir choisir un enregistrement A au hasard dans un
    /// paquet. Quand on obtient plusieurs IP pour un même nom, peu importe
    /// laquelle on retient : on peut donc en prendre une au hasard. (N.d.T. :
    /// en pratique, ce code prend simplement la première.)
    pub fn get_random_a(&self) -> Option<Ipv4Addr> {
        self.answers
            .iter()
            .filter_map(|record| match record {
                DnsRecord::A { addr, .. } => Some(*addr),
                _ => None,
            })
            .next()
    }

    /// Fonction utilitaire qui renvoie un itérateur sur tous les serveurs de
    /// noms de la section Autorité, sous forme de couples (domaine, hôte)
    fn get_ns<'a>(&'a self, qname: &'a str) -> impl Iterator<Item = (&'a str, &'a str)> {
        self.authorities
            .iter()
            // En pratique, ce sont toujours des enregistrements NS dans les
            // paquets bien formés. On convertit chaque enregistrement NS en un
            // couple ne contenant que les données utiles, pour simplifier la
            // suite.
            .filter_map(|record| match record {
                DnsRecord::NS { domain, host, .. } => Some((domain.as_str(), host.as_str())),
                _ => None,
            })
            // On écarte les serveurs qui ne font pas autorité pour notre
            // requête
            .filter(move |(domain, _)| qname.ends_with(*domain))
    }

    /// Les serveurs de noms joignent souvent les enregistrements A
    /// correspondants lorsqu'ils répondent avec des NS. On en profite pour
    /// écrire une fonction qui renvoie, si possible, l'adresse IP effective
    /// d'un enregistrement NS.
    pub fn get_resolved_ns(&self, qname: &str) -> Option<Ipv4Addr> {
        // Itérateur sur les serveurs de noms de la section Autorité
        self.get_ns(qname)
            // Il faut maintenant chercher un enregistrement A correspondant
            // dans la section Additionnelle. Comme seul le premier
            // enregistrement valide nous intéresse, il suffit de construire un
            // flux des enregistrements qui correspondent.
            .flat_map(|(_, host)| {
                self.resources
                    .iter()
                    // On ne garde que les enregistrements A dont le domaine
                    // correspond à l'hôte de l'enregistrement NS en cours de
                    // traitement
                    .filter_map(move |record| match record {
                        DnsRecord::A { domain, addr, .. } if domain == host => Some(addr),
                        _ => None,
                    })
            })
            .map(|addr| *addr)
            // Enfin, on prend la première entrée valide
            .next()
    }

    /// Cependant, tous les serveurs de noms ne sont pas aussi aimables.
    /// Parfois, la section Additionnelle ne contient aucun enregistrement A, et
    /// il faut effectuer une *autre* résolution en cours de route. Pour cela,
    /// on ajoute une méthode qui renvoie le nom d'hôte d'un serveur de noms
    /// approprié.
    pub fn get_unresolved_ns<'a>(&'a self, qname: &'a str) -> Option<&'a str> {
        // Itérateur sur les serveurs de noms de la section Autorité
        self.get_ns(qname)
            .map(|(_, host)| host)
            // Enfin, on prend la première entrée valide
            .next()
    }

} // Fin de DnsPacket
```

### Implémenter la recherche récursive

Passons sans tarder à notre nouvelle fonction `recursive_lookup` :

```rust
fn recursive_lookup(qname: &str, qtype: QueryType) -> Result<DnsPacket> {
    // Pour l'instant, on commence toujours par *a.root-servers.net*.
    let mut ns = "198.41.0.4".parse::<Ipv4Addr>().unwrap();

    // Le nombre d'étapes n'étant pas connu à l'avance, on entre dans une boucle
    // sans limite.
    loop {
        println!("attempting lookup of {:?} {} with ns {}", qtype, qname, ns);

        // L'étape suivante consiste à envoyer la requête au serveur courant.
        let ns_copy = ns;

        let server = (ns_copy, 53);
        let response = lookup(qname, qtype, server)?;

        // Si la section Réponse contient des entrées et qu'il n'y a pas
        // d'erreur, c'est terminé !
        if !response.answers.is_empty() && response.header.rescode == ResultCode::NOERROR {
            return Ok(response);
        }

        // On peut aussi recevoir une réponse `NXDOMAIN` : c'est la façon dont
        // le serveur de noms faisant autorité nous indique que le nom n'existe
        // pas.
        if response.header.rescode == ResultCode::NXDOMAIN {
            return Ok(response);
        }

        // Sinon, on cherche un nouveau serveur de noms à partir d'un
        // enregistrement NS et de l'enregistrement A correspondant dans la
        // section Additionnelle. Si on en trouve un, on change de serveur de
        // noms et on recommence la boucle.
        if let Some(new_ns) = response.get_resolved_ns(qname) {
            ns = new_ns;

            continue;
        }

        // Sinon, il faut résoudre l'adresse IP d'un enregistrement NS. S'il n'y
        // a aucun enregistrement NS, on s'en tient à ce que le dernier serveur
        // nous a répondu.
        let new_ns_name = match response.get_unresolved_ns(qname) {
            Some(x) => x,
            None => return Ok(response),
        };

        // On s'enfonce ici d'un niveau en lançant une _autre_ séquence de
        // résolution au beau milieu de la séquence en cours. Avec un peu de
        // chance, elle nous donnera l'adresse IP d'un serveur de noms
        // approprié.
        let recursive_response = recursive_lookup(&new_ns_name, QueryType::A)?;

        // Enfin, on choisit une adresse IP au hasard dans le résultat et on
        // relance la boucle. Si aucun enregistrement de ce type n'est
        // disponible, on renvoie là encore le dernier résultat obtenu.
        if let Some(new_ns) = recursive_response.get_random_a() {
            ns = new_ns;
        } else {
            return Ok(response);
        }
    }
}
```

> **Note du traducteur — L'algorithme en clair**
>
> On part du serveur racine `198.41.0.4`, puis on répète :
>
> 1. poser la question au serveur courant ;
> 2. s'il donne la réponse (ou affirme que le nom n'existe pas, `NXDOMAIN`),
>    c'est terminé ;
> 3. sinon, s'il indique un serveur plus compétent *avec* son adresse IP, on
>    passe à ce serveur et on recommence ;
> 4. sinon, s'il indique seulement le *nom* d'un serveur plus compétent, on
>    lance une nouvelle recherche complète pour trouver l'adresse IP de ce
>    serveur, puis on recommence avec lui ;
> 5. s'il n'indique rien d'exploitable, on renvoie sa réponse telle quelle.
>
> L'étape 4 est une **récursion** : la fonction `recursive_lookup` s'appelle
> elle-même, comme un sous-traitant qui doit lui-même consulter un autre
> sous-traitant avant de pouvoir avancer. Ce code est pédagogique : il ne fixe
> aucune limite au nombre d'étapes et ne garde aucun cache. Il n'est pas conçu
> pour un usage en production.

Cela nécessite aussi une petite modification de la fonction `lookup`, puisqu'il
faut lui indiquer quel serveur utiliser. Nous ajoutons un paramètre `server` à
la signature de la fonction et supprimons la variable `server` écrite en dur que
nous utilisions au chapitre 4 :

```rust
fn lookup(qname: &str, qtype: QueryType, server: (Ipv4Addr, u16)) -> Result<DnsPacket> {
```

> **N.d.T.** — `server: (Ipv4Addr, u16)` est un couple (adresse IP, numéro de
> port). La **signature** d'une fonction est sa première ligne, qui décrit ce
> qu'elle reçoit et ce qu'elle renvoie.

### Essayer la recherche récursive

Il ne reste plus qu'à modifier notre fonction `handle_query` pour qu'elle
utilise `recursive_lookup` :

```rust
fn handle_query(socket: &UdpSocket) -> Result<()> {

    - (code déjà présenté, omis ici) -

            println!("Received query: {:?}", question);
            if let Ok(result) = recursive_lookup(&question.name, question.qtype) {
                packet.questions.push(question.clone());
                packet.header.rescode = result.header.rescode;

    - (code déjà présenté, omis ici) -

}
```

Essayons !

> **N.d.T.** — Même procédure qu'au chapitre 4 : `cargo run --example sample5`
> dans un premier terminal, puis la commande `dig` ci-dessous dans un second.

```text
# dig @127.0.0.1 -p 2053 www.google.com

; <<>> DiG 9.10.3-P4-Ubuntu <<>> @127.0.0.1 -p 2053 www.google.com
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 41892
;; flags: qr rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 0

;; QUESTION SECTION:
;www.google.com.			IN	A

;; ANSWER SECTION:
www.google.com.		300	IN	A	216.58.211.132

;; Query time: 76 msec
;; SERVER: 127.0.0.1#2053(127.0.0.1)
;; WHEN: Fri Jul 08 14:31:39 CEST 2016
;; MSG SIZE  rcvd: 62
```

Dans la fenêtre de notre serveur, on voit :

```text
Received query: DnsQuestion { name: "www.google.com", qtype: A }
attempting lookup of A www.google.com with ns 198.41.0.4
attempting lookup of A www.google.com with ns 192.12.94.30
attempting lookup of A www.google.com with ns 216.239.34.10
Answer: A { domain: "www.google.com", addr: 216.58.211.132, ttl: 300 }
```

> **N.d.T.** — *attempting lookup of A www.google.com with ns …* signifie
> « tentative de recherche de A www.google.com auprès du serveur de noms … ».
> On retrouve les trois étapes : serveur racine, serveur de `.com`, puis
> serveur de `google.com`. Le programme prend à chaque fois le premier serveur
> de la liste, d'où des adresses différentes de celles choisies à la main plus
> haut. Le temps de réponse (`Query time: 76 msec`) est plus long qu'au
> chapitre 4, puisque notre serveur fait désormais tout le chemin lui-même, sans
> cache.

Cela reproduit la démarche manuelle que nous avons suivie plus haut. Nous savons
désormais résoudre un domaine en partant de la liste des serveurs racine. Nous
disposons maintenant d'un serveur DNS pleinement fonctionnel, même s'il est loin
d'être optimal.

Il y a beaucoup de choses que nous pourrions mieux faire. Par exemple, ce
serveur ne sait pas réellement traiter plusieurs requêtes en parallèle. Il ne
sait ni envoyer ni recevoir de requêtes sur TCP. Nous ne pouvons pas l'utiliser
pour héberger nos propres zones et le faire agir comme serveur faisant autorité.
L'absence de prise en charge de DNSSEC nous expose aux attaques par
empoisonnement DNS, dans lesquelles un serveur malveillant peut renvoyer des
enregistrements relatifs au domaine de quelqu'un d'autre.

> **N.d.T.** — L'**empoisonnement DNS** (*DNS poisoning*) consiste à glisser de
> fausses réponses dans un serveur, pour qu'un nom légitime (celui d'une banque,
> par exemple) pointe vers le serveur d'un attaquant. C'est l'équivalent d'un
> faux avis de changement d'adresse : le courrier part ensuite vers la mauvaise
> destination. DNSSEC permet de vérifier, par signature, que la réponse provient
> bien du serveur faisant autorité.

Beaucoup de ces problèmes ont été corrigés dans mon propre projet
[hermes](https://github.com/EmilHernvall/hermes) : vous pouvez aller y voir
comment je m'y suis pris, ou bien poursuivre par vous-même à partir d'ici. À
moins que vous n'ayez eu votre dose de DNS pour le moment… :) Quoi qu'il en
soit, j'espère que ce guide vous a permis de mieux comprendre le fonctionnement
du DNS.
