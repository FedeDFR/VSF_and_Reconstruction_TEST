# Notas sobre el codigo de CosmoBologna

Aca voy a ir escribiendo todas las notas sobre el codigo de la libreria CosmoBologna. El numero al inicio de cada parrafo hace referencia a la linea de la cual se esta hablando.

# VoidCatalogue.cpp

El constructor Catalogue toma varios parametros entre otros los mas importantes.

* `algorithm`: Es un enum que decide si se ejecuta LaZeVo o Exact.
* `tracer_catalogue` y `random_catalogue`: la muestra a analizar y un catalogo de muestas aleatorio respectivamente.
* `cellsize` El tamaño de los bloques. (se utiliza en ChainMesh).
* `n_rec`: numero de reconstrucciones.
* `stepsize`: ???
* `threshold`: valor de convergencia para el algoritmo. (valor para limitar la cantidad de cambios en el algoritmo).

```c++
cbl::catalogue::Catalogue::Catalogue (
    const VoidAlgorithm algorithm,
    Catalogue tracer_catalogue,
    Catalogue random_catalogue,
    const std::string dir_output,
    const std::string output,
    const double cellsize,
    const int n_rec,
    const double step_size,
    const double threshold,
    const std::vector<bool> print)

```

## Primer etapa

Inicializa variables

```c++
const double start_time = omp_get_wtime();
shared_ptr<Catalogue> tracer_cat, random_cat;
vector<Catalogue> displacement_catalogue(n_rec);
vector<vector<unsigned int>> randoms;
cosmology::Cosmology cosm;
CatalogueChainMesh ChainMesh_tracers;
CatalogueChainMesh ChainMesh_randoms;
```

despues inicia la reconstrucción que depende del valor de `algorithm`.

##  Es  `algorithm==VoidAlgorithm::_LaZeVo_`

1. (80-83) Observa si el `random_catalogue` es vacio, si es asi, genera un nuevo catalogo random uniforme mediante `Catalogue` (linea 81) donde se le pasa como parametro `create_Random_box`. Esta definido en el archivo RandomCatalogue.cpp de la linea 88 a la 118.

Linea 81
```c++
random_catalogue = Catalogue(RandomType::_createRandom_box_, tracer_catalogue, 1., 10, cosm, false, 10., {}, {}, {}, 10, rndd);
```

```c++
Catalogue(
  const RandomType type,
  const Catalogue catalogue,
  const double N_R,
  const int nbin,
  const cosmology::Cosmology &cosm,
  const bool conv,
  const double sigma,
  const std::vector<double> redshift,
  const std::vector<double> RA,
  const std::vector<double> Dec,
  const int z_ndigits,
  const int seed)
```

* `type`: tipo de catálogo random a generar (`_createRandom_box_`, etc.)
* `catalogue`: catálogo original usado como referencia geométrica/distribución
* `N_R`: factor multiplicativo del número de objetos random (`nRandom = N_R * catalogue.nObjects()`).
* `nbin`:número de bins usados en histogramas/distribuciones internas
* `cosm`:objeto cosmológico con parámetros cosmológicos y conversiones
* `conv`:indica si se deben convertir coordenadas/redshift
* `sigma`:parámetro de suavizado o dispersión usado en algunos random catalogues
* `redshift`:vector opcional de redshifts observados
* `RA`:vector opcional de ascensión recta
* `Dec`:vector opcional de declinaciones
* `z_ndigits`:cantidad de dígitos usados para discretizar/redondear redshift
* `seed`:semilla del generador de números aleatorios

2. (84) Si tienen la misma cantidad no hace nada.

1. (88-97) Si `random_catalogue` tiene distinta cantidad de objetos que `tracer_catalogue` le saca o elimina objetos a `random_catalogue`(linea 96). El metodo que se encarga de esto esta definido en RandomCatalogue.cpp de la linea 694 a la 725.

Linea 96
```c++
random_catalogue.equalize_random_box(tracer_catalogue, rndd);
```

4. (99-100) Hace dos punteros, uno a el catalogo de random y al de tracer. Define un vector que guardara las indexacion a tracer, y lo llena con valores consecutivos ordenados desde 0 (linea 104).

### ChainMesh setting

5. (106-110) Define 2 ChainMesh, una para el catalogo de random y para el de tracer.

Linea 107
```c++
vector<Var> variables = {Var::_X_, Var::_Y_, Var::_Z_};
	ChainMesh_tracers = CatalogueChainMesh(variables, cellsize, tracer_cat);
	ChainMesh_randoms = CatalogueChainMesh(variables, cellsize, random_cat);
```

```c++
CatalogueChainMesh (
    std::vector<catalogue::Var> variables,
    double cellsize,
    std::shared_ptr<cbl::catalogue::Catalogue> cat,
    std::shared_ptr<cbl::catalogue::Catalogue> cat2)
```

ChainMesh esta definido en CatalogueChainMesh.cpp desdey este toma 4 parametros.
* `variables`: Un set de variables, en este caso son las cordenadas x, y, z.
* `cellsize`: El tamaño de los bloques.
* `cat`: Un catalogo.
* `cat2`: Otro catalogo, en este caso no usamos este parametro.

Basicamente se encarga de construir una estructura donde divide el espacio en celdas y guarda los puntos que estan en ese espacio. En esta estructura tambien se guardan las celdas vecinas a cada celda.

### Searching close particles

6. (114) Declaro un vector de vectores que va a guardar por cada particula las `N_near_obj` (113) particulas mas cercanas a cada una.

Linea 114
```c++
 vector<vector<unsigned int>> near_part = ChainMesh_tracers.N_nearest_objects_cat(N_near_obj);
```

7. (116-120) Completo el vector de vectores `random` con numeros de 0 hasta la cantidad de particulas que tiene el catalgo tracer ordenados de formla aleatoria.

### Setting starting configuration

8. (125-128) Calcula los limites del box en todas las dimensiones y define a `dist` como la media de distancia entre particulas en tracer por 4.

9. (130-162) Comienza a generar `n_rec` de veces el primer macheo entre tracer con random. Define el rango de los numeros aleatorios para cada dimension, osea que el numero aleatorio para la poscion en x es desde `minX` hasta `maxX`.

(137) Crea una copia de las chain mesh de tracer y catalogo.

(141-142) Ahora va agenerar tantos macheos como cantidad de particulas tenga el catalogo de tracer. Empieza eligiendo una posicion `pos` aleatoria en plano. Desde la copia de la chain mesh de tracer tomo todas las particulas cercanas a `pos` a una distancia a lo sumo de `dist` y lo guarda en `close`.

(143) Toma el minimo entre 100 y la cantidad de particulas en `colse` y lo guarda en `tr_to_rmv`.

(144-145) Si tengo particulas en `close` empiezo tomando desde la copia de chain mesh random la misma cantidad de particulas cercanas a `pos` que tengo en `close` y lo guarda en `close_random`.

(150-158) Elige una particula aleatoria i de `close` y otra j de `close_random` y termina guardando de tal forma que queda en cada posicion de `randoms` es equivalente a una particula en tracer y lo que guarda ahi se refiere a una particula de random.

```c++
randoms[rec][i] = j
```

Despues de esta asignacion va eliminando las particulas utilizadas de su correspondiente copias de chain mesh, asi no se repiten las mismas particulas posteriormente y repite el macheo `tr_to_rmv` cantidad de veces y luego vuelve a la linea 141.

Linea 150-158
```c++
while (tr_to_rmv > 0) {
            std::uniform_int_distribution<std::mt19937::result_type> dist(0, tr_to_rmv-1);
            int rnd1 = dist(rng), rnd2 = dist(rng);
            randoms[rec][close[rnd1]] = close_random[rnd2];
            ChM_tracer_copy.deletePart(close[rnd1]);
            ChM_random_copy.deletePart(close_random[rnd2]);
            close.erase(close.begin()+rnd1);
            close_random.erase(close_random.begin()+rnd2);
            tr_to_rmv--;
          }
```

### Performing the iterations

10. (168–267) Para cada reconstrucción, el algoritmo intenta establecer una correspondencia (matching) entre las partículas del catálogo aleatorio (random) y las partículas trazadoras (tracers). Este proceso se realiza iterativamente, intercambiando asociaciones entre partículas con el objetivo de minimizar la distancia total entre los pares emparejados. Las iteraciones continúan hasta que la variación media producida por estos intercambios es menor o igual que el umbral especificado, indicando que se ha alcanzado una configuración suficientemente estable y cercana al mínimo de distancia global.

(173) Entra al while donde va a repetir este remacheo hasta cumplir con el humbral

(176-180) Mezcla un vector que tiene numeros desde el 0 a el numero de particulas en el catalogo tracer. Define una funcion uniforme que toma numeros aleatorios desde 1 hasta `N_near_obj`-1. Y define dos vectores de booleanos en cual `index_bool` guardara si se realizo alguma permutacion en el macheo y `used` se utilizara para marcar las particulas que se esten utilizando asi cuando se paralelice no ocurran condiciones de carrera y dos hilos distintos tomen una misma particula.

(182-230) Se paraleliza el codigo de tal forma que se divide el siguiente for en la cantidad de hilos que tenga el procesador. Osea si tenemos 4 hilos y `num_objects` = 100 tendremos 4 ciclos en paralelo donde uno va de la particula 0 a la 24, otro de la 25 a la 49, otro de la 50 a la 74 y otro de la 75 a la 99.

(185) El ciclo for pasa por todas las `i` particulas del catalogo tracer.

(187-191) Tomo 3 posiciones aleatorios entre 1 y `N_near_obj`-1. Si alguno es igual a otro los cambio hasta tener 3 distintos.

(192) Chequeo si alguna de las particulas cercanas a `i` que se encuentran en las posiciones aleatorias y en 0 se esta utilizando.

```c++
if (used[near_part[index_tracer_cat_copy[i]][0]] == false &&
    used[near_part[index_tracer_cat_copy[i]][rand1]] == false &&
    used[near_part[index_tracer_cat_copy[i]][rand2]] == false &&
    used[near_part[index_tracer_cat_copy[i]][rand3]] == false)
```

(194-205) Si no es asi, las marco. Las guardo y tambien guardo a la particula que tiene cada una macheada en random.

(210-219) Ahora va a calcular la distancia que hay entre cada macheo y luego va comparar esta distancia con la distancia que habria permutando los macheos. Ejemplo si tengo las particulas 1, 2 y 3 de tracer y las particulas a, b y c de random calculo la distancia que hay entre 1 con a, 2 con b y 3 con c, luego veo la distancia de 1 con b, 2 con c y 3 con a y asi consecutivamente hasta encontrar la menor distancia.

```c++
do {
              dist = 0.;
              for (size_t j=0; j<R.size(); j++)
                dist += cbl::Euclidean_distance(tracer_cat->xx(H[j]), random_cat->xx(R[j]), tracer_cat->yy(H[j]), random_cat->yy(R[j]), tracer_cat->zz(H[j]), random_cat->zz(R[j]));

                if (dist < dist_min) {
                dist_min = dist;
                R_def = R;
              }
            } while(std::next_permutation(R.begin(), R.end()));
```

Si el macheo es distinto al original lo guardo en `index_bool` y actualizo el macheo que habia entre particulas de tracer y de random en el vector `randoms`. Por ultimo desmarco las particulas que estoy usando.

(232-237) Se calcula el ratio de cambios en el macheo e imprime toda la informacion.

(241-248) Crea un vector de objetos en cual va a guardar el vector de desplasamiento de cada particula del catalogo de tracer. Y lo termina guardando en el vector de catalogos `displacement_catalogue` el cual para cada reconstruccion va a tener su catalogo de vectores de desplazamiento.

El output de este primer paso es `displacement_catalogue`.

### End of LaZeVo method

## Es `algorithm==VoidAlgorithm::_Exact_`

...

## Segunda etapa

...






# CatalogueChainMesh.cpp

## La clase CatalogueChainMesh

La clase CatalogueChainMesh tiene 5 atributos:

* `m_cellsize`: El tamaño de las celdas.
* `m_lim`: Vector que guarda los limites del catalogo en cada dimension.
* `m_part_catalogue`: Puntero al catalogo en la chain-mesh.
* `m_part_catalogue2`: Puntero al segundo catalogo.
* `m_dimension`: Cantidad de celdas por lado.

## El constructor de CatalogueChainMesh

* `variables`: Un vector del enum Var, este enum contiene distintos tipos de variables, sean coordenadas x, y, z, Redshift, entre otros, en este caso se usan las coordenadas.
* `cellsize`: Es el maximo tamaño de celda que va a tener el grillado cuando se cree.
* `cat`: Un catalogo
* `cat2`: Otro catalogo

```c++
CatalogueChainMesh (
	std::vector<catalogue::Var> variables,
	double cellsize,
	std::shared_ptr<cbl::catalogue::Catalogue> cat, std::shared_ptr<cbl::catalogue::Catalogue> cat2)
```

### Construccion

1. Inicializa variables

```c++
m_part_catalogue = cat;
m_part_catalogue2 = cat2;
unsigned int nDim=variables.size();
vector<vector<double>> data(nDim);
for (size_t i=0; i<nDim; i++) data[i] = m_part_catalogue->var(variables[i]);
vector<vector<double>> data2={};
if (m_part_catalogue2!=NULL) {
	data2.resize(nDim, vector<double>(m_part_catalogue2->var(variables[0])));
	for (size_t i=0; i<nDim; i++) data2[i] = m_part_catalogue2->var(variables[i]);
}
```

data va a ser un vector donde va a guardar toda la informacion de cada una de las coordenadas

2. Continua buscando en cada variable su valor maximo y minimo y lo guarda en `m_lim`. Tambien lo hace en el segundo catalogo si fue dado.

```c++
vector<unsigned int> num_cells(nDim);
vector<double> delta(nDim);
double start_cellsize = cellsize;
m_lim.resize(nDim);
for (size_t i=0; i<nDim; i++) {
	m_lim[i].resize(2);
	m_lim[i][0] = *min_element(data[i].begin(), data[i].end());
	m_lim[i][0] = (m_part_catalogue2!=NULL) ? min(m_lim[i][0], *min_element(data2[i].begin(), data2[i].end()))-0.05*cellsize :  m_lim[i][0]-0.05*cellsize;
	m_lim[i][1] = *max_element(data[i].begin(), data[i].end());
	m_lim[i][1] = (m_part_catalogue2!=NULL) ? max(m_lim[i][1], *max_element(data2[i].begin(), data2[i].end()))+0.05*cellsize :  m_lim[i][1]+0.05*cellsize;
	delta[i] = m_lim[i][1]-m_lim[i][0];
}
```

En delta se guarda el diferencial entre los puntos mas alejados en esa dimension.

3. Comienza a definir el tamaño de las celdas en funcion a la dimension mas grande, osea con las particulas mas alejadas. Voy dividiendo esta dimesion hasta conseguir un tamaño de celda menor o igual al de `cellsize` que pasamos como parametro.

```c++
unsigned int nCells=0;
double delta_max = *max_element(delta.begin(),delta.end());
cellsize = delta_max;
vector<vector<unsigned int>> cells(1);

while(cellsize > start_cellsize) {
	nCells++;
	cellsize = delta_max/nCells;
	cells.resize((int)pow(nCells, nDim));
}
```

Y luego se guarda como atributo de la clase el `nCells` y `cellsize` nuevo.

4. Se calcula un index para cada celda. Y guarda en el vector `cells` la respectiva particula.

```c++
  for (unsigned int i=0; i<data[0].size(); i++) {
    unsigned int index=0;
    for (unsigned int j=0; j<nDim; j++) index += ((unsigned int)((data[j][i] - m_lim[j][0])/cellsize))*pow(nCells,nDim-j-1);
    cells[index].emplace_back(i);
  }
```

Se normaliza cada componente con el minimo de cada dimension y luego se componen utilizando potencias de `nCells`.


5. Para cada celda creo un vector `nearcells` donde calcula la distancia con cada una de las otras celdas. El index de `nearcells` va a indicar que tan lejos se encuentra la celda.

```c++
  for (size_t i=0; i<cells.size(); i++) {
    vector<vector<unsigned int>> nearCells(dim_nearCells);
    for (size_t j=0; j<cells.size(); j++) {
      double sum = 0;
      unsigned int index1 = i, index2 = j;
      for (unsigned int k=0; k<nDim; k++) {
        double temp_dist = (double)(index1%nCells)-(double)(index2%nCells);
        if ((int)temp_dist != 0) temp_dist -= 0.5;
        sum += temp_dist*temp_dist;
        index1 = (int)(index1/nCells);
        index2 = (int)(index2/nCells);
      }
      nearCells[(int)sqrt(sum)].emplace_back(j);
    }
    check_memory(8.0, true, "cbl::catalogue::Catalogue::Catalogue of ChainMeshCatalogue.cpp. Increase cellsize");
    add_object(move(Object::Create(i, cells[i], nearCells)));
  }
```
La distancia se calculca reconstruyendo cada componente de la cordenada a partir de los index. Como sabemos el index esta construido en base a `nCells` por lo tanto, calcula el modulo de los index con `nCells` y obtiene el valor de cada componente, resta los valores dados de la componente de cada indice, luego lo eleva al cuadrado y lo suma con el resto de de cuadrados de las diferencias de componentes. Para obtener las otras componentes lo que hace es divid el index con `nCells` y repetir el proceso anteriormente descrito hasta hacerlo con todas las dimensiones. Una vez realizado todos los calculos con todas las componentes se calcula la raiz cuadrada de la suma y esto dara con la distancia entre las celdas, y segun esta distancia se guarda en `nearCells`.

![Distancia euclidea](Imagenes/Distancia_euclidea.png)

## Funcion N_nearest_objects(obj, N)

Esta funcion busca las N particulas mas cercanas a la particula obj. Como estamos analizando el caso de uso en LaZeVo N = 3*3*4*pi.

### Algoritmo

1. Define la posicion de la particula y calcula el indice de celda `center_index` en el que se encuentra la particula obj.

2. Declara en `nCells` el vector que contiene a que distancia esta cada celda de la otra.

```c++
vector<vector<unsigned int>> nCells = nearCells(center_index);
```

3. Entra al while y va a salir hasta tener N objetos.

    1. Actualiza a todas las particulas que no entraban en la anterior capa.

    2. Va observando todos los objetos de las celdas de `nCells`, calcula su distancia y guarda si esta dentro del radio.

      ```c++
      for(unsigned int j : nCells[index]) {
        for (unsigned int k : part(j)) {
          new_distance = Euclidean_distance(pos[0], m_part_catalogue->xx(k), pos[1],  m_part_catalogue->yy(k), pos[2],  m_part_catalogue->zz(k));
          temp_Object.push_back(k);
          temp_distance_obj.push_back(new_distance);
          if(new_distance < radius*(index+1)) mask.emplace_back(true);
          else mask.emplace_back(false);
        }
      }
      ```

    3. Si llego a ver todos los elementos de `nCells` dejo de buscar mas particulas..

    4. Si todavia no llegue a ver todos los elementos de `nCells`, agrego todas las particulas que estaban marcas dentro del radio.

4. Termina devolviendo el vector con el indice de las particulas ordenadas de menor a mayor distancia.