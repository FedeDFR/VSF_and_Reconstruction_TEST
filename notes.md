# Notas sobre el codigo de CosmoBologna

# VoidCatalogue.cpp

La funcion Catalogue toma varios inputs entre otros los mas importantes.

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

###  Es  `algorithm==VoidAlgorithm::_LaZeVo_`

1. Observa si el `random_catalogue` es vacio, si es asi, genera un nuevo catalogo random uniforme mediante `Catalogue` (linea 81) donde se le pasa como parametro `create_Random_box`. Esta definido en el archivo RandomCatalogue.cpp de la linea 88 a la 118.

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


2. Si `random_catalogue` tiene distinta cantidad de objetos que `tracer_catalogue` le saca o elimina objetos a `random_catalogue`(linea 96). El metodo que se encarga de esto esta definido en RandomCatalogue.cpp de la linea 694 a la 725.

Linea 96
```c++
random_catalogue.equalize_random_box(tracer_catalogue, rndd);
```

3. Si tienen la misma cantidad no hace nada.

4. Hace dos punteros, uno a el catalogo de random y al de tracer. Define un vector que guardara las indexacion a tracer, y lo llena con valores consecutivos ordenados desde 0 (linea 104).

5. Define 2 ChainMesh, una para el catalogo de random y para el de tracer.

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
    std::shared_ptr<cbl::catalogue::Catalogue> cat, std::shared_ptr<cbl::catalogue::Catalogue> cat2)
```

ChainMesh esta definido en CatalogueChainMesh.cpp desdey este toma 4 parametros.
* `variables`: Un set de variables, en este caso son las cordenadas x, y, z.
* `cellsize`: El tamaño de los bloques.
* `cat`: Un catalogo.
* `cat2`: Otro catalogo, en este caso no usamos este parametro.

Basicamente se encarga de construir una estructura donde divide el espacio en celdas y guarda los puntos que estan en ese espacio. En esta estructura tambien se guardan las celdas vecinas a cada celda.





