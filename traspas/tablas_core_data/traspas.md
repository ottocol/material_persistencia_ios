
#Persistencia en dispositivos móviles
##iOS, sesión 8: Tablas en Core Data


---

## Puntos a tratar

- ***Fetched results controller***
- Refrescar automáticamente la tabla
- Inserciones, Modificaciones y borrados
- Secciones de tabla


---


## ¿De dónde vienen los datos en una tabla?


- Típicamente de un `NSArray`
- En el objeto que actúa como *datasource*:

```objectivec
- (UITableViewCell *)tableView:(UITableView *)tableView cellForRowAtIndexPath:(NSIndexPath *)indexPath {
    UITableViewCell *cell = [tableView dequeueReusableCellWithIdentifier:@"cell" forIndexPath:indexPath];
    NSManagedObject *nota = self.notas[indexPath.row];
    cell.textLabel.text = [nota valueForKey:@"texto"];
    return cell;
}

```

---

## `NSFetchedResultsController`

- Sirve de **"puente"** entre Core Data y la vista de tabla 
- Puede **detectar los cambios en el contexto** para que podamos actualizar la tabla
- Nos ayuda a crear **secciones** e **índices**

---

## Un `NSFetchedResultsController` básico

- Necesitamos
  + Una *fetch request* que obtenga los datos a mostrar. 
  + La *request* debe estar **ordenada** (Usar `NSSortDescriptor`s)
  
```objectivec
NSFetchRequest *request = [[NSFetchRequest alloc] initWithEntityName:@"Nota"];
NSSortDescriptor *orden = [[NSSortDescriptor alloc ] 
                               initWithKey:@"fecha" ascending:NO];
request.sortDescriptors = @[orden];

//Tenemos una @property NSFetchedResultsController *frController  
self.frController = [[NSFetchedResultsController alloc] 
                          initWithFetchRequest:request
                          managedObjectContext:miContexto
                          //Esto por el momento no lo usamos
                          sectionNameKeyPath:nil 
                          cacheName:@"miCache"]; 
[self.frController performFetch:nil];
```

---

## Para ver los datos en la tabla

- Recordemos que debe haber un objeto que actúe de *datasource* 
- El *datasource* debe implementar `numberOfSectionsInTableView`, `tableView:numberOfRowsInSection:` y `tableView:cellForRowAtIndexPath:`

- **Número de secciones**

```objectivec
- (NSInteger)numberOfSectionsInTableView:(UITableView *)tableView {
     return [[self.frController sections] count];
}
```

---

## Para ver los datos en la tabla (II)


- Número de filas en la sección actual:

```objectivec
- (NSInteger)tableView:(UITableView *)tableView 
         numberOfRowsInSection:(NSInteger)section {
   id<NSFetchedResultsSectionInfo> sectionInfo = [self.frController sections][section];
   return [sectionInfo numberOfObjects];
}
```

---

## Para ver los datos en la tabla (III)

- Obtener una celda

```objectivec
- (UITableViewCell *)tableView:(UITableView *)tableView 
                     cellForRowAtIndexPath:(NSIndexPath *)indexPath {
    UITableViewCell *cell = [tableView dequeueReusableCellWithIdentifier:@"cell" 
                                       forIndexPath:indexPath];
    NSManagedObject *nota = [self.frController objectAtIndexPath:indexPath];
    cell.textLabel.text = [nota valueForKey:@"texto"];
    return cell;
}
```

---


## Refrescar la tabla

- Mismo problema de la primera versión: cada vez que creamos una nueva nota esta no aparece en la tabla. 
- El *fetched results controller*, está “suscrito” a los cambios que se producen en el contexto de persistencia
- Para que nos avise a su vez de estos cambios, tenemos que convertirnos en su *delegate*. 

```objectivec
self.frController.delegate = self;
```

---

## Convertirse en el *delegate*

- Declarar que la clase es conforme al protocolo `<NSFetchedResultsControllerDelegate>`

```objectivec
//ListaNotasController.h
#import <UIKit/UIKit.h>
#import <CoreData/CoreData.h>

@interface ListaNotasController : UITableViewController <NSFetchedResultsControllerDelegate>
...
@end
```

---

## "Escuchar" los cambios

- Si se ha modificado algún objeto del contexto se llamará a `controllerDidChangeContent`. Aprovechamos para refrescar los datos

```objectivec
- (void) controllerDidChangeContent:(NSFetchedResultsController *)controller {
     [self.tableView reloadData];
}
```

---

## Permitir el borrado de filas

- Debemos ser el *delegate* de la tabla. Implementar el siguiente método:

```objectivec
- (void)tableView:(UITableView *)tableView 
          commitEditingStyle:(UITableViewCellEditingStyle)editingStyle 
          forRowAtIndexPath:(NSIndexPath *)indexPath {
    if (editingStyle == UITableViewCellEditingStyleDelete) {
        //Borramos el objeto gestionado
        [ self.frController.managedObjectContext
          deleteObject:[self.frController objectAtIndexPath:indexPath]];
        NSError *error;
        [[self.frController managedObjectContext] save:&error];
        if (error) {
            NSLog(@"Error al intentar borrar objeto");
        }
    }
} 
```

---

## Editar la tabla de forma un poco más sofisticada

```objectivec
- (void)controller:(NSFetchedResultsController *)controller 
               didChangeObject:(id) anObject
               atIndexPath:(NSIndexPath *) indexPath 
               forChangeType:(NSFetchedResultsChangeType) type
               newIndexPath:(NSIndexPath *) newIndexPath {
    UITableView *tableView = self.tableView;
    switch(type) {
        case NSFetchedResultsChangeInsert:
            [tableView insertRowsAtIndexPaths:@[newIndexPath] 
                       withRowAnimation:UITableViewRowAnimationFade];
            break;
    ...

}
```

---

## Crear secciones automáticamente


- Al crear el controller

```objectivec
[[NSFetchedResultsController alloc] initWithFetchRequest:request 
  managedObjectContext:miContexto sectionNameKeyPath:@"DiaFecha" 
  cacheName:@"miCache"];
```

- En el *datasource*

```objectivec
- (NSString *) tableView:(UITableView *)tableView titleForHeaderInSection:(NSInteger)section {
    id <NSFetchedResultsSectionInfo> sectionInfo = 
                          [[self.frController sections] objectAtIndex:section];
    return [sectionInfo name];
}
```

---




# ¿Alguna pregunta?