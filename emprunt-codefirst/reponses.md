# TP1 — SOAP et WSDL · Réponses
**Module** : Architecture SOA & Web Services  
**Institut** : SupNum · Licence Informatique · 2026-2027  
**Binôme** : Nom1 / Nom2

---

## Partie A — Code-first

### Étape 1 — Questions

**Q1 : Quelle annotation fait de `EmpruntService` un service web ? Quelle annotation fait de `LivreIndisponibleException` une erreur déclarée dans le contrat ?**

- `@WebService` (sur la classe `EmpruntService`) déclare la classe comme service web SOAP : JAXWS génère alors automatiquement le WSDL à partir des méthodes publiques annotées `@WebMethod`.  
- `@WebFault(name = "LivreIndisponible", targetNamespace = "http://supnum.mr/bib")` (sur `LivreIndisponibleException`) signale à JAXWS que cette exception doit apparaître dans le WSDL comme un message de Fault nommé `LivreIndisponible`, avec son propre élément `<detail>` typé.

**Q2 : Pourquoi les classes de données doivent-elles avoir un constructeur vide ?**

JAXB (le moteur de sérialisation/désérialisation XML utilisé par JAXWS) instancie les objets par réflexion, puis remplit les champs via leurs setters ou en accès direct. Sans constructeur sans argument, la réflexion Java ne peut pas créer l'instance (`InstantiationException`) et la désérialisation du message SOAP échoue.

---

### Étape 2 — Lire le WSDL généré

**Q1 : Quelle est l'URL du service ? Quelles sont les deux opérations et leurs messages d'entrée et de sortie ?**

- URL du service : `http://localhost:8080/bib/EmpruntService`  
- Opération **`emprunterLivre`** :
  - Entrée : `emprunterLivreRequest` (idAdherent + isbn)  
  - Sortie : `emprunterLivreResponse` (emprunt)  
  - Fault : `LivreIndisponible`  
- Opération **`rendreLivre`** :
  - Entrée : `rendreLivreRequest` (numeroEmprunt)  
  - Sortie : `rendreLivreResponse` (penalite)

**Q2 : Quel style et quel `use` le binding déclare-t-il ? Est-ce du SOAP 1.1 ou 1.2 ?**

- Style : **`document`**, Use : **`literal`** → c'est le style recommandé WS-I BP 1.1.  
- SOAP **1.1** (namespace `http://schemas.xmlsoap.org/soap/envelope/`).

**Q3 : Quel type XSD a été généré pour `isbn` et pour `dateRetour` ? La règle de format de l'ISBN apparaît-elle ?**

- `isbn` → `xs:string` : JAXWS en code-first ne connaît pas le `Pattern` Java ; la contrainte regex **n'apparaît pas** dans le WSDL généré.  
- `dateRetour` → `xs:string` (car le champ Java est de type `String`) ; aucun format de date n'est imposé par le XSD.

**Q4 : Dans la requête SoapUI, les paramètres `idAdherent` et `isbn` portent-ils un préfixe d'espace de noms ?**

Oui. Ils sont qualifiés avec le namespace `http://supnum.mr/bib` (style document/literal, `elementFormDefault="qualified"` implicite), donc ils portent le préfixe `bib:` (ou sont dans le namespace par défaut `xmlns="http://supnum.mr/bib"`).

**Q5 : Dans l'onglet Raw, où voyez-vous HTTP, XML, SOAP ? Quel rôle joue chacun ?**

| Couche | Rôle |
|--------|------|
| **HTTP** | Protocole de transport : `POST /bib/EmpruntService`, en-têtes `Content-Type: text/xml`, `SOAPAction`. |
| **XML** | Langage de sérialisation : structure l'enveloppe et le corps du message. |
| **SOAP** | Protocole de messagerie : définit `<Envelope>`, `<Header>`, `<Body>`, `<Fault>`. |
| **WSDL** | Contrat du service : décrit les opérations, les types, le binding et l'adresse — c'est la « carte d'identité » du service. |

---

### Étape 3 — Envoyer des messages SOAP

**Q1 : Pourquoi la requête utilise-t-elle `POST`, même pour lire une information ?**

SOAP est un protocole de messagerie orienté **action**, non orienté ressource. Toutes les opérations (lecture ou écriture) sont envoyées en `POST` car le message complet (operation + paramètres) se trouve dans le corps HTTP. Le verbe `POST` est le seul qui autorise un corps de requête sans restriction de taille ni de mise en cache.

**Q2 : Le bloc `jeton` du Header a-t-il empêché le traitement ? Pourquoi ?**

Non. Sans l'attribut `soap:mustUnderstand="1"`, le serveur **ignore** les blocs de Header qu'il ne reconnaît pas. Le traitement du `Body` se déroule normalement.

**Q3 : Que répond le service au message SOAP 1.2 ? Expliquez avec le binding.**

Le service répond avec un Fault SOAP 1.1 du type `VersionMismatch` (ou rejette la requête), car le binding déclare SOAP 1.1 (`xmlns:soap="http://schemas.xmlsoap.org/wsdl/soap/"`). Le serveur ne comprend pas l'espace de noms SOAP 1.2 (`http://www.w3.org/2003/05/soap-envelope`) et retourne une erreur.

**Q4 : Listez les cinq erreurs du message malformé et leur correction.**

| # | Erreur | Correction | Côté serveur ? |
|---|--------|------------|----------------|
| 1 | `<soap:envelope>` en minuscule | `<soap:Envelope>` (XML sensible à la casse) | **Oui** — parse error |
| 2 | Namespace `http://supnum.mr/bibliotheque` incorrect | `http://supnum.mr/bib` | **Oui** — opération inconnue |
| 3 | `<bib:numero>` au lieu de `<bib:numeroEmprunt>` | `<bib:numeroEmprunt>` | **Oui** — paramètre absent |
| 4 | `</rendreLivre>` sans préfixe (mauvais fermant) | `</bib:rendreLivre>` | **Oui** — parse error |
| 5 | `<soap:Header>` après `</soap:Body>` | Déplacer le Header avant le Body | **Non** — certains parseurs tolèrent l'ordre (mais c'est invalide SOAP) |

---

### Étape 4 — Provoquer et construire des Faults

**Q1 : Pour les cas 1 et 2, quels `faultcode` et `faultstring` recevez-vous, et quel code HTTP ?**

- Cas 1 (ISBN `978-2-10`, format invalide) :  
  - `faultcode` : `soap:Client`  
  - `faultstring` : `ISBN mal formé : 978-2-10`  
  - Code HTTP : **500** (SOAP Fault → toujours 500 en SOAP 1.1)

- Cas 2 (ISBN `978-2-10-074444-4`, livre indisponible) :  
  - `faultcode` : `soap:Server` (car JAXWS choisit Server pour les exceptions métier)  
  - `faultstring` : `Livre indisponible`  
  - Code HTTP : **500**

**Q2 : Dans le cas 2, que contient `<detail>` ? Où sa structure est-elle décrite dans le WSDL ?**

`<detail>` contient un élément `<LivreIndisponible>` avec `<isbn>` et `<retourPrevu>`. Sa structure est décrite dans la section `<wsdl:fault name="LivreIndisponible">` du portType et du binding, qui référence le message `LivreIndisponibleFault`, lequel pointe vers le type `LivreIndisponibleInfo`.

**Q3 : Quel code JAXWS a-t-il choisi pour le livre indisponible ? Est-ce une erreur du client ou du serveur ? Qu'auriez-vous choisi ?**

JAXWS choisit `soap:Server`. Or, un livre indisponible est **une condition métier prévisible** : le client a envoyé un ISBN valide, mais le livre est actuellement sorti. C'est davantage une réponse métier qu'une faute serveur. On aurait pu choisir `soap:Client` pour signaler que la demande ne peut pas être satisfaite dans l'état actuel, ou créer un Fault spécifique `LivreIndisponible` (ce que fait correctement l'approche contract-first avec `@WebFault`).

**Q4 : Que se passe-t-il avec `mustUnderstand="1"` ? Et avec `"0"` ?**

- `mustUnderstand="1"` : le serveur DOIT traiter le bloc Header. S'il ne le comprend pas, il retourne un Fault `soap:MustUnderstand` et **n'exécute pas** l'opération.  
- `mustUnderstand="0"` (ou absent) : le serveur peut ignorer le bloc Header inconnu ; l'opération se déroule normalement.

**Q5 : Quelle différence voyez-vous entre votre Fault 1.2 et les Faults 1.1 reçues ?**

| Aspect | SOAP 1.1 | SOAP 1.2 |
|--------|----------|----------|
| Élément code | `<faultcode>soap:Client</faultcode>` | `<soap:Code><soap:Value>soap:Sender</soap:Value></soap:Code>` |
| Élément message | `<faultstring>...</faultstring>` | `<soap:Reason><soap:Text xml:lang="fr">...</soap:Text></soap:Reason>` |
| Détail | `<detail>` | `<soap:Detail>` |
| Codes prédéfinis | `Client`, `Server`, `MustUnderstand`, `VersionMismatch` | `Sender`, `Receiver`, `MustUnderstand`, `VersionMismatch`, `DataEncodingUnknown` |

---

## Partie B — Contract-first

### Étape 5 — Questions

**Q1 : Dans ce WSDL, retrouvez l'opération, son message de fault et l'action SOAP à envoyer.**

- Opération : `emprunterLivre` (dans `EmpruntPortType`)  
- Message de fault : `LivreIndisponibleFault` (élément `bib:LivreIndisponible`)  
- SOAPAction : `http://supnum.mr/bib/emprunterLivre`

**Q2 : Quelle est la signature de `emprunterLivre` dans `EmpruntPortType` ?**

```java
EmpruntType emprunterLivre(String idAdherent, String isbn)
    throws LivreIndisponibleFault;
```

**Q3 : Quel type Java a été généré pour `dateRetour` ? Comparez avec la partie A.**

- Partie B (contract-first) : `XMLGregorianCalendar` — car le WSDL déclare `xs:date`, et `wsimport` mappe `xs:date` sur `XMLGregorianCalendar`.  
- Partie A (code-first) : `String` — car le champ Java était de type `String` ; aucune contrainte de date n'est exprimée dans le WSDL généré.  
→ La partie B est **plus stricte et plus interopérable**.

**Q4 : Quel est le nom de la classe d'exception générée, et d'où vient ce nom dans le WSDL ?**

La classe générée s'appelle **`LivreIndisponibleFault`**. Ce nom vient de l'attribut `name` du message de fault dans le WSDL : `<wsdl:message name="LivreIndisponibleFault">`.

---

### Étape 6 — Questions

**Q1 : Le WSDL publié est-il celui que vous avez fourni ? Contient-il le motif de l'ISBN et le type `xs:date` ?**

Oui. Grâce à `wsdlLocation = "wsdl/EmpruntService.wsdl"` dans `@WebService`, JAXWS sert exactement le fichier WSDL fourni. Il contient bien :
- le motif `97[89]-\d-\d{2,5}-\d{2,6}-\d` (type `IsbnType`)
- le type `xs:date` pour `dateRetour` et `retourPrevu`

**Q2 : Avez-vous dû modifier `01-emprunter.xml` pour qu'il fonctionne ici ? Pourquoi ?**

Non. Le message `01-emprunter.xml` a été construit conformément au WSDL contract-first (même namespace `http://supnum.mr/bib`, même opération `emprunterLivre`, mêmes paramètres). Il fonctionne directement, à condition d'ajouter l'en-tête `SOAPAction: "http://supnum.mr/bib/emprunterLivre"` requis par ce binding.

**Q3 : Sans `@SchemaValidation`, l'ISBN `978-2-10` est-il refusé ? Et avec ? Quelle Fault obtenez-vous ?**

- **Sans** `@SchemaValidation` : l'ISBN invalide passe la désérialisation (JAXWS ne valide pas le motif XSD par défaut) ; la validation applicative dans `EmpruntServiceImpl` n'est pas implémentée → le service retourne un emprunt fictif sans erreur.  
- **Avec** `@SchemaValidation` : JAXWS valide le message contre le schéma XSD intégré au WSDL. L'ISBN `978-2-10` ne respecte pas le motif `IsbnType` → Fault `soap:Client` avec `faultstring` indiquant la violation de schéma (p.ex. *"cvc-pattern-valid"*).

**Q4 : Retirez `wsdlLocation` de l'annotation et redémarrez : qu'est-ce qui change dans le WSDL publié ?**

Sans `wsdlLocation`, JAXWS **régénère le WSDL à partir du code Java** (comme en code-first). Le WSDL publié perd alors :
- le motif de validation `IsbnType` (remplacé par `xs:string`)
- le type `xs:date` pour `dateRetour` (remplacé par `xs:string`)  
Le contrat publié ne correspond plus au contrat original écrit à la main.

---

## Conclusion — Choisir et rendre

**Q1 : Citez deux différences observées entre le WSDL de la partie A et celui de la partie B.**

1. **Types de données** : en partie A, `isbn` et `dateRetour` sont des `xs:string` sans contrainte ; en partie B, `isbn` est un `IsbnType` avec motif regex et `dateRetour` est un `xs:date` strictement typé.  
2. **Origine du WSDL** : en partie A, le WSDL est **généré** par JAXWS à partir des annotations Java (il peut changer si le code change) ; en partie B, le WSDL est **fourni** et figé — c'est le vrai contrat partagé.

**Q2 : Si on renomme le champ `dateRetour` dans le code, quel projet voit son contrat changer ? Quel risque pour les clients ?**

- **Partie A (code-first)** : le WSDL est régénéré → le champ change de nom dans le WSDL → tous les clients existants qui utilisaient le nom `dateRetour` **cassent** sans préavis.  
- **Partie B (contract-first)** : le WSDL reste inchangé (il n'est pas régénéré) → les clients ne voient rien ; seule l'implémentation interne change. Le risque est nul pour les clients.

**Q3 : Quelle approche choisiriez-vous pour le service Paiement ? Et pour un prototype interne de deux jours ?**

- **Service Paiement (multi-équipes, multi-langages : .NET, PHP, Node.js)** → **contract-first**. Le WSDL est le contrat unique, figé et partagé entre toutes les équipes dès le départ. Chaque équipe peut générer son stub indépendamment. Toute modification du contrat est volontaire et versionnée.  
- **Prototype interne de deux jours** → **code-first**. On va vite, on n'a qu'un seul consommateur, et le WSDL sera figé (ou mis à la poubelle) avant d'aller en production.

---

*Rendu : `TP1_Nom1_Nom2.zip` contenant `emprunt-codefirst/`, `emprunt-contractfirst/`, `xml/` et `reponses.md`.*
