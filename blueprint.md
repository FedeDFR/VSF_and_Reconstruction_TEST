# Modulo de Reconstruccion LAZEVO

El modulo de reconstruccion LAZEVO tiene como objetivo una asignacion de pare 
entre trazadores del catalogo y un catalogo de randoms.

    1. Entradas : trazadores, randoms (Mismo shape)
    2. Salida : Esquema de Asignacion indices (? - ver esto)


## Resumen de funcionamiento

Empezamos teniendo 2 tipos de trazadores:

    - Los del catalogo (T)
    - Conjunto de trazadores aleatorios (A).


### Proceso

    1. Se parte de un emparejamiento inicial de los objetos. La idea de de este
    paso es que sea un punto de partida “bueno” aunque no seria el mas optimo.
    En pasos posteriores este emparejamiento se refina.
    
    Para realizar el emparejamiento inicial, se toma un punto ficticio P.
    Alrededor de ese punto P, se buscan todos los puntos trazadores T (No emparejados), que se encuentran a una distancia máxima de 4 veces la distancia media entre partículas.
    
    El limite maximo de los trazadores que se pueden incorporar dentro de esta region es 100 (Creo que se pueden Fijar). Posteriormente, se buscan todos los aleatorios A (No emparejados) que esten alrededor de P y a la misma distancia, tambien imponiendo un maximo. 
    
    Luego, aleatoriamente emparejamos los trazadores en esta region. Este proceso idealmente sigue hasta conseguir emparejar todos los trazadores. Con esto conseguimos un emparejamiento “local de objetos”.


    2. Seguidamente viene el refinamiento por cuartetos. Tomamos los trazadores T, hacemos un shuffling de sus indices (aleatorio).
    
    En orden recorremos los trazadores tomando a cada uno que elijamos como semilla. Para una dada semilla T_i, se elegiran 4 vecinos cercanos: T_i_1, T_i_2, T_i_3. Estos trazadores constituyen un cuarteto (A los cuales no olvidemos tenemos asociado sus pares aleatorios). Estos trazaodres reciben un flag de ya no estar libres o disponibles para realizar cuartetos. Se realizan 4! permutaciones de trazadores T con A computando cada vez la distancia entre estos y registrando la distancia total T_max. La idea es que la permutacion correcta es la que de la suma minima. Se calcula cuantos cuartetos se modificaron. Y si la cantidad de modificaciones supera cierto umbral se repite el paso de asignacion de cuartetos. Este paso termina cuando practicamente ningun cuarteto cambia.