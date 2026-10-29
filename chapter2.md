2 - Construire un résolveur minimal (*stub resolver*)
====================================================

S'il est assez satisfaisant de savoir que nous sommes capables de décoder des
paquets DNS, se contenter de les lire depuis un fichier n'a pas grand intérêt.
Prochaine étape : nous allons nous en servir pour construire un `stub resolver`
(résolveur minimal), c'est-à-dire un client DNS qui ne sait pas effectuer
lui-même de recherche récursive et qui ne fonctionne qu'avec un serveur DNS qui,
lui, en est capable. Plus tard, nous implémenterons un véritable résolveur
récursif pour nous passer de ce serveur.

> **Note du traducteur — Qu'est-ce qu'un résolveur minimal ?**
>
> Un **résolveur** est le programme qui « résout » un nom, c'est-à-dire qui
> trouve l'adresse IP correspondante. Le résolveur *minimal* ne fait pas la
> recherche lui-même : il pose la question à un serveur DNS plus complet (ici
> celui de Google, `8.8.8.8`) et se contente de lire la réponse. C'est ce que
> fait votre ordinateur au quotidien : il délègue la recherche au serveur DNS
> de votre box ou de votre fournisseur d'accès, comme on demande un
> renseignement à un guichet plutôt que de fouiller soi-même les archives.
>
> Concrètement, ce chapitre ajoute la fonction inverse de la lecture :
> l'**écriture** d'un paquet, pour pouvoir envoyer nos propres questions.

### Étendre BytePacketBuffer pour l'écriture

Pour pouvoir traiter une requête, nous devons être capables non seulement de
lire des paquets, mais aussi d'en écrire. Pour cela, il nous faut ajouter
quelques méthodes à `BytePacketBuffer` :

> **N.d.T.** — Dans les extraits de code, la mention
> `- (code déjà présenté, omis ici) -` n'est pas du code : elle remplace les
> parties vues au chapitre précédent, pour ne montrer que les ajouts. Le code
> complet de chaque chapitre se trouve dans le dossier `examples/`.

```rust
impl BytePacketBuffer {

    - (code déjà présenté, omis ici) -

    fn write(&mut self, val: u8) -> Result<()> {
        if self.pos >= 512 {
            return Err("End of buffer".into());
        }
        self.buf[self.pos] = val;
        self.pos += 1;
        Ok(())
    }

    fn write_u8(&mut self, val: u8) -> Result<()> {
        self.write(val)?;

        Ok(())
    }

    fn write_u16(&mut self, val: u16) -> Result<()> {
        self.write((val >> 8) as u8)?;
        self.write((val & 0xFF) as u8)?;

        Ok(())
    }

    fn write_u32(&mut self, val: u32) -> Result<()> {
        self.write(((val >> 24) & 0xFF) as u8)?;
        self.write(((val >> 16) & 0xFF) as u8)?;
        self.write(((val >> 8) & 0xFF) as u8)?;
        self.write(((val >> 0) & 0xFF) as u8)?;

        Ok(())
    }
```

> **N.d.T.** — Ces fonctions font l'inverse de leurs équivalents de lecture :
> `write` écrit un octet à la position courante puis avance le curseur ;
> `write_u16` et `write_u32` découpent un nombre de 2 ou 4 octets en octets
> successifs, en commençant par l'octet de poids fort (`>>` décale vers la
> droite, `& 0xFF` ne garde que le dernier octet).

Il nous faut aussi une fonction pour écrire les noms demandés sous forme de
labels :

```rust
    fn write_qname(&mut self, qname: &str) -> Result<()> {
        for label in qname.split('.') {
            let len = label.len();
            if len > 0x3f {
                return Err("Single label exceeds 63 characters of length".into());
            }

            self.write_u8(len as u8)?;
            for b in label.as_bytes() {
                self.write_u8(*b)?;
            }
        }

        self.write_u8(0)?;

        Ok(())
    }

} // Fin de BytePacketBuffer
```

> **N.d.T.** — `qname.split('.')` découpe le nom à chaque point (`google.com`
> devient `google` et `com`). Pour chaque label, on écrit sa longueur puis ses
> caractères, et on termine par un octet nul : on obtient bien
> `[6]google[3]com[0]`. La limite de `0x3f` (63 en décimal) correspond à la
> longueur maximale d'un label autorisée par la norme. Remarque : cette version
> n'utilise pas la compression par sauts vue au chapitre 1 ; les paquets
> produits sont donc un peu plus longs, mais restent parfaitement valides.

### Étendre DnsHeader pour l'écriture

En nous appuyant sur ces nouvelles fonctions, nous pouvons compléter les
structures qui représentent notre protocole. Commençons par `DnsHeader` :

```rust
impl DnsHeader {

    - (code déjà présenté, omis ici) -

    pub fn write(&self, buffer: &mut BytePacketBuffer) -> Result<()> {
        buffer.write_u16(self.id)?;

        buffer.write_u8(
            (self.recursion_desired as u8)
                | ((self.truncated_message as u8) << 1)
                | ((self.authoritative_answer as u8) << 2)
                | (self.opcode << 3)
                | ((self.response as u8) << 7) as u8,
        )?;

        buffer.write_u8(
            (self.rescode as u8)
                | ((self.checking_disabled as u8) << 4)
                | ((self.authed_data as u8) << 5)
                | ((self.z as u8) << 6)
                | ((self.recursion_available as u8) << 7),
        )?;

        buffer.write_u16(self.questions)?;
        buffer.write_u16(self.answers)?;
        buffer.write_u16(self.authoritative_entries)?;
        buffer.write_u16(self.resource_entries)?;

        Ok(())
    }

}
```

> **N.d.T.** — C'est l'opération inverse de la lecture vue au chapitre 1 :
> chaque drapeau (`true`/`false`, converti en 1/0 par `as u8`) est décalé à sa
> place (`<< 1`, `<< 2`…) puis tous sont assemblés avec `|` pour reformer les
> deux octets de drapeaux.

### Étendre DnsQuestion pour l'écriture

Passons à `DnsQuestion` :

```rust
impl DnsQuestion {

    - (code déjà présenté, omis ici) -

    pub fn write(&self, buffer: &mut BytePacketBuffer) -> Result<()> {
        buffer.write_qname(&self.name)?;

        let typenum = self.qtype.to_num();
        buffer.write_u16(typenum)?;
        buffer.write_u16(1)?;

        Ok(())
    }

}
```

> **N.d.T.** — `buffer.write_u16(1)` écrit la classe, toujours égale à 1
> (`IN`, Internet).

### Étendre DnsRecord pour l'écriture

`DnsRecord` reste pour l'instant assez compact lui aussi, même si nous finirons
par y ajouter pas mal de code pour gérer les différents types
d'enregistrements :

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
            DnsRecord::UNKNOWN { .. } => {
                println!("Skipping record: {:?}", self);
            }
        }

        Ok(buffer.pos() - start_pos)
    }

}
```

> **N.d.T.** — Pour un enregistrement A, on écrit le préambule (nom, type,
> classe 1, TTL, longueur 4) puis les 4 octets de l'adresse IP (`octets()`
> renvoie les quatre nombres de l'adresse). Les enregistrements de type inconnu
> ne peuvent pas être réécrits : ils sont ignorés, avec un message
> (*Skipping record* = « enregistrement ignoré »). La fonction renvoie le
> nombre d'octets écrits (position finale moins position de départ).

### Étendre DnsPacket pour l'écriture

Assemblons le tout dans `DnsPacket` :

```rust
impl DnsPacket {

    - (code déjà présenté, omis ici) -

    pub fn write(&mut self, buffer: &mut BytePacketBuffer) -> Result<()> {
        self.header.questions = self.questions.len() as u16;
        self.header.answers = self.answers.len() as u16;
        self.header.authoritative_entries = self.authorities.len() as u16;
        self.header.resource_entries = self.resources.len() as u16;

        self.header.write(buffer)?;

        for question in &self.questions {
            question.write(buffer)?;
        }
        for rec in &self.answers {
            rec.write(buffer)?;
        }
        for rec in &self.authorities {
            rec.write(buffer)?;
        }
        for rec in &self.resources {
            rec.write(buffer)?;
        }

        Ok(())
    }

}
```

> **N.d.T.** — Avant d'écrire l'en-tête, on met à jour ses quatre compteurs à
> partir de la longueur réelle des listes (`len()`), pour garantir que
> l'en-tête annonce exactement ce que contient le paquet. On écrit ensuite les
> sections dans l'ordre imposé par le format.

### Implémenter un résolveur minimal

Nous sommes prêts à implémenter notre résolveur minimal. Rust fournit un
`UDPSocket` bien pratique, qui fait l'essentiel du travail.

> **N.d.T.** — Un **socket** est le « point de branchement » d'un programme sur
> le réseau : on l'ouvre sur un port local (ici 43210, choisi arbitrairement),
> puis on s'en sert pour envoyer (`send_to`) et recevoir (`recv_from`) des
> paquets. `0.0.0.0` signifie « sur toutes les interfaces réseau de cette
> machine ».

```rust
fn main() -> Result<()> {
    // Effectue une requête de type A pour google.com
    let qname = "google.com";
    let qtype = QueryType::A;

    // On utilise le serveur DNS public de Google
    let server = ("8.8.8.8", 53);

    // Ouvre un socket UDP sur un port choisi arbitrairement
    let socket = UdpSocket::bind(("0.0.0.0", 43210))?;

    // Construit notre paquet de requête. Il ne faut pas oublier d'activer le
    // drapeau `recursion_desired` (récursion souhaitée). Comme indiqué plus
    // haut, l'identifiant du paquet est arbitraire.
    let mut packet = DnsPacket::new();

    packet.header.id = 6666;
    packet.header.questions = 1;
    packet.header.recursion_desired = true;
    packet
        .questions
        .push(DnsQuestion::new(qname.to_string(), qtype));

    // On utilise notre nouvelle méthode write pour écrire le paquet dans un
    // tampon...
    let mut req_buffer = BytePacketBuffer::new();
    packet.write(&mut req_buffer)?;

    // ...puis on l'envoie au serveur via notre socket :
    socket.send_to(&req_buffer.buf[0..req_buffer.pos], server)?;

    // Pour préparer la réception de la réponse, on crée un nouveau
    // `BytePacketBuffer` et on demande au socket d'écrire la réponse
    // directement dans ce tampon.
    let mut res_buffer = BytePacketBuffer::new();
    socket.recv_from(&mut res_buffer.buf)?;

    // Comme au chapitre précédent, `DnsPacket::from_buffer()` sert ensuite à
    // décoder le paquet, après quoi on peut afficher la réponse.
    let res_packet = DnsPacket::from_buffer(&mut res_buffer)?;
    println!("{:#?}", res_packet.header);

    for q in res_packet.questions {
        println!("{:#?}", q);
    }
    for rec in res_packet.answers {
        println!("{:#?}", rec);
    }
    for rec in res_packet.authorities {
        println!("{:#?}", rec);
    }
    for rec in res_packet.resources {
        println!("{:#?}", rec);
    }

    Ok(())
}
```

> **N.d.T.** — `&req_buffer.buf[0..req_buffer.pos]` désigne uniquement les
> octets effectivement écrits (de la position 0 jusqu'au curseur), et non les
> 512 octets du tampon. Le nombre `6666` est l'identifiant du paquet, choisi
> arbitrairement : la réponse du serveur reprendra ce même identifiant.

L'exécution affiche :

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
    addr: 216.58.209.110,
    ttl: 79
}
```

> **Note du traducteur — Lancer cet exemple**
>
> Le programme complet est dans `examples/sample2.rs` : lancez-le avec
> `cargo run --example sample2`. Il a besoin d'un accès à Internet. Certains
> réseaux d'entreprise bloquent les requêtes DNS vers des serveurs extérieurs
> (port 53 en UDP) : si le programme semble attendre indéfiniment, c'est
> probablement la cause, et il faudra l'essayer depuis une autre connexion.
> L'adresse IP et le TTL obtenus seront différents de ceux affichés ci-dessus.

Le chapitre suivant ajoute la prise en charge d'un éventail plus riche de types
d'enregistrements : [Chapitre 3 - Ajouter d'autres types d'enregistrements](chapter3.md)
