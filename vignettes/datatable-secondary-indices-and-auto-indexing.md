---
title: "Secondary indices and auto indexing"
date: "2024-10-04"
output:
  markdown::html_format
vignette: >
  %\VignetteIndexEntry{Secondary indices and auto indexing}
  %\VignetteEngine{knitr::knitr}
  \usepackage[utf8]{inputenc}
---



Esta viñeta supone que el lector está familiarizado con la sintaxis `[i, j, by]` de data.table y con la forma de realizar subconjuntos rápidos basados en claves. Si no está familiarizado con estos conceptos, lea primero las viñetas *"Introducción a data.table"*, *"Semántica de referencia"* y *"Claves y subconjuntos rápidos basados en búsquedas binarias"*.

***

## Datos {#data}

Utilizaremos los mismos datos de "vuelos" que en la viñeta *"Introducción a data.table"*.




``` r
flights <- fread("flights14.csv")
head(flights)
#     year month   day dep_delay arr_delay carrier origin   dest air_time distance  hour
#    <int> <int> <int>     <int>     <int>  <char> <char> <char>    <int>    <int> <int>
# 1:  2014     1     1        14        13      AA    JFK    LAX      359     2475     9
# 2:  2014     1     1        -3        13      AA    JFK    LAX      363     2475    11
# 3:  2014     1     1         2         9      AA    JFK    LAX      351     2475    19
# 4:  2014     1     1        -8       -26      AA    LGA    PBI      157     1035     7
# 5:  2014     1     1         2         1      AA    JFK    LAX      350     2475    13
# 6:  2014     1     1         4         0      AA    EWR    LAX      339     2454    18
dim(flights)
# [1] 253316     11
```

## Introducción

En esta viñeta, vamos a

* analizar los *índices secundarios* y justificar por qué los necesitamos citando casos en los que configurar claves no es necesariamente ideal,

* realizar subconjuntos rápidos, una vez más, pero utilizando el nuevo argumento `on`, que calcula índices secundarios internamente para la tarea (temporalmente) y reutiliza si ya existe uno,

* y finalmente observe la *indexación automática* que va un paso más allá y crea índices secundarios automáticamente, pero lo hace en la sintaxis nativa de R para crear subconjuntos.

## 1. Índices secundarios

### a) ¿Qué son los índices secundarios?

Los índices secundarios son similares a las `claves` en *data.table*, excepto por dos diferencias importantes:

* No reordena físicamente toda la tabla de datos en la RAM. En cambio, solo calcula el orden para el conjunto de columnas proporcionadas y almacena ese *vector de orden* en un atributo adicional llamado `index`.

* Puede haber más de un índice secundario para una tabla de datos (como veremos a continuación).

### b) Establecer y obtener índices secundarios

#### -- ¿Cómo podemos establecer la columna `origen` como un índice secundario en la *tabla de datos* `vuelos`?


``` r
setindex(flights, origin)
head(flights)
#     year month   day dep_delay arr_delay carrier origin   dest air_time distance  hour
#    <int> <int> <int>     <int>     <int>  <char> <char> <char>    <int>    <int> <int>
# 1:  2014     1     1        14        13      AA    JFK    LAX      359     2475     9
# 2:  2014     1     1        -3        13      AA    JFK    LAX      363     2475    11
# 3:  2014     1     1         2         9      AA    JFK    LAX      351     2475    19
# 4:  2014     1     1        -8       -26      AA    LGA    PBI      157     1035     7
# 5:  2014     1     1         2         1      AA    JFK    LAX      350     2475    13
# 6:  2014     1     1         4         0      AA    EWR    LAX      339     2454    18

## alternatively we can provide character vectors to the function 'setindexv()'
# setindexv(flights, "origin") # useful to program with

# 'index' attribute added
names(attributes(flights))
# [1] "names"             "row.names"         "class"             ".internal.selfref"
# [5] "index"
```

* `setindex` y `setindexv()` permiten agregar un índice secundario a data.table.

* Tenga en cuenta que `vuelos` **no** se reordena físicamente en orden creciente de `origen`, como habría sido el caso con `setkey()`.

* Tenga en cuenta también que se ha añadido el atributo `index` a `flights`.

* `setindex(flights, NULL)` eliminaría todos los índices secundarios.

#### -- ¿Cómo podemos obtener todos los índices secundarios establecidos hasta ahora en 'vuelos'?


``` r
indices(flights)
# [1] "origin"

setindex(flights, origin, dest)
indices(flights)
# [1] "origin"       "origin__dest"
```

* La función `indices()` devuelve todos los índices secundarios actuales en la tabla de datos. Si no existe ninguno, se devuelve `NULL`.

* Nótese que al crear otro índice en las columnas `origin, dest`, no perdemos el primer índice creado en la columna `origin`, es decir, podemos tener múltiples índices secundarios.

### c) ¿Por qué necesitamos índices secundarios?

#### -- Reordenar una tabla de datos puede ser costoso y no siempre ideal.

Considere el caso en el que desea realizar un subconjunto rápido basado en clave en la columna `origin` para el valor "JFK". Lo haríamos de la siguiente manera:


``` r
## not run
setkey(flights, origin)
flights["JFK"] # or flights[.("JFK")]
```

#### `setkey()` requiere:

a) calcular el vector de orden para la(s) columna(s) proporcionada(s), aquí, `origen`, y

b) reordenar toda la tabla de datos, por referencia, en función del vector de orden calculado.

#

Calcular el orden no es la parte que consume mucho tiempo, ya que data.table utiliza un ordenamiento por base real en vectores de números enteros, caracteres y números. Sin embargo, reordenar data.table puede consumir mucho tiempo (según la cantidad de filas y columnas).

A menos que nuestra tarea implique la creación repetida de subconjuntos en la misma columna, la creación rápida de subconjuntos basada en claves podría anularse efectivamente al momento de reordenar, dependiendo de las dimensiones de nuestra tabla de datos.

#### -- Solo puede haber una `clave` como máximo

Ahora, si quisiéramos repetir la misma operación pero en la columna `dest`, para el valor "LAX", entonces tenemos que `setkey()`, *nuevamente*.


``` r
## not run
setkey(flights, dest)
flights["LAX"]
```

Y esto reordena los "vuelos" por "destino", *nuevamente*. Lo que realmente nos gustaría es poder realizar la subdivisión rápida eliminando el paso de reordenación.

¡Y esto es precisamente lo que permiten los *índices secundarios*!

#### -- Los índices secundarios se pueden reutilizar

Dado que puede haber varios índices secundarios y crear un índice es tan simple como almacenar el vector de orden como un atributo, esto nos permite incluso eliminar el tiempo para volver a calcular el vector de orden si ya existe un índice.

#### -- El nuevo argumento `on` permite una sintaxis más limpia y la creación y reutilización automática de índices secundarios.

Como veremos en la siguiente sección, el argumento `on` proporciona varias ventajas:

#### argumento `on`

* permite la creación de subconjuntos mediante el cálculo de índices secundarios sobre la marcha. Esto elimina la necesidad de ejecutar `setindex()` cada vez.

* permite la reutilización sencilla de índices existentes simplemente verificando los atributos.

* permite una sintaxis más clara al incluir las columnas en las que se ejecuta el subconjunto como parte de la sintaxis. Esto hace que el código sea más fácil de seguir cuando se lo analiza más adelante.

    Note that `on` argument can also be used on keyed subsets as well. In fact, we encourage providing the `on` argument even when subsetting using keys for better readability.

#

## 2. Creación rápida de subconjuntos mediante el argumento `on` e índices secundarios

### a) Subconjuntos rápidos en `i`

#### -- Subconjunto de todas las filas donde el aeropuerto de origen coincide con *"JFK"* usando `on`


``` r
flights["JFK", on = "origin"]
#         year month   day dep_delay arr_delay carrier origin   dest air_time distance  hour
#        <int> <int> <int>     <int>     <int>  <char> <char> <char>    <int>    <int> <int>
#     1:  2014     1     1        14        13      AA    JFK    LAX      359     2475     9
#     2:  2014     1     1        -3        13      AA    JFK    LAX      363     2475    11
#     3:  2014     1     1         2         9      AA    JFK    LAX      351     2475    19
#     4:  2014     1     1         2         1      AA    JFK    LAX      350     2475    13
#     5:  2014     1     1        -2       -18      AA    JFK    LAX      338     2475    21
#    ---                                                                                    
# 81479:  2014    10    31        -4       -21      UA    JFK    SFO      337     2586    17
# 81480:  2014    10    31        -2       -37      UA    JFK    SFO      344     2586    18
# 81481:  2014    10    31         0       -33      UA    JFK    LAX      320     2475    17
# 81482:  2014    10    31        -6       -38      UA    JFK    SFO      343     2586     9
# 81483:  2014    10    31        -6       -38      UA    JFK    LAX      323     2475    11

## alternatively
# flights[.("JFK"), on = "origin"] (or)
# flights[list("JFK"), on = "origin"]
```

* Esta instrucción también realiza una búsqueda binaria rápida basada en subconjuntos, calculando el índice sobre la marcha. Sin embargo, tenga en cuenta que no guarda el índice como un atributo automáticamente. Esto puede cambiar en el futuro.

* Si ya hubiéramos creado un índice secundario, utilizando `setindex()`, entonces `on` lo reutilizaría en lugar de (re)computarlo. Podemos ver esto utilizando `verbose = TRUE`:

    
    ``` r
    setindex(flights, origin)
    flights["JFK", on = "origin", verbose = TRUE][1:5]
    # i.V1 has same type (character) as x.origin. No coercion needed.
    # on= matches existing index, using index
    # Starting bmerge ...
    # <forder.c>: recibió 1 filas y 1 columnas
    # forderReuseSorting: opt=-1, took 0.000s
    # bmerge: looping bmerge_r took 0.000s
    # bmerge: took 0.000s
    # bmerge done in 0.000s elapsed (0.000s cpu)
    # Constructing irows for '!byjoin || nqbyjoin' ... 0.000s elapsed (0.000s cpu)
    #     year month   day dep_delay arr_delay carrier origin   dest air_time distance  hour
    #    <int> <int> <int>     <int>     <int>  <char> <char> <char>    <int>    <int> <int>
    # 1:  2014     1     1        14        13      AA    JFK    LAX      359     2475     9
    # 2:  2014     1     1        -3        13      AA    JFK    LAX      363     2475    11
    # 3:  2014     1     1         2         9      AA    JFK    LAX      351     2475    19
    # 4:  2014     1     1         2         1      AA    JFK    LAX      350     2475    13
    # 5:  2014     1     1        -2       -18      AA    JFK    LAX      338     2475    21
    ```

#### -- ¿Cómo puedo crear subconjuntos basados en las columnas `origen` *y* `dest`?

Por ejemplo, si queremos crear un subconjunto de la combinación `"JFK", "LAX"`, entonces:


``` r
flights[.("JFK", "LAX"), on = c("origin", "dest")][1:5]
#     year month   day dep_delay arr_delay carrier origin   dest air_time distance  hour
#    <int> <int> <int>     <int>     <int>  <char> <char> <char>    <int>    <int> <int>
# 1:  2014     1     1        14        13      AA    JFK    LAX      359     2475     9
# 2:  2014     1     1        -3        13      AA    JFK    LAX      363     2475    11
# 3:  2014     1     1         2         9      AA    JFK    LAX      351     2475    19
# 4:  2014     1     1         2         1      AA    JFK    LAX      350     2475    13
# 5:  2014     1     1        -2       -18      AA    JFK    LAX      338     2475    21
```

* El argumento `on` acepta un vector de caracteres de nombres de columnas correspondientes al orden proporcionado a `i-argument`.

* Dado que el tiempo para calcular el índice secundario es bastante pequeño, no tenemos que usar `setindex()`, a menos que, una vez más, la tarea implique subconjuntos repetidos en la misma columna.

### b) Seleccionar en `j`

Todas las operaciones que analizaremos a continuación no son diferentes a las que ya vimos en la viñeta *Subconjunto basado en claves y búsqueda binaria rápida*. Excepto que usaremos el argumento `on` en lugar de establecer claves.

#### -- Devuelve la columna `arr_delay` sola como una tabla de datos correspondiente a `origin = "LGA"` y `dest = "TPA"`


``` r
flights[.("LGA", "TPA"), .(arr_delay), on = c("origin", "dest")]
#       arr_delay
#           <int>
#    1:         1
#    2:        14
#    3:       -17
#    4:        -4
#    5:       -12
#   ---          
# 1848:        39
# 1849:       -24
# 1850:       -12
# 1851:        21
# 1852:       -11
```

### c) Encadenamiento

#### -- Sobre el resultado obtenido anteriormente, utilice el encadenamiento para ordenar la columna en orden decreciente.


``` r
flights[.("LGA", "TPA"), .(arr_delay), on = c("origin", "dest")][order(-arr_delay)]
#       arr_delay
#           <int>
#    1:       486
#    2:       380
#    3:       351
#    4:       318
#    5:       300
#   ---          
# 1848:       -40
# 1849:       -43
# 1850:       -46
# 1851:       -48
# 1852:       -49
```

### d) Calcular o *hacer* en `j`

#### -- Encuentra el retraso máximo de llegada correspondiente a `origin = "LGA"` y `dest = "TPA"`.


``` r
flights[.("LGA", "TPA"), max(arr_delay), on = c("origin", "dest")]
# [1] 486
```

### e) *sub-asignar* por referencia usando `:=` en `j`

Ya hemos visto este ejemplo en la viñeta *Semántica de referencia* y *Claves y subconjunto basado en búsqueda binaria rápida*. Echemos un vistazo a todas las `horas` disponibles en la *tabla de datos* `vuelos`:


``` r
# get all 'hours' in flights
flights[, sort(unique(hour))]
#  [1]  0  1  2  3  4  5  6  7  8  9 10 11 12 13 14 15 16 17 18 19 20 21 22 23 24
```

Vemos que hay un total de `25` valores únicos en los datos. Parece que están presentes tanto *0* como *24* horas. Reemplacemos *24* por *0*, pero esta vez usemos `on` en lugar de claves de configuración.


``` r
flights[.(24L), hour := 0L, on = "hour"]
# Índices: <origin>, <origin__dest>
#          year month   day dep_delay arr_delay carrier origin   dest air_time distance  hour
#         <int> <int> <int>     <int>     <int>  <char> <char> <char>    <int>    <int> <int>
#      1:  2014     1     1        14        13      AA    JFK    LAX      359     2475     9
#      2:  2014     1     1        -3        13      AA    JFK    LAX      363     2475    11
#      3:  2014     1     1         2         9      AA    JFK    LAX      351     2475    19
#      4:  2014     1     1        -8       -26      AA    LGA    PBI      157     1035     7
#      5:  2014     1     1         2         1      AA    JFK    LAX      350     2475    13
#     ---                                                                                    
# 253312:  2014    10    31         1       -30      UA    LGA    IAH      201     1416    14
# 253313:  2014    10    31        -5       -14      UA    EWR    IAH      189     1400     8
# 253314:  2014    10    31        -8        16      MQ    LGA    RDU       83      431    11
# 253315:  2014    10    31        -4        15      MQ    LGA    DTW       75      502    11
# 253316:  2014    10    31        -5         1      MQ    LGA    SDF      110      659     8
```

Ahora, verifiquemos si `24` se reemplaza con `0` en la columna `hora`.


``` r
flights[, sort(unique(hour))]
#  [1]  0  1  2  3  4  5  6  7  8  9 10 11 12 13 14 15 16 17 18 19 20 21 22 23
```

* Esta es una gran ventaja de los índices secundarios. Antes, para actualizar unas pocas filas de `hour`, teníamos que ejecutar `setkey()` sobre él, lo que inevitablemente reordenaba toda la tabla de datos. Con `on`, se conserva el orden y la operación es mucho más rápida. Si observamos el código, la tarea que queríamos realizar también es bastante clara.

### f) Agregación utilizando `por`

#### -- Obtener el retraso máximo de salida para cada `mes` correspondiente a `origen = "JFK"`. Ordenar el resultado por `mes`


``` r
ans <- flights["JFK", max(dep_delay), keyby = month, on = "origin"]
head(ans)
# Key: <month>
#    month    V1
#    <int> <int>
# 1:     1   881
# 2:     2  1014
# 3:     3   920
# 4:     4  1241
# 5:     5   853
# 6:     6   798
```

* Tendríamos que haber establecido la `clave` nuevamente en `origen, destino`, si no usáramos `on`, que internamente construye índices secundarios sobre la marcha.

### g) El argumento *mult*

Los demás argumentos, incluido `mult`, funcionan exactamente de la misma manera que vimos en la viñeta *Subconjunto basado en claves y búsqueda binaria rápida*. El valor predeterminado para `mult` es "all". Podemos elegir, en lugar de eso, solo se deben devolver las "primeras" o "últimas" filas coincidentes.

#### -- Subconjunto solo de la primera fila coincidente donde `dest` coincide con *"BOS"* y *"DAY"*


``` r
flights[c("BOS", "DAY"), on = "dest", mult = "first"]
#     year month   day dep_delay arr_delay carrier origin   dest air_time distance  hour
#    <int> <int> <int>     <int>     <int>  <char> <char> <char>    <int>    <int> <int>
# 1:  2014     1     1         3         1      AA    JFK    BOS       39      187    12
# 2:  2014     1     1        25        35      EV    EWR    DAY      102      533    17
```

#### -- Subconjunto solo de la última fila coincidente donde `origin` coincide con *"LGA", "JFK", "EWR"* y `dest` coincide con *"XNA"*


``` r
flights[.(c("LGA", "JFK", "EWR"), "XNA"), on = c("origin", "dest"), mult = "last"]
#     year month   day dep_delay arr_delay carrier origin   dest air_time distance  hour
#    <int> <int> <int>     <int>     <int>  <char> <char> <char>    <int>    <int> <int>
# 1:  2014    10    31        -5       -11      MQ    LGA    XNA      165     1147     6
# 2:    NA    NA    NA        NA        NA    <NA>    JFK    XNA       NA       NA    NA
# 3:  2014    10    31        -2       -25      EV    EWR    XNA      160     1131     6
```

### h) El argumento *nomatch*

Podemos elegir si las consultas que no coinciden deben devolver "NA" o ignorarse por completo utilizando el argumento "nomatch".

#### -- Del ejemplo anterior, crea un subconjunto de todas las filas solo si hay una coincidencia


``` r
flights[.(c("LGA", "JFK", "EWR"), "XNA"), mult = "last", on = c("origin", "dest"), nomatch = NULL]
#     year month   day dep_delay arr_delay carrier origin   dest air_time distance  hour
#    <int> <int> <int>     <int>     <int>  <char> <char> <char>    <int>    <int> <int>
# 1:  2014    10    31        -5       -11      MQ    LGA    XNA      165     1147     6
# 2:  2014    10    31        -2       -25      EV    EWR    XNA      160     1131     6
```

* No hay vuelos que conecten "JFK" y "XNA". Por lo tanto, esa fila se omite en el resultado.

## 3. Indexación automática

Primero, analizamos cómo crear subconjuntos rápidos mediante búsqueda binaria con *claves*. Luego, descubrimos que podíamos mejorar aún más el rendimiento y tener una sintaxis más clara utilizando índices secundarios.

Esto es lo que hace la *indexación automática*. Por el momento, solo está implementada para los operadores binarios `==` y `%in%`. Se crea un índice automáticamente *y* se guarda como un atributo. Es decir, a diferencia del argumento `on` que calcula el índice sobre la marcha cada vez (a menos que ya exista uno), aquí se crea un índice secundario.

Comencemos creando una tabla de datos lo suficientemente grande para resaltar la ventaja.


``` r
set.seed(1L)
dt = data.table(x = sample(1e5L, 1e7L, TRUE), y = runif(100L))
print(object.size(dt), units = "Mb")
# 114.4 Mb
```

Cuando usamos `==` o `%in%` en una sola columna por primera vez, se crea automáticamente un índice secundario y se utiliza para realizar el subconjunto.


``` r
## have a look at all the attribute names
names(attributes(dt))
# [1] "names"             "row.names"         "class"             ".internal.selfref"

## run thefirst time
(t1 <- system.time(ans <- dt[x == 989L]))
#    user  system elapsed 
#    0.34    0.01    0.53
head(ans)
#        x         y
#    <int>     <num>
# 1:   989 0.7757157
# 2:   989 0.6813302
# 3:   989 0.2815894
# 4:   989 0.4954259
# 5:   989 0.7885886
# 6:   989 0.5547504

## secondary index is created
names(attributes(dt))
# [1] "names"             "row.names"         "class"             ".internal.selfref"
# [5] "index"

indices(dt)
# [1] "x"
```

El tiempo necesario para crear el subconjunto la primera vez es el tiempo necesario para crear el índice + el tiempo necesario para crear el subconjunto. Dado que crear un índice secundario implica únicamente la creación del vector de orden, esta operación combinada es más rápida que los escaneos de vectores en muchos casos. Pero la verdadera ventaja se encuentra en los subconjuntos sucesivos, ya que son extremadamente rápidos.


``` r
## successive subsets
(t2 <- system.time(dt[x == 989L]))
#    user  system elapsed 
#    0.00    0.00    0.01
system.time(dt[x %in% 1989:2012])
#    user  system elapsed 
#    0.01    0.00    0.03
```

* La primera ejecución tardó 0.530 segundos, mientras que la segunda vez tardó 0.010 segundos.

* La indexación automática se puede desactivar configurando el argumento global `options(datatable.auto.index = FALSE)`.

* Deshabilitar la indexación automática aún permite usar índices creados explícitamente con `setindex` o `setindexv`. Puede deshabilitar los índices por completo configurando el argumento global `options(datatable.use.index = FALSE)`.

#

En la versión reciente, ampliamos la indexación automática a expresiones que involucran más de una columna (combinadas con el operador `&`). En el futuro, planeamos ampliar la búsqueda binaria para que funcione con más operadores binarios como `<`, `<=`, `>` y `>=`.

Discutiremos *subconjuntos* rápidos usando claves e índices secundarios para *uniones* en la siguiente viñeta, *"Uniones y uniones continuas"*.

***




