3 - Ajouter d'autres types d'enregistrements
============================================

Utilisons notre programme pour rechercher « yahoo.com ».

> **N.d.T.** — Il suffit pour cela de modifier, dans la fonction `main` du
> chapitre 2, la ligne qui définit le nom recherché :

```rust
let qname = "www.yahoo.com";
```

L'exécution donne :

```text
DnsHeader {
    id: 6666,
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
    answers: 3,
    authoritative_entries: 0,
    resource_entries: 0
}
DnsQuestion {
    name: "www.yahoo.com",
    qtype: A
}
UNKNOWN {
    domain: "www.yahoo.com",
    qtype: 5,
    data_len: 15,
    ttl: 259
}
A {
    domain: "fd-fp3.wg1.b.yahoo.com",
    addr: 46.228.47.115,
    ttl: 19
}
A {
    domain: "fd-fp3.wg1.b.yahoo.com",
    addr: 46.228.47.114,
    ttl: 19
}
```

Étrange : nous obtenons un enregistrement UNKNOWN (inconnu) en plus de deux
enregistrements A. L'enregistrement UNKNOWN, de type 5, est un CNAME. Il existe
un bon nombre de types d'enregistrements DNS, dont beaucoup ne sont jamais
utilisés en pratique. Cela dit, jetons un œil à quelques-uns, essentiels :

| ID  | Nom   | Description                                                             | Encodage                                               |
| --- | ----- | ----------------------------------------------------------------------- | ------------------------------------------------------ |
| 1   | A     | Alias - Associe des noms à des adresses IP                              | Préambule + 4 octets pour l'adresse IPv4               |
| 2   | NS    | Name Server - L'adresse du serveur DNS d'un domaine                     | Préambule + séquence de labels                         |
| 5   | CNAME | Canonical Name (nom canonique) - Associe des noms à d'autres noms       | Préambule + séquence de labels                         |
| 15  | MX    | Mail eXchange - L'hôte du serveur de messagerie d'un domaine            | Préambule + 2 octets de priorité + séquence de labels  |
| 28  | AAAA  | Alias IPv6                                                              | Préambule + 16 octets pour l'adresse IPv6              |

> **Note du traducteur — Les types d'enregistrements en pratique**
>
> - **A** : dans la norme, « A » signifie *Address* (adresse), et non « Alias »
>   comme l'écrit l'auteur ; c'est l'enregistrement qui donne l'adresse IPv4
>   d'un nom. Le véritable alias, c'est le CNAME.
> - **NS** (*Name Server*) : indique quel serveur DNS gère un domaine, un peu
>   comme le nom du service qui tient le registre d'une commune.
> - **CNAME** : un alias, qui dit « ce nom est un autre nom pour… ». Ici,
>   `www.yahoo.com` renvoie vers `fd-fp3.wg1.b.yahoo.com`, qui porte, lui, les
>   adresses IP. C'est le principe d'une domiciliation : on écrit à une adresse,
>   le courrier est réexpédié vers l'adresse réelle.
> - **MX** (*Mail eXchange*) : indique quel serveur reçoit les courriels d'un
>   domaine (pour une adresse `contact@exemple.fr`, c'est l'enregistrement MX
>   de `exemple.fr` qui est consulté). La **priorité** permet d'en déclarer
>   plusieurs : la plus petite valeur est essayée en premier.
> - **AAAA** : l'équivalent de A pour les adresses **IPv6**, la nouvelle
>   génération d'adresses Internet, beaucoup plus longues (16 octets au lieu de
>   4), et notées en groupes hexadécimaux séparés par `:` (par exemple
>   `2001:503:231d::2:30`). Le nom « AAAA » vient de ce qu'une adresse IPv6
>   est quatre fois plus longue qu'une adresse IPv4.

### Ajouter des types d'enregistrements à QueryType

Ajoutons-les à notre code ! Commençons par mettre à jour notre énumération
`QueryType` :

```rust
#[derive(PartialEq, Eq, Debug, Clone, Hash, Copy)]
pub enum QueryType {
    UNKNOWN(u16),
    A,     // 1
    NS,    // 2
    CNAME, // 5
    MX,    // 15
    AAAA,  // 28
}
```

Il faut aussi modifier nos fonctions utilitaires.

```rust
impl QueryType {
    pub fn to_num(&self) -> u16 {
        match *self {
            QueryType::UNKNOWN(x) => x,
            QueryType::A => 1,
            QueryType::NS => 2,
            QueryType::CNAME => 5,
            QueryType::MX => 15,
            QueryType::AAAA => 28,
        }
    }

    pub fn from_num(num: u16) -> QueryType {
        match num {
            1 => QueryType::A,
            2 => QueryType::NS,
            5 => QueryType::CNAME,
            15 => QueryType::MX,
            28 => QueryType::AAAA,
            _ => QueryType::UNKNOWN(num),
        }
    }
}
```

### Étendre DnsRecord pour lire les nouveaux types d'enregistrements

Il nous faut maintenant un moyen de stocker les données de ces enregistrements ;
nous apportons donc quelques modifications à `DnsRecord`.

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
    NS {
        domain: String,
        host: String,
        ttl: u32,
    }, // 2
    CNAME {
        domain: String,
        host: String,
        ttl: u32,
    }, // 5
    MX {
        domain: String,
        priority: u16,
        host: String,
        ttl: u32,
    }, // 15
    AAAA {
        domain: String,
        addr: Ipv6Addr,
        ttl: u32,
    }, // 28
}
```

> **N.d.T.** — Le type `Ipv6Addr` doit être importé en tête du fichier :
> remplacez `use std::net::Ipv4Addr;` par
> `use std::net::{Ipv4Addr, Ipv6Addr};`, sinon la compilation échoue avec
> l'erreur *cannot find type `Ipv6Addr` in this scope*. Voir le début de
> `examples/sample3.rs`.

> **N.d.T.** — Chaque variante de l'énumération décrit les informations propres
> à un type d'enregistrement : `addr` (l'adresse) pour A et AAAA, `host` (le nom
> d'hôte vers lequel pointe l'enregistrement) pour NS et CNAME, `priority` et
> `host` pour MX. Tous ont en commun `domain` (le nom concerné) et `ttl`.

Voici le gros du travail. Nous devons étendre les fonctions d'écriture et de
lecture des enregistrements. En commençant par la lecture, nous la complétons
avec du code supplémentaire pour chaque type d'enregistrement. Tout d'abord, on
lit le préambule commun :

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
            QueryType::AAAA => {
                let raw_addr1 = buffer.read_u32()?;
                let raw_addr2 = buffer.read_u32()?;
                let raw_addr3 = buffer.read_u32()?;
                let raw_addr4 = buffer.read_u32()?;
                let addr = Ipv6Addr::new(
                    ((raw_addr1 >> 16) & 0xFFFF) as u16,
                    ((raw_addr1 >> 0) & 0xFFFF) as u16,
                    ((raw_addr2 >> 16) & 0xFFFF) as u16,
                    ((raw_addr2 >> 0) & 0xFFFF) as u16,
                    ((raw_addr3 >> 16) & 0xFFFF) as u16,
                    ((raw_addr3 >> 0) & 0xFFFF) as u16,
                    ((raw_addr4 >> 16) & 0xFFFF) as u16,
                    ((raw_addr4 >> 0) & 0xFFFF) as u16,
                );

                Ok(DnsRecord::AAAA {
                    domain: domain,
                    addr: addr,
                    ttl: ttl,
                })
            }
            QueryType::NS => {
                let mut ns = String::new();
                buffer.read_qname(&mut ns)?;

                Ok(DnsRecord::NS {
                    domain: domain,
                    host: ns,
                    ttl: ttl,
                })
            }
            QueryType::CNAME => {
                let mut cname = String::new();
                buffer.read_qname(&mut cname)?;

                Ok(DnsRecord::CNAME {
                    domain: domain,
                    host: cname,
                    ttl: ttl,
                })
            }
            QueryType::MX => {
                let priority = buffer.read_u16()?;
                let mut mx = String::new();
                buffer.read_qname(&mut mx)?;

                Ok(DnsRecord::MX {
                    domain: domain,
                    priority: priority,
                    host: mx,
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

    - (code déjà présenté, omis ici) -
}
```

C'est un peu indigeste, mais aucun de ces enregistrements n'est particulièrement
compliqué en soi ; c'est de les voir tous ensemble qui rend le code un peu
lourd.

> **N.d.T.** — Le principe est toujours le même : après le préambule commun, un
> `match` aiguille vers la lecture adaptée au type. Pour AAAA, on lit 16 octets
> sous forme de quatre blocs de 4 octets (`read_u32`), que l'on redécoupe en
> huit groupes de 2 octets : ce sont les huit groupes séparés par `:` dans
> l'écriture habituelle d'une adresse IPv6. Pour NS, CNAME et MX, les données
> contiennent un nom de domaine, que l'on lit avec `read_qname`, exactement
> comme le nom du préambule.

### Étendre BytePacketBuffer pour modifier des valeurs sur place

Avant de passer à l'écriture des enregistrements, nous devons ajouter deux
fonctions supplémentaires à `BytePacketBuffer` :

```rust
impl BytePacketBuffer {

    - (code déjà présenté, omis ici) -

    fn set(&mut self, pos: usize, val: u8) -> Result<()> {
        self.buf[pos] = val;

        Ok(())
    }

    fn set_u16(&mut self, pos: usize, val: u16) -> Result<()> {
        self.set(pos, (val >> 8) as u8)?;
        self.set(pos + 1, (val & 0xFF) as u8)?;

        Ok(())
    }

}
```

Lorsque nous écrivons les labels d'un enregistrement, nous ne connaissons pas à
l'avance le nombre d'octets nécessaires, puisque nous pourrions être amenés à
utiliser des sauts pour réduire la taille. Nous résolvons ce problème en
écrivant d'abord une taille nulle, puis en revenant en arrière pour y inscrire
la taille réelle.

> **N.d.T.** — C'est la technique du sommaire dont on laisse les numéros de page
> en blanc, pour les compléter une fois le document terminé : on note la
> position de l'emplacement réservé (`pos`), on y écrit provisoirement 0, on
> écrit les données, puis `set_u16` revient à cette position pour y inscrire la
> longueur réellement écrite. Contrairement à `write`, les fonctions `set` ne
> déplacent pas le curseur.

### Étendre DnsRecord pour écrire les nouveaux types d'enregistrements

Nous pouvons maintenant compléter `DnsRecord::write`. Voici notre nouvelle
fonction :

```rust
impl DnsRecord {

    - (code déjà présenté, omis ici) -

    pub fn write(&self, buffer: &mut BytePacketBuffer) -> Result<usize> {
        let start_pos = buffer.pos();

        match *self {
            DnsRecord::A {
                ref domain,
                ref addr,
                ttl,
            } => {
                buffer.write_qname(domain)?;
                buffer.write_u16(QueryType::A.to_num())?;
                buffer.write_u16(1)?;
                buffer.write_u32(ttl)?;
                buffer.write_u16(4)?;

                let octets = addr.octets();
                buffer.write_u8(octets[0])?;
                buffer.write_u8(octets[1])?;
                buffer.write_u8(octets[2])?;
                buffer.write_u8(octets[3])?;
            }
            DnsRecord::NS {
                ref domain,
                ref host,
                ttl,
            } => {
                buffer.write_qname(domain)?;
                buffer.write_u16(QueryType::NS.to_num())?;
                buffer.write_u16(1)?;
                buffer.write_u32(ttl)?;

                let pos = buffer.pos();
                buffer.write_u16(0)?;

                buffer.write_qname(host)?;

                let size = buffer.pos() - (pos + 2);
                buffer.set_u16(pos, size as u16)?;
            }
            DnsRecord::CNAME {
                ref domain,
                ref host,
                ttl,
            } => {
                buffer.write_qname(domain)?;
                buffer.write_u16(QueryType::CNAME.to_num())?;
                buffer.write_u16(1)?;
                buffer.write_u32(ttl)?;

                let pos = buffer.pos();
                buffer.write_u16(0)?;

                buffer.write_qname(host)?;

                let size = buffer.pos() - (pos + 2);
                buffer.set_u16(pos, size as u16)?;
            }
            DnsRecord::MX {
                ref domain,
                priority,
                ref host,
                ttl,
            } => {
                buffer.write_qname(domain)?;
                buffer.write_u16(QueryType::MX.to_num())?;
                buffer.write_u16(1)?;
                buffer.write_u32(ttl)?;

                let pos = buffer.pos();
                buffer.write_u16(0)?;

                buffer.write_u16(priority)?;
                buffer.write_qname(host)?;

                let size = buffer.pos() - (pos + 2);
                buffer.set_u16(pos, size as u16)?;
            }
            DnsRecord::AAAA {
                ref domain,
                ref addr,
                ttl,
            } => {
                buffer.write_qname(domain)?;
                buffer.write_u16(QueryType::AAAA.to_num())?;
                buffer.write_u16(1)?;
                buffer.write_u32(ttl)?;
                buffer.write_u16(16)?;

                for octet in &addr.segments() {
                    buffer.write_u16(*octet)?;
                }
            }
            DnsRecord::UNKNOWN { .. } => {
                println!("Skipping record: {:?}", self);
            }
        }

        Ok(buffer.pos() - start_pos)
    }
}
```

Encore pas mal de code supplémentaire, mais c'est heureusement la dernière chose
à faire. Nous n'utilisons toujours pas la partie écriture, mais elle nous sera
utile lorsque nous écrirons notre serveur.

### Tester les nouveaux types d'enregistrements

Nous sommes prêts à relancer notre requête sur « yahoo.com » :

```text
DnsHeader {
    id: 6666,
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
    answers: 3,
    authoritative_entries: 0,
    resource_entries: 0
}
DnsQuestion {
    name: "www.yahoo.com",
    qtype: A
}
CNAME {
    domain: "www.yahoo.com",
    host: "fd-fp3.wg1.b.yahoo.com",
    ttl: 3
}
A {
    domain: "fd-fp3.wg1.b.yahoo.com",
    addr: 46.228.47.115,
    ttl: 19
}
A {
    domain: "fd-fp3.wg1.b.yahoo.com",
    addr: 46.228.47.114,
    ttl: 19
}
```

> **N.d.T.** — L'enregistrement inconnu de tout à l'heure est désormais reconnu
> comme un CNAME : `www.yahoo.com` est un alias de `fd-fp3.wg1.b.yahoo.com`,
> qui possède deux adresses IP. Avoir plusieurs adresses pour un même nom
> permet de répartir la charge entre plusieurs serveurs.

Pour faire bonne mesure, essayons aussi une recherche de type MX :

```rust
let qname = "yahoo.com";
let qtype = QueryType::MX;
```

Ce qui donne :

```text
- (coupé) -
DnsQuestion {
    name: "yahoo.com",
    qtype: MX
}
MX {
    domain: "yahoo.com",
    priority: 1,
    host: "mta6.am0.yahoodns.net",
    ttl: 1794
}
MX {
    domain: "yahoo.com",
    priority: 1,
    host: "mta7.am0.yahoodns.net",
    ttl: 1794
}
MX {
    domain: "yahoo.com",
    priority: 1,
    host: "mta5.am0.yahoodns.net",
    ttl: 1794
}
```

> **Note du traducteur — Lancer cet exemple**
>
> Le programme complet est dans `examples/sample3.rs` ; il effectue directement
> la recherche MX sur `yahoo.com` : `cargo run --example sample3`. Il utilise un
> affichage compact (`{:?}`, une ligne par élément) au lieu de l'affichage
> détaillé (`{:#?}`) montré ci-dessus. Pour tester d'autres noms ou d'autres
> types, modifiez les lignes `qname` et `qtype` au début de la fonction
> `main` ; vous pouvez par exemple essayer le nom de domaine de votre propre
> site, avec le type A, puis MX.

Prochaine étape : [Chapitre 4 - Mon premier serveur DNS](chapter4.md)
