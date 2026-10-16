Voici un contenu propre, professionnel et totalement générique, prêt à être copié-collé dans le `README.md` de ton dépôt Git pour documenter ces questions techniques Angular & RxJS.

---

### 1. Gestion des erreurs et résilience d'un flux (`retry` & `catchError`)

**La question :**

> Comment intercepter une erreur HTTP dans un Observable pour relencher automatiquement la requête plusieurs fois avec un délai, tout en prévoyant une valeur de secours si l'erreur persiste sans casser le flux ?

**La réponse type :**
Par défaut, lorsqu'un Observable émet une erreur, il est définitivement fermé (terminé) et ne peut plus émettre de nouvelles valeurs. Pour gérer la résilience, on utilise des opérateurs spécifiques dans le `pipe()`, en respectant un **ordre strict** :

1. **`retry`** : Permet de tenter à nouveau l'opération un certain nombre de fois avec un délai optionnel.
2. **`catchError`** : Intercepte l'erreur finale si tous les essais ont échoué et permet de retourner un flux de secours (fallback) pour éviter de casser l'abonnement global.

```typescript
import { HttpClient } from '@angular/common/http';
import { retry, catchError, of } from 'rxjs';

this.http.get<User[]>('/api/users').pipe(
  // 1. Tente 3 essais maximum avec 1 seconde de pause entre chaque tentative
  retry({ count: 3, delay: 1000 }),
  
  // 2. Si l'erreur persiste, catchError intercepte et renvoie une valeur par défaut (ex: tableau vide)
  catchError((error) => {
    console.error('Échec de la requête après plusieurs tentatives', error);
    return of([]);
  })
).subscribe(users => {
  // Le flux reste actif et traite les données de secours si besoin
});

```

* **Piège à éviter :** L'ordre des opérateurs est vital. Si `catchError` est placé *avant* `retry`, il intercepte l'erreur immédiatement et l'opérateur `retry` ne se déclenchera jamais.

---

### 2. Différences entre `Subject`, `BehaviorSubject` et `ReplaySubject`

**La question :**

> Quelles sont les différences fondamentales entre un `Subject`, un `BehaviorSubject` et un `ReplaySubject` dans la gestion d'état ?

**La réponse type :**
Ces trois classes héritent de `Subject` et agissent à la fois comme des `Observable` et des `Observer`, mais leur gestion de la mémoire et de l'historique diffère :

* **`Subject` (Sans mémoire) :**
C'est un simple émetteur d'événements. Si un composant s'abonne *après* qu'une valeur a été émise, il ne la recevra jamais.
* *Cas d'usage :* Événements ponctuels (clics, notifications globales, déclencheurs d'actions).


* **`BehaviorSubject` (Mémoire de la dernière valeur) :**
Il exige obligatoirement une valeur initiale à sa création et **conserve en mémoire la dernière valeur émise**. Tout nouvel abonné reçoit immédiatement cette dernière valeur, même s'il arrive après l'émission. Il permet aussi d'accéder à sa valeur de manière synchrone via `.getValue()`.
* *Cas d'usage :* Gestion d'état local (Store, données utilisateur, thème courant).


* **`ReplaySubject` (Mémoire de l'historique) :**
Il ne nécessite pas de valeur initiale, mais permet de définir un **tampon (*buffer*)**. Par exemple, `new ReplaySubject(3)` stocke et rejoue les **3 dernières valeurs** émises à n'importe quel nouvel abonné.
* *Cas d'usage :* Historique de messages, mise en cache de flux de données récents.



---

### 3. Création d'un Custom Operator RxJS (Opérateur personnalisé)

**La question :**

> Comment factoriser une suite répétitive d'opérateurs RxJS en un opérateur personnalisé réutilisable ?

**La réponse type :**
Un opérateur RxJS personnalisé n'est rien d'autre qu'une fonction pure qui encapsule une suite logique à l'aide de la fonction utilitaire `pipe`. Cela permet de garder les composants propres et de mutualiser la logique métier récurrente (ex: filtrage des valeurs nulles, désduplication, mapping).

**Exemple d'implémentation :**

```typescript
import { pipe, Observable } from 'rxjs';
import { filter, distinctUntilChanged, map } from 'rxjs';

// Définition d'un opérateur personnalisé pour nettoyer et extraire des données
export function filterAndExtractData<T, R>() {
  return pipe(
    filter((val: T | null | undefined) => val !== null && val !== undefined),
    distinctUntilChanged(),
    map((val: any) => val.data as R)
  );
}

```

**Utilisation dans un composant :**

```typescript
this.myService.rawResponse$.pipe(
  filterAndExtractData<MyApiResponse, MyCleanModel>()
).subscribe(cleanData => {
  // Données prêtes à l'emploi et filtrées
});

```


## Multicasting

Voici la fiche technique récapitulative pour la question sur le **Multicasting (`shareReplay`) vs Cold Observables**, prête à être ajoutée à ton `README.md` juste avant la question 4 :

---

### 5. Multicasting et suppression des appels HTTP doublons (`shareReplay`)

**La question :**

> Dans Angular, si deux composants s'abonnent séparément à un même flux retourné par `HttpClient` (ex: `http.get()`), combien d'appels réseau réels seront déclenchés ? Comment s'assurer qu'un seul et unique appel HTTP est exécuté puis partagé entre tous les abonnés ?

```ts
const data$ = this.http.get<any[]>('/api/expensive-data').pipe(
  map(response => response.data)
);

// Premier abonnement
data$.subscribe(data => console.log('Composant A :', data));

// Deuxième abonnement (un peu plus bas dans le même composant ou dans un autre)
data$.subscribe(data => console.log('Composant B :', data));
```

**La réponse type :**
Par défaut, le `HttpClient` d'Angular retourne des **Cold Observables**. Un Observable "froid" réexécute l'intégralité de sa logique à chaque nouvel `.subscribe()`. Ainsi, $N$ abonnés déclencheront $N$ requêtes HTTP distinctes vers le serveur.

Pour transformer ce flux en un **Hot Observable** et partager la réponse avec l'ensemble des abonnés actuels et futurs, on utilise l'opérateur **`shareReplay(1)`**.

**Exemple d'implémentation :**

```typescript
import { Injectable } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { shareReplay } from 'rxjs';

@Injectable({ providedIn: 'root' })
export class DataService {
  // Le flux est partagé entre tous les abonnés de l'application
  readonly data$ = this.http.get<Data[]>('/api/expensive-data').pipe(
    shareReplay(1) // Garde en mémoire la dernière émission et la rejoue aux abonnés
  );

  constructor(private http: HttpClient) {}
}

```

```typescript
// Composant A
this.dataService.data$.subscribe(data => console.log('A :', data));

// Composant B (s'abonne plus tard ou simultanément)
this.dataService.data$.subscribe(data => console.log('B :', data));

```

**Fonctionnement de `shareReplay(1)` :**

1. **Multicasting (`share`) :** La souscription à la source HTTP n'a lieu qu'une seule fois. Un seul appel réseau est émis.
2. **Tampon mémoire (`replay`) :** Il conserve en mémoire les $N$ dernières valeurs (ici $1$) et les distribue immédiatement à tout nouveau composant qui s'y abonne, sans jamais réinterroger le serveur.

⚠️ Le seul piège à connaître :

**Idéal pour une liste de pays, ou une donnée fixe, mais une donné amenée à muter, il faut éviter.**

La seule façon d'avoir une deuxième requête HTTP avec shareReplay(1), c'est si :

    Tous les composants se détruisent en même temps (0 abonné actif).

    Tu as configuré shareReplay avec l'option { refCount: true, bufferSize: 1 }.

    Un nouveau composant s'abonne après que le flux est repassé à 0 abonné.

---