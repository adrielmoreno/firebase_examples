# Firebase

## Deep linking

### Android

1. Agregue los siguientes metadados dentro de la etiqueta *activity* en *android/app/src/main/AndroidManifest.xml*. Modfique el :host="su_dominio.com"

    ```xml
    <meta-data android:name="flutter_deeplinking_enabled" android:value="true" />
    <intent-filter android:autoVerify="true">
        <action android:name="android.intent.action.VIEW" />
        <category android:name="android.intent.category.DEFAULT" />
        <category android:name="android.intent.category.BROWSABLE" />
        <data android:scheme="https" android:host="su_dominio.com" />
    </intent-filter>
    ```

2. Subir el fichero *.well-known/assetlinks.json* a su hosting web. Este archivo le dice al navegador móvil qué aplicación de Android abrir en lugar del navegador.

    2.1 el fichero debe estar en la ruta *su_dominio.com/.well-known/assetlinks.json*

    ```json
    [{
    "relation": ["delegate_permission/common.handle_all_urls"],
    "target": {
        "namespace": "android_app",
        // aplicationId
        "package_name": "com.example.app",
        "sha256_cert_fingerprints":
        ["FF:2A:CF:7B:DD:CC:F1:03:3E:E8:B2:27:7C:A2..."]
    }
    }]
    ```

    2.2 Si ya está subida la app a producción puede localizar la configuración de este fichero en play.google.com/console y buscar **Digital Asset Links JSON**

    2.3 Si no está subida la app, debe crear su [clave de firma](https://developer.android.com/studio/publish/app-signing?hl=es-419) sha256.

3. Para probar use el comando adb:

   ```bash
   adb shell 'am start -a android.intent.action.VIEW \
    -c android.intent.category.BROWSABLE \
    -d "https://<web-domain>/details"' \
    <package name>
   ```

### iOS

1. Agregar la clave *FlutterDeepLinkingEnabled* con valor true en el **info.plist**:

    ```xml
   <key>FlutterDeepLinkingEnabled</key>
   <true/>
    ```

2. Agregar su dominio web en el **Runner.entitlements**:

    ```xml
   <key>com.apple.developer.associated-domains</key>
   <array>
    <string>applinks:su_dominio.com</string>
    </array>
    ```

3. Subir el fichero *.well-known/apple-app-site-association* a su hosting web.

    3.1 *Fichero sin extensión* debe estar en la ruta *su_dominio.com/.well-known/apple-app-site-association*:

    ```json
    {
    "applinks": {
        "apps": [],
        "details": [
        {
            "appID": "S8QB4VV633.com.surgschool.mobile.app",
            "paths": ["*"]
        }
        ]
    }
    }
    ```

    - **appID** es la combinación del [team_id_apple](https://developer.apple.com/account/resources/identifiers/list).bundle_id_apple
    - **["*"]**: Permite todas las rutas.

4. Para probar use el comando:

   ```bash
   xcrun simctl openurl booted https://<web domain>/details
   ```

## Cloud Firestore

- Agregar la dependencia:

  ```bash
   flutter pub add cloud_firestore
   ```

### Escritura

1. Inicializa una instancia de Cloud Firestore:

   ```dart
   final db = FirebaseFirestore.instance;
   ```

2. Crear o reemplazar. **set()**:

   ```dart
    final city = <String, String>{
    "name": "Los Angeles",
    "state": "CA",
    "country": "USA"
    };

    db.collection("cities")
    .doc("LA")
    .set(city)
    .onError((e, _) => print("Error writing document: $e"));    
   ```

    ```dart
    final docData = {
    "stringExample": "Hello world!",
    "booleanExample": true,
    "numberExample": 3.14159265,
    "dateExample": Timestamp.now(),
    "listExample": [1, 2, 3],
    "nullExample": null
    };

    final nestedData = {
    "a": 5,
    "b": true,
    };

    docData["objectExample"] = nestedData;

    db.collection("data")
    .doc("one")
    .set(docData)
    .onError((e, _) => print("Error writing document: $e"));  
     ```

    2.1. Combinación de los datos. **SetOptions()**:

    ```dart
    // Update one field, creating the document if it does not already exist.
    final data = {"capital": true};

    db.collection("cities").doc("BJ").set(data, SetOptions(merge: true));   
     ```

3. Agregar un documento:

   ```dart
   // Create a new user with a first and last name
    final user = <String, dynamic>{
    "first": "Ada",
    "last": "Lovelace",
    "born": 1815
    };

    // Add a new document with a generated ID
    db.collection("users").add(user).then((DocumentReference doc) => print('DocumentSnapshot added with ID: ${doc.id}'));
   ```

4. Escritura con Objetos personalizados (**Map** o **Dictinary**):

    ```dart
    class City {
    final String? name;
    final String? state;
    final String? country;
    final bool? capital;
    final int? population;
    final List<String>? regions;

    City({
        this.name,
        this.state,
        this.country,
        this.capital,
        this.population,
        this.regions,
    });

    factory City.fromFirestore(
        DocumentSnapshot<Map<String, dynamic>> snapshot,
        SnapshotOptions? options,
    ) {
        final data = snapshot.data();
        return City(
        name: data?['name'],
        state: data?['state'],
        country: data?['country'],
        capital: data?['capital'],
        population: data?['population'],
        regions:
            data?['regions'] is Iterable ? List.from(data?['regions']) : null,
        );
    }

    Map<String, dynamic> toFirestore() {
        return {
        if (name != null) "name": name,
        if (state != null) "state": state,
        if (country != null) "country": country,
        if (capital != null) "capital": capital,
        if (population != null) "population": population,
        if (regions != null) "regions": regions,
        };
    }
    }
    ```

    ```dart
    final city = City(
    name: "Los Angeles",
    state: "CA",
    country: "USA",
    capital: false,
    population: 5000000,
    regions: ["west_coast", "socal"],
    );
    final docRef = db
        .collection("cities")
        .withConverter(
        fromFirestore: City.fromFirestore,
        toFirestore: (City city, options) => city.toFirestore(),
        )
        .doc("LA");
    await docRef.set(city);
    ```

5. Actualizar datos:

   ```dart
    final washingtonRef = db.collection("cites").doc("DC");
    washingtonRef
    .update({"capital": true})
    .then((value) => print("DocumentSnapshot successfully updated!"),
        onError: (e) => print("Error updating document $e"));
   ```

    - Actualiza los campos en objetos anidados. Usando . es más optimo.

   ```dart
    // Assume the document contains:
    // {
    //   name: "Frank",
    //   favorites: { food: "Pizza", color: "Blue", subject: "recess" }
    //   age: 12
    // }
    db.collection("users")
        .doc("frank")
        .update({"age": 13, "favorites.color": "Red"});
   ```

    - Actualizar elementos de un array. **arrayUnion()**, **arrayRemove()**

   ```dart
   final washingtonRef = db.collection("cities").doc("DC");

    // Atomically add a new region to the "regions" array field.
    washingtonRef.update({
    "regions": FieldValue.arrayUnion(["greater_virginia"]),
    });

    // Atomically remove a region from the "regions" array field.
    washingtonRef.update({
    "regions": FieldValue.arrayRemove(["east_coast"]),
    });
   ```

    - Actualizar Incremetar. **increment(value)**

   ```dart
    var washingtonRef = db.collection('cities').doc('DC');

    // Atomically increment the population of the city by 50.
    washingtonRef.update(
    {"population": FieldValue.increment(50)},
    );
   ```

#### Transacciones

   ```dart
   // lectura deben ejecutarse siempre antes de cualquier operación 
    final sfDocRef = db.collection("cities").doc("SF");
    db.runTransaction((transaction) async {
    final snapshot = await transaction.get(sfDocRef);
    // Note: this could be done without a transaction
    //       by updating the population using FieldValue.increment()
    final newPopulation = snapshot.get("population") + 1;
    transaction.update(sfDocRef, {"population": newPopulation});
    }).then(
    (value) => print("DocumentSnapshot successfully updated!"),
    onError: (e) => print("Error updating document $e"),
    );
   ```

- Pasar información fuera la transacción.

No modifiques el estado de la aplicación dentro de las funciones de transacción. Si lo haces, es posible que se generen problemas de simultaneidad debido a que las funciones de transacción pueden ejecutarse varias veces y no se garantiza que se ejecuten en el subproceso de IU.

   ```dart
    final sfDocRef = db.collection("cities").doc("SF");
    db.runTransaction((transaction) {
    return transaction.get(sfDocRef).then((sfDoc) {
        final newPopulation = sfDoc.get("population") + 1;
        transaction.update(sfDocRef, {"population": newPopulation});
        return newPopulation;
    });
    }).then(
    (newPopulation) => print("Population increased to $newPopulation"),
    onError: (e) => print("Error updating document $e"),
    );
   ```

#### Escrituras en lotes. Múltiples operaciones

```dart
// Get a new write batch
final batch = db.batch();

// Set the value of 'NYC'
var nycRef = db.collection("cities").doc("NYC");
batch.set(nycRef, {"name": "New York City"});

// Update the population of 'SF'
var sfRef = db.collection("cities").doc("SF");
batch.update(sfRef, {"population": 1000000});

// Delete the city 'LA'
var laRef = db.collection("cities").doc("LA");
batch.delete(laRef);

// Commit the batch
batch.commit().then((_) {
// ...
});
```

### Borrado

1. **delete()**:

   ```dart
    db.collection("cities").doc("DC").delete().then(
        (doc) => print("Document deleted"),
        onError: (e) => print("Error updating document $e"),
        );
   ```

2. Borrar campos

    ```dart
    final docRef = db.collection("cities").doc("BJ");

    // Remove the 'capital' field from the document
    final updates = <String, dynamic>{
    "capital": FieldValue.delete(),
    };

    docRef.update(updates);
    ```

3. Borrar colecciones

    Si borras un documento, no se borrarán las subcolecciones que contiene.

    Para borrar por completo una colección o subcolección en Cloud Firestore, recupera (lee) todos los documentos de la colección o subcolección y bórralos. Este proceso genera costos de lectura y eliminación. Si tienes colecciones más grandes, te recomendamos borrar los documentos en lotes pequeños para evitar errores de memoria insuficiente. Repite el proceso hasta que borres toda la colección o subcolección.

    No se recomienda borrar colecciones desde el cliente.

### Lectura

1. Leer datos:

   ```dart
    final docRef = db.collection("cities").doc("SF");
    docRef.get().then(
    (DocumentSnapshot doc) {
        final data = doc.data() as Map<String, dynamic>;
        // ...
    },
    onError: (e) => print("Error getting document: $e"),
    );
   ```

    - Datos en caché

   ```dart
    final docRef = db.collection("cities").doc("SF");

    // Source can be CACHE, SERVER, or DEFAULT.
    const source = Source.cache;

    docRef.get(const GetOptions(source: source)).then(
        (res) => print("Successfully completed"),
        onError: (e) => print("Error completing: $e"),
        );
   ```

    - Lectura con Objetos personalizados (**Map** o **Dictinary**):
  
   ```dart
    final ref = db.collection("cities").doc("LA").withConverter(
        fromFirestore: City.fromFirestore,
        toFirestore: (City city, _) => city.toFirestore(),
        );
    final docSnap = await ref.get();
    final city = docSnap.data(); // Convert to City object
    if (city != null) {
        print(city);
    } else {
        print("No such document.");
    }
   ```

    - Iterar colecciones
  
   ```dart
    await db.collection("users").get().then((event) {
    for (var doc in event.docs) {
        print("${doc.id} => ${doc.data()}");
    }
    });
   ```

    - Filtrado (**where**):
  
   ```dart
    db.collection("cities").where("capital", isEqualTo: true).get().then(
    (querySnapshot) {
        print("Successfully completed");
        for (var docSnapshot in querySnapshot.docs) {
        print('${docSnapshot.id} => ${docSnapshot.data()}');
        }
    },
    onError: (e) => print("Error completing: $e"),
    );
   ```

    - Subcolecciones:
  
   ```dart
    db.collection("cities").doc("SF").collection("landmarks").get().then(
    (querySnapshot) {
        print("Successfully completed");
        for (var docSnapshot in querySnapshot.docs) {
        print('${docSnapshot.id} => ${docSnapshot.data()}');
        }
    },
    onError: (e) => print("Error completing: $e"),
    );
   ```

#### Lectura en timpo real. **Stream**, **StreamBuilder**, **listen**
  
  ```dart
    final docRef = db.collection("cities").doc("SF");
    docRef.snapshots()
    .listen((event) => print("current data: ${event.data()}"),
    onError:(error) => print("Listen failed: $error"),
    );
   ```

- En la UI

   ```dart
    class UserInformation extends StatefulWidget {
    @override
    _UserInformationState createState() => _UserInformationState();
    }

    class _UserInformationState extends State<UserInformation> {
    final Stream<QuerySnapshot> _usersStream =
        FirebaseFirestore.instance.collection('users').snapshots();

    @override
    Widget build(BuildContext context) {
        return StreamBuilder<QuerySnapshot>(
        stream: _usersStream,
        builder: (BuildContext context, AsyncSnapshot<QuerySnapshot> snapshot) {
            if (snapshot.hasError) {
            return const Text('Something went wrong');
            }

            if (snapshot.connectionState == ConnectionState.waiting) {
            return const Text("Loading");
            }

            return ListView(
            children: snapshot.data!.docs
                .map((DocumentSnapshot document) {
                    Map<String, dynamic> data =
                        document.data()! as Map<String, dynamic>;
                    return ListTile(
                    title: Text(data['full_name']),
                    subtitle: Text(data['company']),
                    );
                })
                .toList()
                .cast(),
            );
        },
        );
    }
    }
   ```

- Escuchar varios documentos en una colección

    ```dart
    db
        .collection("cities")
        .where("state", isEqualTo: "CA")
        .snapshots()
        .listen((event) {
    final cities = [];
    for (var doc in event.docs) {
        cities.add(doc.data()["name"]);
    }
    print("cities in CA: ${cities.join(", ")}");
    });
   ```

- Escuchar modificaciones individuales

    ```dart
    db
        .collection("cities")
        .where("state", isEqualTo: "CA")
        .snapshots()
        .listen((event) {
    for (var change in event.docChanges) {
        switch (change.type) {
        case DocumentChangeType.added:
            print("New City: ${change.doc.data()}");
            break;
        case DocumentChangeType.modified:
            print("Modified City: ${change.doc.data()}");
            break;
        case DocumentChangeType.removed:
            print("Removed City: ${change.doc.data()}");
            break;
        }
    }
    });
   ```

- Cancelar la escucha

    ```dart
    final collection = db.collection("cities");
    final listener = collection.snapshots().listen((event) {
    // ...
    });
    listener.cancel();
   ```

- Uso de **where()**
  El método where() usa tres parámetros: un campo para filtrar, una operación de comparación y un valor. Cloud Firestore admite los siguientes operadores de comparación:
  
  - < menor que
  - <= menor o igual que
  - == igual que
  - > mayor que
  - >= mayor que o igual que
  - != no igual a
  - array-contains
  - array-contains-any
  - in
  - not-in

    ```dart
    final citiesRef = db.collection("cities");

    final stateQuery = citiesRef.where("state", isEqualTo: "CA");
    final nameQuery = citiesRef.where("name", isEqualTo: "San Francisco");
    final notCapitals = citiesRef.where("capital", isNotEqualTo: true);

    final populationQuery = citiesRef.where("population", isLessThan: 100000);

    final westCoastcities = citiesRef.where("regions", arrayContains: "west_coast");

    // el campo coincide con cualquiera de los valores
    final cities = citiesRef.where("country", whereIn: ["USA", "Japan"]);
    // no coincide con ninguno de los valores de comparación
    final cities = citiesRef.where("country", whereNotIn: ["USA", "Japan"]);

    final cities = citiesRef.where("regions", arrayContainsAny: ["west_coast", "east_coast"]);
    ```

- Encadenar múltiples operadores

    ```dart
    citiesRef
        .where("state", isEqualTo: "CO")
        .where("name", isEqualTo: "Denver");

    citiesRef
        .where("state", isEqualTo: "CA")
        .where("population", isLessThan: 1000000);
    
    citiesRef
        .where("state", isGreaterThanOrEqualTo: "CA")
        .where("state", isLessThanOrEqualTo: "IN");
    citiesRef
        .where("state", isEqualTo: "CA")
        .where("population", isGreaterThan: 1000000);
   ```

- Consultas combinadas

    ```dart
    var query = db.collection("cities")
        .where(
            Filter.or(
            Filter("capital", isEqualTo: true),
            Filter("population", isGreaterThan: 1000000)
            ));

    var query = db.collection("cities")
        .where(
        Filter.and(
            Filter("state", isEqualTo: "CA"),
        Filter.or(
            Filter("capital", isEqualTo: true),
            Filter("population", isGreaterThan: 1000000)
        )));
   ```

- Consultas de grupo

    ```dart
    final citiesRef = db.collection("cities");

    final ggbData = {"name": "Golden Gate Bridge", "type": "bridge"};
    citiesRef.doc("SF").collection("landmarks").add(ggbData);

    final lohData = {"name": "Legion of Honor", "type": "museum"};
    citiesRef.doc("SF").collection("landmarks").add(lohData);

    db
        .collectionGroup("landmarks")
        .where("type", isEqualTo: "museum")
        .get()
        .then(
        (res) => print("Successfully completed"),
        onError: (e) => print("Error completing: $e"),
        );
   ```

   Antes de usar una consulta de grupos de colecciones, debes crear un índice que la admita. [Puedes crear un índice mediante un mensaje de error, la consola o Firebase CLI](https://firebase.google.com/docs/firestore/query-data/indexing?hl=es-419).

#### Orden **orderBy()** y límite **limit()**

```dart
    final citiesRef = db.collection("cities");
    citiesRef.orderBy("name").limit(3);

    // descendente
    citiesRef.orderBy("name", descending: true).limit(3);

    citiesRef
        .where("population", isGreaterThan: 100000)
        .orderBy("population")
        .limit(2);

    // no válido
    citiesRef.where("population", isGreaterThan: 100000).orderBy("country");
```

#### Paginar datos con cursores **startAt()**, **startAfter()**,**endAt()**, **endBefore()**

```dart
// startAt(A) -> A-Z
// startAfter(A) -> B-Z
db.collection("cities").orderBy("population").startAt([1000000]);
db.collection("cities").orderBy("population").endAt([1000000]);
```

- Documento para definir el cursor de consulta

    ```dart
    // todas las ciudades con una población mayor o igual que la de San Francisco
    db.collection("cities").doc("SF").get().then(
    (documentSnapshot) {
        final biggerThanSf = db
            .collection("cities")
            .orderBy("population")
            .startAtDocument(documentSnapshot);
    },
    onError: (e) => print("Error: $e"),
    );
    ```

- Paginar una consulta. Combina cursores de consulta con el método limit().

    ```dart
    // Construct query for first 25 cities, ordered by population
    final first = db.collection("cities").orderBy("population").limit(25);

    first.get().then(
    (documentSnapshots) {
        // Get the last visible document
        final lastVisible = documentSnapshots.docs[documentSnapshots.size - 1];

        // Construct a new query starting at this document,
        // get the next 25 cities.
        final next = db
            .collection("cities")
            .orderBy("population")
            .startAfterDocument(lastVisible).limit(25);

        // Use the query for pagination
        // ...
    },
    onError: (e) => print("Error completing: $e"),
    );
    ```

- Cursor en función de varios campos.

    ```dart
    /*
    *       Ciudades
    * Nombre        Estado
    * Springfield   Massachusetts
    * Springfield   Missouri
    * Springfield   Wisconsin
    */
    
    // Will return all Springfields
    db
        .collection("cities")
        .orderBy("name")
        .orderBy("state")
        .startAt(["Springfield"]);

    // Will return "Springfield, Missouri" and "Springfield, Wisconsin"
    db
        .collection("cities")
        .orderBy("name")
        .orderBy("state")
        .startAt(["Springfield", "Missouri"]);
    ```

#### Datos sin conexión

En las plataformas de Android y Apple, la persistencia sin conexión está habilitada de forma predeterminada. En la Web, la persistencia sin conexión está inhabilitada de forma predeterminada.

```dart
// Apple and Android
db.settings = const Settings(persistenceEnabled: true);

// Web
await db
    .enablePersistence(const PersistenceSettings(synchronizeTabs: true));
```

- Tamaño de la caché
  
    ```dart
    db.settings = const Settings(
    persistenceEnabled: true,
    cacheSizeBytes: Settings.CACHE_SIZE_UNLIMITED,
    );
    ```

- Esucha sin conexión
  
    ```dart
    db
        .collection("cities")
        .where("state", isEqualTo: "CA")
        .snapshots(includeMetadataChanges: true)
        .listen((querySnapshot) {
    for (var change in querySnapshot.docChanges) {
        if (change.type == DocumentChangeType.added) {
        final source =
            (querySnapshot.metadata.isFromCache) ? "local cache" : "server";

        print("Data fetched from $source}");
        }
    }
    });
    ```

1. [Protege los datos](https://firebase.google.com/docs/firestore/quickstart?hl=es-419#secure_your_data)

2. Referencias:

    ```dart
    final myDocumentRef = db.collection("users").doc("id_name");

    final myDocumentRef = db.collection("users/id_name");
   ```

   - Subcolección:

    ```dart
    final messageRef = db
        .collection("rooms")
        .doc("roomA")
        .collection("messages");
    ```
