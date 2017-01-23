
#Persistencia en dispositivos móviles
##iOS, sesión 9: 
##Contextos múltiples en Core Data


---

## Puntos a tratar

- Por qué contextos múltiples
- Contextos para trabajos en *background*
- Comunicación entre contextos
- Contextos anidados

---

## Puntos a tratar

- **Por qué contextos múltiples**
- Contextos para trabajos en *background*
- Comunicación entre contextos
- Contextos anidados

---

## Aplicaciones que necesitan concurrencia

- El código que escribimos normalmente se ejecuta en el mismo *thread* que la interfaz de usuario. Si realizamos una operación costosa la interfaz se bloqueará.
- La solución es realizar estas operaciones en otro *thread*. Como el API de *threading* es de bajo nivel, mejor *colas de operaciones*, una abstracción mucho más fácil de usar (o alternativamente *grand central dispatch*)

```swift
let background = OperationQueue();
background.addOperation() {
    print("Comienzo mi duro trabajo...")
    sleep(4)
    print("...terminado!")
    print("pero yo no puedo tocar la interfaz")
    OperationQueue.main.addOperation() {
        print("Soy main. Desde aquí se puede actualizar la interfaz")
    }
}
```

---

## Multi-threading y Core Data

- Por desgracia ni los objetos gestionados ni el contexto de persistencia son *thread-safe*, no debemos operar con ellos desde distintos *threads*
- Solución:
   * Creamos varios contextos
   * **Cada contexto trabaja en su propio *thread***

```swift
//Crear un contexto asociado a la cola de operaciones principal
let contexto = NSManagedObjectContext(concurrencyType: .mainQueueConcurrencyType)
//Crear un contexto asociado a una nueva cola de operaciones
let contexto2 = NSManagedObjectContext(concurrencyType: .privateQueueConcurrencyType)
```

---

## Puntos a tratar

- Por qué contextos múltiples
- **Contextos para trabajos en *background***
- Comunicación entre contextos
- Contextos anidados

---

## Múltiples contextos para *background*

El caso de uso más típico para los múltiples contextos es un contexto adicional asociado a una nueva cola desde el que hacemos las operaciones costosas. Así no bloqueamos la interfaz.

Es tan típico que en iOS10 el `NSPersistentContainer` tiene un API especial para esto

```swift
let miDelegate = UIApplication.shared.delegate as! AppDelegate
miDelegate.persistentContainer.performBackgroundTask() {
   contextoBG in
    let request = NSFetchRequest<Nota>(entityName: "Nota")  
    let lista = try! contextoBG.fetch(request)
    for nota in lista {
        //Aquí hacemos una operación costosa, como exportar cada nota a PDF
        //o subirlas a un servidor
    }
}
```

---

## Múltiples contextos para *background* (II)


- También podemos pedirle al *persistent container* explícitamente un nuevo contexto

```swift
let miDelegate = UIApplication.shared.delegate as! AppDelegate
let contextoBG = miDelegate.persistentContainer.newBackgroundContext()
```

- En iOS<10 se tendría que crear el contexto directamente

```swift
let contextoBG = NSManagedObjectContext(concurrencyType: .privateQueueConcurrencyType)
//Aquí falta conectar el contexto con un "persistent store coordinator"
//(ver primera sesión de Core Data)
```


---

## Múltiples contextos para *background* (III)


Una vez creado el contexto de *background*, para ejecutar una operación en él se usa `perform` (asíncrono) o `performAndWait` (síncrono)

```swift
let nuevoContexto = NSManagedObjectContext(concurrencyType:.privateQueueConcurrencyType)
...
nuevoContexto.perform() {
   //Aquí vendría la operación costosa
   ...
}
```


---

## Puntos a tratar

- Por qué contextos múltiples
- Contextos para trabajos en *background*
- **Comunicación entre contextos**
- Contextos anidados

---

Supongamos una *app* en la que tenemos una búsqueda/recuperación de datos muy costosa, y la hacemos en un contexto asociado a una nueva cola de operaciones (*background*). **Cuando recuperamos los datos, los obtenemos en otro contexto**, ¿cómo los mostramos desde el contexto principal?

Importante: **los objetos gestionados no se deben compartir directamente entre contextos**. Cada objeto gestionado está asociado al contexto en que "nació"

---

## Pasar objetos entre contextos con el identificador

- Cada objeto persistente tiene un identificador único (propiedad `objectID`).**El id es único también entre contextos**
- Es sencillo recuperar un objeto a partir de su id: `objectWithID`

Solución: desde el hilo secundario le pasamos al principal un array con los `id` de los objetos, y los "re-materializamos" en el hilo principal

---

## Pasar objetos entre contextos sincronizando

- Recordemos que cuando un objeto persistente se guarda, emite una notificación
- Esta notificación se puede escuchar desde cualquier hilo
- El método `mergeChanges` *refresca* un contexto actualizándolo con la información contenida en la notificación

---

```swift
let miDelegate = UIApplication.shared.delegate as! AppDelegate
//El contexto principal, como hasta ahora
let miContexto = miDelegate.persistentContainer.viewContext
//Un contexto secundario
let contextoBg = miDelegate.persistentContainer.newBackgroundContext()
//El "notification center" de la aplicación
let nc = NotificationCenter.default
//Cuando el contextoBg emita una notificación de este tipo, ejecutamos el código
nc.addObserver(forName: .NSManagedObjectContextDidSave,
               object: contextoBg,
               queue: nil) {
     notificacion in
       miContexto.mergeChanges(fromContextDidSave: notificacion)
}
```


---

## Puntos a tratar

- Por qué contextos múltiples
- Contextos para trabajos en *background*
- Comunicación entre contextos
- **Contextos anidados**

---

## Contextos anidados

- Como ya hemos visto, normalmente el contexto de persistencia está conectado con el *persistent store coordinator*, y este "gestiona la BD".

- Desde iOS5 se pueden crear contextos conectados a otros (hijos/padres). Cuando el hijo hace `save` se lo enviamos al padre, pero hasta que este no hace `save` no se hace persistente

---

![](img/contexto_hijo.png)

---

## Contextos anidados para "modo borrador"

- Supongamos una pantalla de la *app* donde podamos hacer muchos cambios, y donde queremos que el usuario pueda cancelar
- Creamos un contexto hijo y hacemos las operaciones en él. Si el usuario le da a *guardar*, hacemos `save()` del hijo y del padre. Si le da a cancelar, simplemente no hacemos nada. Se perderá el contexto hijo y con él los cambios

---

## Contextos anidados para almacenar en *background*

Alternativa al método que vimos al principio.

- Tenemos un contexto hijo en la cola principal y el padre en una cola aparte
- Trabajamos habitualmente en el contexto hijo
- Las operaciones de guardado no serán costosas ya que `save()` del hijo solo guardará en memoria. El `save()` del padre se hace en *background* y por tanto tampoco paraliza la UI

---


# ¿Alguna pregunta?