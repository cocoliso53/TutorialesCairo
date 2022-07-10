# Diccionarios en cairo vol. 1

## ¿Diccionarios en cairo?

Como tal la estructura de diccionario no existe en cairo, lo que se hace es usar un `struct`. 
En particular el llamado `DictAccess` que tiene la siguiente forma:

```
struct DictAccess:
    member key : felt
    member prev_value : felt
    member new_value : felt
end
```

Esto representa un par `{key: value}`  donde `value = new_value` , así que un diccionario en cairo es un array de `DictAccess`.
De ahora en adelante cuando vean *diccionario* piensen en un array cuyos elementos son de tipo `DictAccess` 
(o, en términos mas de cairo, un diccionario es `DictAccess*`.


Además, en dicho array podemos ir guardando los cambios en los valores para una o más llaves. 
Por ejemplo, imaginemos que tenemos un diccionario con los siguientes elementos:

```
func foo ....

    alloc_locals()

    let (local dict: DictAccess*) = alloc()
    assert[0] = DictAccess(key=0,prev_value=0,new_value=1)
    assert[1] = DictAccess(key=0,prev_value=1,new_value=2)
    assert[2] = DictAccess(key=0,prev_value=2,new_value=3)
    assert[3] = DictAccess(key=1,prev_value=0,new_value=0)

    ...

    return()
end
```

Lo que hicimos en el código arriba fue crear un array de 4 elementos cada uno de tipo `DictAccess` 
(si no saben por qué `DictAccess*` es un array en cairo entonces revisen 
[este](https://mirror.xyz/espejel.eth/jN2rapzYQDKJqY-MAz2Bz0VMIkmqi0IAmWbFdhq09CU) tutorial de Omar).

Noten aquí que los tres primeros elementos tienen la misma `key` que es 0, 
así que cada uno de ellos está representando como se va *“actualizando”* el valor que coresponde a esa llave.  
Es por eso que los `prev_value` y `new_value` tienen que ser congruentes entre sí, para representar la continuidad del cambio.

## Aplastando diccionarios

Otra cosa importante a tomar en cuenta es que este array tiene cuatro elementos en total pero solo dos keys distintas. 
Si solo nos interesa la configuración final del diccionario y no sus estados transitorios, entonces podemos resumir la información que tenemos. 
En cairo ya existe una función que resume un diccionario, esta función es `squash_dict` y la podemos encontrar [acá](https://github.com/starkware-libs/cairo-lang/blob/master/src/starkware/cairo/common/squash_dict.cairo).

Veamos la definición de la función:

```
func squash_dict{range_check_ptr}(
    dict_accesses : DictAccess*, dict_accesses_end : DictAccess*,   
    squashed_dict : DictAccess*) -> (squashed_dict : DictAccess*):

  ...

end
```

La función `squash_dict` toma tres argumentos (cuatro si se cuenta el argumento implícito), 
estos son `dict_accesses`, `dict_accesses_end` y `squashed_dict`  donde todos son de tipo `DictAccess*`, 
es decir son pointers a un `DictAccess`. Ahora vamos a describir cada uno de ellos:

- `dict_accesses` - Pointer al **inicio del array** de `DictAccess` (del diccionario) que queremos aplastar o resumir

- `dict_accesses_end` - Pointer al **final del mismo diccionario** al que se referencio en el argumento `dict_acess`. Basicamente así es como le decimos a cairo que tan grande es el array y cuantos elementos tiene.

- `squashed_dict` - Pointer a un **nuevo array** donde se empezará a escribir en la memoria el resultado de aplastar el diccionario original.

Por alguna razón en el código fuente un argumento y el resultado tienen el mismo nombre: `squashed_dict`. 
Aunque en la práctica podemos ver que normalmente al resultado de la función se le nombra `squashed_dict_end`. 
La razón es sencilla, el resultado de `squash_dict` es un pointer al final del array que comienza en `squashed_dict` 
(algunas veces se el argumento `squashed_dict` es nombrado `squashed_dict_start`).

El código hasta ahora con el diccionario aplastado se vería así:

```
%builtins range_check

from starkware.cairo.common.squash_dict import squash_dict
from starkware.cairo.common.alloc import alloc
from starkware.cairo.common.dict_access import DictAccess

func foo ....

    alloc_locals()
    
    #Creando el pointer del primer diccionario
    let (local dict: DictAccess*) = alloc()
    
    #Metiendo elementos en el diccionario
    #noten que los primeros tres elementos
    #tienen la misma llave
    #y que new_value del elemento anterior es igual
    #a prev_value del siguiente
    assert[0] = DictAccess(key=0,prev_value=0,new_value=1)
    assert[1] = DictAccess(key=0,prev_value=1,new_value=2)
    assert[2] = DictAccess(key=0,prev_value=2,new_value=3)
    assert[3] = DictAccess(key=1,prev_value=0,new_value=0)

    #Ahora vamos con los diccionarios aplastados

    #Pointer inicial del diccionario aplastado
    let (local squashed_dict: DictAccess) = alloc()

    #Creación del diccionario aplastado
    #y obtenemos el pointer al final del mismo
    let (squashed_dict_end) = squash_dict(dict_access=dict,
                             dict_access_end=dict+DictAccess.SIZE*4,
                             squashed_dict=squashed_dict)

    ...

    return()
end
```

Ahora el nuevo diccionario aplastado que empieza en `squashed_dict` es de solo dos elementos de longitud.

### Casteando magia negra

Hasta ahora todo es muy bonito y (más o menos) entendible. 
Pero qué pasa si por alguna razón queremos que nuestros diccionarios tengan values que no sean de tipo `felt`?

Supongamos que tenemos un `struct`  propio que se ve de la siguiente forma:

```
struct Posicion:
    member x: felt
    member y: felt
end
```

La forma y los miembros del `struct` son arbitrarios pero por ahora y para este ejemplo vamos a dejar las cosas simples. 

Si quisieramos tener un diccionario con `values` de tipo `Posicion` entonces qué hacemos? 

Una posible respuesta es crear un `struct` nuevo desde cero que tenga una estructura similar a `DictAccess`, por ejemplo:

```
struct PosAccess:
  member key: felt
  member prev_posicion: Posicion*
  member new_posicion: Posicion*
end
```

Una desventaja sería que sobre este `struct` no podemos aplicar la función `squash_dict` 
o algunas otras funciones que estén implementadas para `DictAccess` si es que la necesitaramos.
O más bien tendríamos que implementarlas desde cero para nuestro `struct` en específico.


La segunda solución que se me ocurre es un poco más esotérica, y para ser honesto, todavía no entiendo muy bien sus límites al 100%. 

Esta solución consiste en tomar una dirección en la memoria con tipo `Posicion*` y *castearla* a un `felt` con la función `cast` (duh). 
Habiendo hecho eso podemos usar un `DictAccess` común y corriente donde los valores para `prev_value` y `new_value` son `Posicion*` vestidos de `felt`.  

Ahora ya podríamos utilizar la función `squash_dict`. Va un ejemplo en código para que quede más claro:


```
%builtins output

from starkware.cairo.common.serialize import serialize_word
from starkware.cairo.common.dict_access import DictAccess
from starkware.cairo.common.alloc import alloc

# Creamos un struct cualquiera
struct Posicion:
    member x: felt
    member y: felt
end

func main{output_ptr: felt*}():
    alloc_locals

    # Array de strcuts Posicion
    let (local pos : Posicion*) = alloc()
    
    # Array de DictAccess checar para más detalle
    let (local dict : DictAccess*) = alloc()


    # Llenamos el array de Posicion
    assert pos[0] = Posicion(x=0,y=0)
    assert pos[1] = Posicion(x=1,y=0)
    assert pos[2] = Posicion(x=1,y=1)
    assert pos[3] = Posicion(x=0,y=1)

    # Llenamos el array de DictAccess
    # los tipos de `prev_value` y `next_value` deben de ser `felt`
    # por eso acá tomamos la dirección de una Posicion y la casteamos a felt
    # notar que solo existen los valores 0 y 1 para key
    assert dict[0] = DictAccess(key=0,prev_value=cast(&pos[3],felt),new_value=cast(&pos[1],felt))
    assert dict[1] = DictAccess(key=0,prev_value=cast(&pos[1],felt),new_value=cast(&pos[0],felt))
    assert dict[2] = DictAccess(key=0,prev_value=cast(&pos[0],felt),new_value=cast(&pos[2],felt))
    assert dict[3] = DictAccess(key=1,prev_value=cast(&pos[2],felt),new_value=cast(&pos[3],felt))
    
    # Creamos el espacio para el diccionario aplastado
    let (local squashed_dict : DictAccess*) = alloc()
    
    # Aplastamos el diccionario
    let (squashed_dict_end : DictAccess*) = squash_dict(dict_accesses=dict,dict_accesses_end=dict+DictAccess.SIZE*4,squashed_dict=squashed_dict)
    
    # Ahora vamos a recuperar los valores
    tempvar p: Posicion*
    
    # Tomamos el primer elemento de squashed_dict (que es un DictAccess)
    # new_value es un pointer que casteamos a felt
    # hay que hacer entonces lo inverso
    # recordar que squashed_dict marca el inicio del array
    # squashed_dict_end marca el final
    p = cast(squashed_dict[0].new_value,Posicion*)
    
    # Podríamos ya acceder a los miembros de p
    serialize_word(p.x)
    serialize_word(p.y)
    
    return()
end
```

### Comentarios finales

La siguiente pregunta que debe surgir es: si podemos castear pointers a `felt`s y así asignar `prev_value` y `new_value`, podemos
hacer lo mismo pero con `key`? Esta es una de las preguntas que pienso responder en el siguiente tutorial. La verdad todavía no sé la respuesta. 

También es importante mencionar que, no porque se pueda hacer quiere decir que debamos hacerlo. El lenguaje todavía es muy nuevo y no sabemos que efectos
secundarios podríamos llegar a tener por hacer este tipo de cosas en donde podríamos decir que estamos "engañando" al lenguaje para
que haga lo que nosotros queramos, pero en una de esas podemos tener resultados adversos. 

