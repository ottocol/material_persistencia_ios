
#Persistencia en dispositivos móviles
##iOS, sesión 4: Persistencia como servicio


---

##Puntos a tratar

- **Backend as a Service**
- Introducción a Parse
- Objetos persistentes
- Consultas
- Usuarios, roles y permisos


---

##El *backend* en aplicaciones móviles

- Hay muchas aplicaciones móviles que pueden funcionar sin *backend* pero otras lo necesitan
  + Para que el usuario pueda compartir los datos entre todos sus dispositivos
  + Para el aspecto "social": compartir información con otros

---

##Problema: desarrollar un *backend* es complicado

- Muchos desarrolladores de apps móviles no tienen experiencia y/o capacidad para desarrollar la parte del *backend*. Los lenguajes, plataformas y herramientas son totalmente distintos a los de móviles.

![](img/ikea1.png)


---

##Backend as a Service

- Plataformas en la nube que nos ofrecen los servicios típicos de *backend*, entre ellos la **persistencia de datos** (Database as a Service)

![](img/ikea1.png)
![](img/ikea2.png)


---

##Puntos a tratar

- Backend as a Service
- **Introducción a Parse**
- Objetos persistentes
- Consultas
- Usuarios, roles y permisos

---

##Para comenzar a trabajar con Parse

1. Registrarse como desarrollador en parse.com
2. Registrar una nueva aplicación en el *dashboard*
  - Cada aplicación tiene unas claves que debemos usar en nuestro código
3. Integrar las librerías de Parse en el proyecto Xcode

---

##Hello, Parse

```objectivec
[Parse setApplicationId:@"YJC4zDW3m6juerUr3e5khFWaAwK6LZPuymLsFY4R"
      clientKey:@"r7JVQa0aS1hyD5jwbbRPKWCyE8sQ3u35cVFIHmWG"];
PFObject *objetoDePrueba = [PFObject objectWithClassName:@"Prueba"];
objetoDePrueba[@"unaPropiedad"] = @"unValor";
[objetoDePrueba saveInBackground];
```

---

##Puntos a tratar

- Backend as a Service
- Introducción a Parse
- **Objetos persistentes**
- Consultas
- Usuarios, roles y permisos

---

## Objetos persistentes en Parse

- Son colecciones dinámicas de propiedades, muy similares a un `NSMutableDictionary`. 
- Por pertenecer a la misma clase de Parse dos objetos no tienen por qué tener las mismas propiedades. 
- Cuando un objeto no tiene una propiedad el valor de esta es `nil`.

---

##Propiedades

- Todas deben ser objetos, no se admiten valores primitivos

```objectivec
PFObject *objetoDePrueba = [PFObject objectWithClassName:@"Prueba"];
objetoDePrueba[@"numero"] = @42;
```

- Tampoco se admite cualquier clase como propiedad, solo `PFObject` (un objeto Parse puede tener como propiedad otro objeto Parse), `NSNumber`, `NSData`, `NSDate`, `NSString`, `NSNull`, `NSArray` y `NSDictionary`

---

##Creación de objetos

- `save` es síncrono, `saveInBackground` asíncrono. Para poder ejecutar código cuando acabe la operación (sea con OK o fallo):

```objectivec
[objetoDePrueba saveInBackgroundWithBlock:^(BOOL succeeded, NSError *error) {
  if (succeeded)
     NSLog(@"¡Ahora sí sé que se ha guardado!");
  else
     NSLog(@"Pues no, ha habido un error: %@", error.description);
}];
```

---

##Actualización y borrado

- **Actualizar**: modificar las propiedades + guardar de nuevo con `save` o `saveXXX`
- **Borrar**: con `delete` o `deleteInBackground`

---

##Relaciones entre objetos

- Asignamos un objeto Parse como propiedad de otro

```objectivec
PFObject *miBlog = [PFObject objectWithClassName:@"Blog"];
miBlog[@"titulo"] = @"Mis pensamientos más profundos";
PFObject *unPost = [PFObject objectWithClassName:@"Post"];
unPost[@"titulo"] = @"Hello world";
unPost[@"blog"] = miBlog;
[unPost saveInBackground];
PFObject *otroPost = [PFObject objectWithClassName:@"Post"];
otroPost[@"titulo"] = @"Hello Kitty";
otroPost[@"blog"] = miBlog;
[otroPost saveInBackground];
```


---

##Almacenamiento local

- Para datos que no nos interese almacenar en el servidor. Internamente usa SQLite

```objectivec
[Parse enableLocalDatastore]
[Parse setApplicationId:@"MI_ID_DE_APLICACION"
       clientKey:@"MI_CLAVE_CLIENTE"];
PFObject *unObjeto;
...
[unObjeto pinInBackground];
```

---

##Puntos a tratar

- Backend as a Service
- Introducción a Parse
- Objetos persistentes
- **Consultas**
- Usuarios, roles y permisos

---

## Consultas

Esquema de uso general:

- Crear un objeto `PFQuery`
- Añadir condiciones, (restricciones que deben cumplir los objetos u otros criterios, por ejemplo ordenación)
- Llamar a `findObjectsInBackgroundWithBlock:` o `findObjectsInBackgroundWithTarget:selector:` para obtener un `NSArray` de resultados

---

## Ejemplo de consulta

```objectivec
PFQuery *query = [PFQuery queryWithClassName:@"Alumno"];
[query whereKey:@"edad" equalTo:@20];
[query orderByAscending:@"nombre"];
query.limit = 50;
[query findObjectsInBackgroundWithBlock:^(NSArray *results, NSError *error) {
  if (!error) {
    NSLog(@"Encontrados %d alumnos", [results count]);
    for (PFObject *object in objects) {
        NSLog(@"%@", object.objectId);
    }
  } else {
    NSLog(@"Error: %@ %@", error, [error userInfo]);
  }
}];
```

---

## Más sobre consultas

- Además de `equalTo` tenemos `notEqualTo`, `greaterThan`, `greaterOrEqualTo`, …

- Cuando una propiedad es un array podemos comprobar si contiene un determinado valor

```objectivec
//Si "notas" es un array, aquí equalTo significa "contiene" no "es igual a"
[query whereKey:@"notas" equalTo:@10];
```

---

## Más sobre consultas (II)

- **Propiedades tipo cadena**: buscar subcadenas con `containsString`, o buscar las que comienzan o terminan por una cadena con `hasPrefix` y `hasSuffix`

```objectivec
[query whereKey:@"nombre" hasPrefix:@"Fran"];
```

---

## Más sobre consultas (y III)

Cuando un objeto tiene otros relacionados no se bajan automáticamente. Podemos incluir los objetos relacionados que deseemos con `includeKey`

```objectivec
PFQuery *query = [PFQuery queryWithClassName:@"Cancion"];
[query containsString:@"love"]
[query includeKey:@"disco"];
...
```


---

##Puntos a tratar

- Backend as a Service
- Introducción a Parse
- Objetos persistentes
- Consultas
- **Usuarios, roles y permisos**

---

## Gestión de usuarios

- Registro de nuevos usuarios
- Login y logout
- Pequeños detalles
  + *email* para confirmar registro
  + ¿olvidaste tu contraseña?

---

## Usuarios en Parse

- Clase `PFUser`: ya tiene algunos campos predefinidos: `email`, `username` y `password`. Podemos añadir los que queramos.

```objectivec
PFUser *usuario = [PFUser user];
//Campos "estándar"
usuario.username = @"master";
usuario.password = @"123456";
//Campo propio, no podemos usar la notación "."
usuario[@"telefono"] = @"555-123456";
```

---

## Registrar un nuevo usuario


```objectivec
[usuario signUpInBackgroundWithBlock:^(BOOL ok, NSError *error) {
  if (ok) {
      NSLog(@"Usuario registrado correctamente");
  }
  else {
      NSLog(@"Error al registrar usuario: %@", error.userInfo[@"error"]);
  }
}];
```

---

## Login y logout

- Login

```objectivec
[PFUser logInWithUsernameInBackground:@"myname" password:@"mypass"
  block:^(PFUser *user, NSError *error) {
    if (user) {
        NSLog(@"Login OK!!!!");
    } else {
        NSLog(@"Ha fallado el login");
    }
}];
```

- Logout

```objectivec
[PFUser logOut]
```

- ¿Quién es el usuario actual?

```objectivec
[PFUser currentUser]
```

---

##Permisos y objetos

- Una ACL es una lista de permisos de lectura o escritura sobre un determinado objeto. 
- Los permisos se pueden dar para un usuario concreto (o una lista de usuarios concretos) o para un rol (o lista de roles).

---

##Crear objeto con permisos

- Primero creamos una ACL con los permisos deseados y luego se la asignamos al objeto.

```objectivec
PFObject *obj = [PFObject objectWithClassName:@"Prueba"];
obj[@"texto"] = @"Soy un objeto con permisos!";
//Falta saber cómo se crea una ACL. Ahora veremos alternativas
PFACL *miACL = ...
//Asignamos la ACL al objeto
obj.ACL = miACL;
[obj saveInBackground];

```

---

## Crear ACLs

- ACL para objetos totalmente privados de un usuario

```objectivec
PFACL *miACL = [PFACL ACLWithUser:[PFUser currentUser]];
```

- Crear ACL vacía e irle añadiendo permisos

```objectivec
//Creamos una ACL "vacía"
PFACL *miACL = [PFACL ACL];
//Esta ACL es para un objeto visible para todos
//pero solo modificable para el usuario actual
[miACL setPublicReadAccess:YES]; 
[miACL setWriteAccess:YES forUser:[PFUser currentUser]];
```

