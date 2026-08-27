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

```c++
auto timeS = ((long int)chrono::steady_clock::now().time_since_epoch().count()) %1000000;   //132
    random::UniformRandomNumbers randX(minX, maxX, timeS);  //133
    random::UniformRandomNumbers randY(minY, maxY, timeS+1);    //134
    random::UniformRandomNumbers randZ(minZ, maxZ, timeS+2);    //135
```
    Alrededor de ese punto P, se buscan todos los puntos trazadores T
    (No emparejados), que se encuentran a una distancia máxima de 4 veces la
    distancia media entre partículas.

```c++
vector<unsigned int> close=ChM_tracer_copy.Close_objects(pos, dist);    //143
```

    Donde esta distancia se calcula de la siguiente manera

```c++
double dist = 4*tracer_cat->mps();  //125
```

    El limite maximo de los trazadores que se pueden incorporar dentro de esta
    region es 100 (Creo que se pueden Fijar).

```c++
unsigned int tr_to_rmv = std::min(100, (int)close.size());  //144
```
    Posteriormente, se busca en los aleatorios A (No emparejados) la misma
    cantidad de vecinos encontrados en los trazadores T que esten alrededor
    de P y a la misma distancia.

```c++
vector<unsigned int> close_random =
                ChM_random_copy.N_nearest_objects(pos, close.size());   //146
```

    Luego, aleatoriamente emparejamos los trazadores en esta region. Este
    proceso idealmente sigue hasta conseguir emparejar todos los trazadores.
    Con esto conseguimos un emparejamiento “local de objetos”.

```c++
std::uniform_int_distribution<std::mt19937::result_type> dist(0, tr_to_rmv-1);
            int rnd1 = dist(rng), rnd2 = dist(rng); //152
            randoms[rec][close[rnd1]] = close_random[rnd2]; //153
```

    Dando como resultado una lista en la que cada índice corresponde a un
    trazador T, y almacena la partícula aleatoria A que fue emparejada con
    dicho trazador.

    2. Seguidamente viene el refinamiento por cuartetos. Tomamos los
    trazadores T, hacemos un shuffling de sus indices (aleatorio).

```c++
auto time = (long int)chrono::steady_clock::
                now().time_since_epoch().count()%1000000;   //175
std::shuffle(
        index_tracer_cat_copy.begin(),
        index_tracer_cat_copy.end(),
        default_random_engine(time));   //176
```

    En orden recorremos los trazadores tomando a cada uno que elijamos como
    semilla. Para una dada semilla T_i, se elegiran 4 vecinos cercanos: T_i_1,
    T_i_2, T_i_3. (Los cuales tienen que ser distintos)

```c++
unsigned int rand1 = dist(rng), rand2 = dist(rng), rand3 = dist(rng);   //187
    while (rand1 == rand2 || rand1==rand3 || rand2 == rand3) {  //188
        rand2 = dist(rng);  //189
        rand3 = dist(rng);  //190
    }
```
    Estos trazadores constituyen un cuarteto (A los cuales no
    olvidemos tenemos asociado sus pares aleatorios). Si estos trazaodres estan
    libres entonces reciben un flag de ya no estar libres o disponibles para
    realizar cuartetos.

```c++
if (used[near_part[index_tracer_cat_copy[i]][0]] == false &&
    used[near_part[index_tracer_cat_copy[i]][rand1]] == false &&
    used[near_part[index_tracer_cat_copy[i]][rand2]] == false &&
    used[near_part[index_tracer_cat_copy[i]][rand3]] == false) {
        used[near_part[index_tracer_cat_copy[i]][0]] = true;
        used[near_part[index_tracer_cat_copy[i]][rand1]] = true;
        used[near_part[index_tracer_cat_copy[i]][rand2]] = true;
        used[near_part[index_tracer_cat_copy[i]][rand3]] = true;
        ...
```

    Se realizan 4! permutaciones de trazadores T con A computando cada vez la
    distancia entre estos y registrando la distancia total T_max la cual es la
    la suma de las distancias entre los emparejamientos.

```c++
do {    //210
    dist = 0.;
    for (size_t j=0; j<R.size(); j++)
        dist += cbl::Euclidean_distance(
            tracer_cat->xx(H[j]), random_cat->xx(R[j]), tracer_cat->yy(H[j]), random_cat->yy(R[j]), tracer_cat->zz(H[j]), random_cat->zz(R[j]));

        if (dist < dist_min) {  //215
        dist_min = dist;    //216
        R_def = R;  //217
    }
} while(std::next_permutation(R.begin(), R.end())); //219
```

    Guardando la permutacion con la menor distancia T_max. Y libera a los
    trazadores utilizados.

```c++
if(R_def != R_copy) index_bool[i] = true;
for (size_t j=0; j<4; j++) randoms[rec][H[j]] = R_def[j];
used[H[0]] = false;
used[H[1]] = false;
used[H[2]] = false;
used[H[3]] = false;
```

    La idea es que si la permutacion correcta (osea con el menor T_max) no es
    la que tenemos guardada desde un principio, se cambia por la optima y se
    cuenta la modificacion. Si al finalizar todo el recorrido por los trazadores
    T la cantidad de modificaciones supera cierto umbral se repite el paso de
    asignacion de cuartetos. Este paso termina cuando practicamente ningun
    cuarteto cambia.

```c++
 unsigned int changed_couples = count(index_bool.begin(),
                                     index_bool.end(),
                                     true); //232
ratio = (double)changed_couples/num_objects;    //233
```

    El while que chequea esta condicion es

```c++
while (ratio > threshold) { //173
```

    Como resultado tenemos la misma lista del paso anterior, con las distancias
    minimizadas entre los trazadores T y los aleatorios A emparejados.


### Aclaracion

    Todo este proceso de reconstruccion se realiza K cantidad de veces para mas
    adelante calucular la media de divergencia.