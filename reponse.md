# TP1 — SOAP et WSDL
## Réponses aux questions

Ce document présente nos réponses aux questions du TP. Nous avons réalisé les deux approches demandées : code-first et contract-first, puis nous avons comparé les contrats WSDL.

**Institut :** Institut Supérieur du Numérique (SupNum)  
**Formation :** Licence Informatique  
**Module :** Architecture SOA & Web Services  
**Année universitaire :** 2026–2027  
**Travail :** TP n° 1 — SOAP et WSDL  
**Binôme :** [Nom et prénom de l’étudiant 1] / [Nom et prénom de l’étudiant 2]

---

## Partie A — Approche Code-first

Dans cette partie, nous avons commencé par écrire le service en Java. Ensuite, JAX-WS a généré le fichier WSDL à partir de la classe et de ses annotations.

### Étape 1 — Créer et lancer le service

**Question 1 — Quelle annotation fait de `EmpruntService` un service web ? Quelle annotation fait de `LivreIndisponibleException` une erreur déclarée dans le contrat ?**

- `@WebService`, placée sur la classe `EmpruntService`, déclare cette classe comme un service web SOAP. JAX-WS peut alors exposer ses opérations et générer le WSDL correspondant.
- `@WebFault(name = "LivreIndisponible", targetNamespace = "http://supnum.mr/bib")`, placée sur `LivreIndisponibleException`, indique que cette exception métier doit être représentée dans le contrat SOAP comme une Fault nommée `LivreIndisponible`, avec des informations structurées dans l’élément `<detail>`.

**Question 2 — Pourquoi les classes de données doivent-elles avoir un constructeur vide ?**

JAXB utilise le constructeur vide pour créer un objet Java lorsqu’il reçoit les données XML. Après la création, il remplit les propriétés de l’objet. Sans ce constructeur, la conversion du message SOAP en objet Java peut échouer.

### Étape 2 — Lire le WSDL généré

**Question 1 — Quelle est l’URL du service ? Quelles sont les deux opérations et leurs messages d’entrée et de sortie ?**

L’adresse que nous avons utilisée pour accéder au service est :

`http://localhost:8080/bib/EmpruntService`

Dans le WSDL, nous trouvons deux opérations :

| Opération | Message d’entrée | Message de sortie |
|---|---|---|
| `emprunterLivre` | `emprunterLivreRequest` : `idAdherent` et `isbn` | `emprunterLivreResponse` : un objet `emprunt` |
| `rendreLivre` | `rendreLivreRequest` : `numeroEmprunt` | `rendreLivreResponse` : `penalite` |

L’opération `emprunterLivre` peut également produire la Fault métier `LivreIndisponible`.

**Question 2 — Quel style et quel `use` le binding déclare-t-il ? Est-ce du SOAP 1.1 ou 1.2 ?**

Le binding utilise le style **`document`** et le mode **`literal`**. Il s’agit de **SOAP 1.1**, identifiable notamment par l’espace de noms de l’enveloppe :

`http://schemas.xmlsoap.org/soap/envelope/`

**Question 3 — Quel type XSD a été généré pour `isbn` et pour `dateRetour` ? La règle de format de l’ISBN apparaît-elle ?**

- `isbn` est de type `xs:string`. La règle de validation définie par le motif Java `Pattern` n’est pas automatiquement traduite en contrainte XSD dans cette approche.
- `dateRetour` est également de type `xs:string`, car l’attribut correspondant est déclaré comme une chaîne de caractères en Java.

Donc, dans notre WSDL code-first, le format de l’ISBN n’apparaît pas comme une contrainte XSD et `dateRetour` reste une chaîne de caractères.

**Question 4 — Dans la requête SoapUI, les paramètres `idAdherent` et `isbn` portent-ils un préfixe d’espace de noms ?**

Oui. Les éléments sont associés au namespace `http://supnum.mr/bib`. Ils peuvent être écrits avec le préfixe `bib:`, par exemple `bib:idAdherent` et `bib:isbn`, ou utiliser directement ce namespace comme espace de noms par défaut.

**Question 5 — Dans l’onglet Raw, où voyez-vous HTTP, XML et SOAP ? Quel rôle joue chacun ? Quel rôle joue le WSDL ?**

| Élément | Rôle |
|---|---|
| **HTTP** | Transporte la requête et la réponse entre le client et le serveur. On le voit notamment dans la méthode `POST`, l’URL et les en-têtes HTTP. |
| **XML** | Représente les données échangées sous forme d’éléments et d’attributs. |
| **SOAP** | Définit la structure du message, notamment `<Envelope>`, `<Header>`, `<Body>` et `<Fault>`. |
| **WSDL** | Décrit le contrat du service : ses opérations, les messages, les types, le binding et l’adresse du service. |

### Étape 3 — Envoyer des messages SOAP

**Question 1 — Pourquoi la requête utilise-t-elle `POST`, même pour lire une information ?**

Nous utilisons `POST` parce que la requête SOAP complète est envoyée dans le corps du message HTTP. Cela reste vrai même lorsque l’opération sert seulement à lire une information.

**Question 2 — Le bloc `jeton` du Header a-t-il empêché le traitement ? Pourquoi ?**

Non, dans le cas testé sans `soap:mustUnderstand="1"`. Un bloc d’en-tête que le serveur ne reconnaît pas peut être ignoré lorsque l’attribut `mustUnderstand` n’impose pas son traitement. Le serveur peut alors continuer à traiter le contenu du `Body`.

**Question 3 — Que répond le service au message SOAP 1.2 ? Expliquez avec le binding.**

Le binding du service déclare SOAP 1.1. Le message envoyé utilise, quant à lui, l’espace de noms de SOAP 1.2 :

`http://www.w3.org/2003/05/soap-envelope`

Il y a donc une incompatibilité entre la version SOAP attendue par le service et celle utilisée par la requête. Le serveur peut rejeter le message et retourner une Fault de type `VersionMismatch`.

**Question 4 — Listez les cinq erreurs du message malformé et leur correction. Certaines ne déclenchent aucune erreur côté serveur : lesquelles, et pourquoi ?**

| N° | Erreur dans le message | Correction |
|---:|---|---|
| 1 | `<soap:envelope>` commence par une minuscule. | Écrire `<soap:Envelope>`, car XML distingue les majuscules des minuscules. |
| 2 | Le namespace `http://supnum.mr/bibliotheque` est incorrect. | Utiliser `http://supnum.mr/bib`. |
| 3 | Le paramètre est nommé `bib:numero` au lieu de `bib:numeroEmprunt`. | Utiliser `bib:numeroEmprunt`. |
| 4 | La balise fermante `</rendreLivre>` ne correspond pas à la balise ouvrante préfixée. | Écrire `</bib:rendreLivre>`. |
| 5 | Le `Header` est placé après le `Body`. | Placer le `Header` avant le `Body`, à l’intérieur de l’`Envelope`. |

Les erreurs de structure XML (notamment la casse et les balises qui ne correspondent pas) empêchent l’analyse correcte du document. Le namespace et le nom du paramètre peuvent provoquer une erreur lors de la recherche de l’opération ou de la désérialisation. L’ordre `Header` puis `Body` fait partie de la structure SOAP attendue ; même si un parseur particulier semble tolérer un message incorrect, il faut respecter cet ordre.

### Étape 4 — Provoquer et construire des Faults

**Question 1 — Pour les cas 1 et 2, quels `faultcode` et `faultstring` recevons-nous, et quel code HTTP ?**

D’après les cas prévus dans le TP :

- **Cas 1 — ISBN `978-2-10` (format invalide)**
  - `faultcode` : `soap:Client`
  - `faultstring` : `ISBN mal formé : 978-2-10`
  - Code HTTP attendu : `500` pour une réponse SOAP 1.1 contenant une Fault.

- **Cas 2 — ISBN `978-2-10-074444-4` (livre indisponible)**
  - `faultcode` : `soap:Server`, selon le comportement indiqué pour l’exception métier dans cette implémentation.
  - `faultstring` : `Livre indisponible`
  - Code HTTP attendu : `500`.

**Question 2 — Dans le cas 2, que contient `<detail>` ? Où sa structure est-elle décrite dans le WSDL ?**

L’élément `<detail>` contient les informations associées à la Fault métier, notamment un élément `LivreIndisponible` avec les champs `isbn` et `retourPrevu`.

Dans le WSDL, cette structure est reliée au message de Fault `LivreIndisponibleFault`, déclaré dans la section `wsdl:message`, puis référencé par l’opération dans `wsdl:portType` et dans le binding.

**Question 3 — Quel code JAX-WS a-t-il choisi pour le livre indisponible ? Est-ce une erreur du client ou du serveur ? Qu’auriez-vous choisi ?**

Dans le cas décrit, JAX-WS renvoie `soap:Server`. Pourtant, un livre déjà emprunté représente une situation métier prévisible : la demande est correctement formée, mais le service ne peut pas satisfaire l’emprunt à cet instant.

Nous aurions préféré une Fault métier spécifique, clairement définie dans le contrat, afin que le client puisse distinguer cette situation d’une véritable erreur technique du serveur. Le contrat contract-first du TP prévoit justement une Fault nommée `LivreIndisponible`.

**Question 4 — Que se passe-t-il avec `mustUnderstand="1"` ? Et avec `"0"` ?**

- Avec `soap:mustUnderstand="1"`, le serveur doit comprendre et traiter le bloc d’en-tête. S’il ne le comprend pas, il renvoie une Fault `MustUnderstand` et ne poursuit pas normalement le traitement de l’opération.
- Avec `soap:mustUnderstand="0"` ou lorsque l’attribut est absent, le serveur peut ignorer un bloc d’en-tête qu’il ne comprend pas et continuer le traitement.

**Question 5 — Quelle différence voyez-vous entre votre Fault 1.2 et les Faults 1.1 reçues ?**

Nous remarquons que SOAP 1.1 et SOAP 1.2 n’utilisent pas les mêmes balises pour représenter une Fault.

| Élément | SOAP 1.1 | SOAP 1.2 |
|---|---|---|
| Code | `<faultcode>` | `<soap:Code><soap:Value>` |
| Message explicatif | `<faultstring>` | `<soap:Reason><soap:Text>` |
| Détail | `<detail>` | `<soap:Detail>` |
| Exemples de codes | `Client`, `Server`, `MustUnderstand`, `VersionMismatch` | `Sender`, `Receiver`, `MustUnderstand`, `VersionMismatch`, `DataEncodingUnknown` |

---

## Partie B — Approche Contract-first

Pour cette partie, nous avons commencé par le fichier WSDL. Ensuite, `wsimport` a généré les classes et l’interface Java que nous avons utilisées pour écrire l’implémentation du service.

### Étape 5 — Générer le code depuis le WSDL

**Question 1 — Dans ce WSDL, retrouvez l’opération, son message de Fault et l’action SOAP à envoyer.**

- Opération : `emprunterLivre`, déclarée dans `EmpruntPortType`.
- Message de Fault : `LivreIndisponibleFault`, associé à l’élément `bib:LivreIndisponible`.
- Valeur de `SOAPAction` : `http://supnum.mr/bib/emprunterLivre`.

**Question 2 — Quelle est la signature de `emprunterLivre` dans `EmpruntPortType` ?**

La signature générée est de la forme :

```java
EmpruntType emprunterLivre(String idAdherent, String isbn)
    throws LivreIndisponibleFault;
```

**Question 3 — Quel type Java a été généré pour `dateRetour` ? Comparez avec la partie A.**

- En **contract-first**, `dateRetour` est de type `XMLGregorianCalendar`, car le WSDL déclare le champ avec le type `xs:date`.
- En **code-first**, `dateRetour` est de type `String`, car le champ Java correspondant est une chaîne de caractères.

On voit donc que contract-first conserve le type défini dans le WSDL, alors qu’en code-first le champ était une simple chaîne.

**Question 4 — Quel est le nom de la classe d’exception générée, et d’où vient ce nom dans le WSDL ?**

Dans le projet, la classe générée est `LivreIndisponibleFault`. Elle correspond à la Fault déclarée dans le WSDL, notamment au message `LivreIndisponibleFault` et à la déclaration `wsdl:fault` de l’opération.

### Étape 6 — Implémenter et comparer

**Question 1 — Le WSDL publié est-il celui que vous avez fourni ? Contient-il le motif de l’ISBN et le type `xs:date` ?**

Oui, lorsque le service est publié avec `wsdlLocation = "wsdl/EmpruntService.wsdl"`, il utilise le contrat WSDL fourni. Celui-ci contient :

- le type `IsbnType`, avec le motif `97[89]-\d-\d{2,5}-\d{2,6}-\d` ;
- le type `xs:date` pour `dateRetour` et `retourPrevu`.

**Question 2 — Avez-vous dû modifier `01-emprunter.xml` pour qu’il fonctionne ici ? Pourquoi ?**

La structure du message XML reste conforme au même namespace et à la même opération. En revanche, pour cet appel contract-first, il faut envoyer l’en-tête HTTP `SOAPAction` demandé par le binding :

`SOAPAction: "http://supnum.mr/bib/emprunterLivre"`

Le message doit donc être envoyé avec les informations HTTP attendues par ce contrat.

**Question 3 — Sans `@SchemaValidation`, l’ISBN `978-2-10` est-il refusé ? Et avec ? Quelle Fault obtenez-vous ?**

- **Sans `@SchemaValidation`** : la validation du motif XSD n’est pas automatiquement appliquée au message entrant. Si aucune vérification équivalente n’est réalisée dans le code Java, l’ISBN invalide peut être accepté par l’implémentation.
- **Avec `@SchemaValidation`** : le message est validé par rapport au schéma associé au WSDL. L’ISBN `978-2-10` ne respecte pas le motif défini dans `IsbnType`. Le service renvoie alors une Fault liée à l’échec de validation du schéma, généralement identifiée comme une erreur côté émetteur (`soap:Client` en SOAP 1.1).

**Question 4 — Retirez `wsdlLocation` de l’annotation et redémarrez : qu’est-ce qui change dans le WSDL publié ?**

Sans `wsdlLocation`, JAX-WS peut reconstruire le WSDL à partir des annotations et des classes Java au lieu de publier le contrat fourni. Certaines contraintes propres au WSDL initial risquent alors de disparaître, notamment :

- le motif de validation de `IsbnType`, qui peut être remplacé par un simple `xs:string` ;
- le type `xs:date` de `dateRetour`, qui peut redevenir `xs:string` si le champ Java correspondant est une chaîne.

Dans ce cas, le WSDL publié peut être différent du fichier que nous avions fourni au départ.

---

## Conclusion — Choisir entre Code-first et Contract-first

**Question 1 — Citez deux différences observées entre le WSDL de la partie A et celui de la partie B.**

1. **Les types et les contraintes :** en code-first, `isbn` et `dateRetour` sont décrits comme des `xs:string` sans les contraintes attendues. En contract-first, le WSDL définit `IsbnType` avec un motif de validation et utilise `xs:date` pour les dates.
2. **L’origine du contrat :** en code-first, le WSDL est généré à partir du code Java. En contract-first, le WSDL est rédigé en premier et sert de référence à la génération des classes Java.

**Question 2 — Si on renomme le champ `dateRetour` dans le code, quel projet voit son contrat changer ? Quel risque pour les clients ?**

- **Code-first :** le WSDL étant généré à partir du code, le changement de nom peut modifier le contrat publié. Les clients qui utilisent l’ancien nom peuvent alors ne plus réussir à échanger correctement avec le service.
- **Contract-first :** le WSDL fourni reste la référence et ne change pas simplement parce qu’un nom interne est modifié dans l’implémentation. Il faut toutefois conserver la correspondance entre l’implémentation et le contrat.

**Question 3 — Quelle approche choisiriez-vous pour le service Paiement, appelé par Scolarité (.NET), Bibliothèque (PHP) et le portail (Node.js) ? Et pour un prototype interne de deux jours ?**

- **Pour le service Paiement : contract-first.** Plusieurs équipes utilisant des langages différents doivent partager un contrat commun et stable. Le WSDL permet à chaque équipe de connaître les opérations, les types et les messages attendus, puis de générer ses propres classes clientes.
- **Pour un prototype interne de deux jours : code-first.** Cette approche permet de démarrer rapidement en écrivant directement le service Java, puis en laissant JAX-WS générer le WSDL. Elle est adaptée à un prototype avec peu de consommateurs et des besoins encore limités.

---

## Éléments du rendu

Pour le rendu, nous devons mettre dans l’archive `TP1_Nom1_Nom2.zip` :

- le projet `emprunt-codefirst/` ;
- le projet `emprunt-contractfirst/` ;
- le dossier `xml/` ;
- le présent fichier `reponse.md`.

**À compléter avant le dépôt :** écrire les noms et prénoms des deux membres du binôme en haut du document.
