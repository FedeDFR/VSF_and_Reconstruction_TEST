# Configuracion del codigo deacuerdo al Ejemplpo/Script de lanzamiento

```c
cbl::catalogue::Catalogue tracer_catalogue {
    cbl::catalogue::ObjectType::_Halo_,
    cbl::CoordinateType::_comoving_,
    <span style="color:red">var_names_tracers</span>,
    columns_tracers,
    {file_tracers},
    0,
    nSub,
    scaleFact
    };
```


# Pasos para usar el Ejemplo de BIT

- Cargo modulos: 

```bash
module load gcc gsl hdf5 openmpi cfitsio fftw boost
```

## Compilacion del ejemplo

```bash
cd CosmoBolognaLib/CosmoBolognaLib/Examples/cosmicVoids/codes
make
```

### Sobre el Makefile

- Corregir el compilador a 14 o mayor (almenos para el caso de Sersic):
    - FLAGS0 = -std=c++17 -fopenmp

- El objeto 4 es el correspondiente al VF:
```bash
OBJ4 = voidFinder.o
```

- Corregir la receta de compilacion (que tiene OBJ3 en lugar de OBJ4):
```bash
voidFinder: $(OBJ4) 
    	$(CXX) $(OBJ4) -o voidFinder $(FLAGS_LIB)
```

## Salida

```bash

CBL > Reading the catalogue: ../input/halo_catalogue.txt

CBL > > > > > > >>>>>>>>>>   BACK-IN-TIME VOID FINDER   <<<<<<<<<< < < < < <

CBL >                         LaZeVo reconstruction

CBL > * * * I'm creating a random catalogue with cubic geometry * * *

CBL > * * * ChainMesh setting * * *

CBL > * * * Searching close particles * * * 

CBL > * * * Setting starting configuration * * *

CBL > * * * Performing the iterations * * *

CBL > Realisation 5 of 5 | Iteration N°: 2454 - current ratio: 0.000000 - final threshold: 0.000000

CBL > Writing the displacement field in ../output/Displacement_output.dat ...

CBL > * * * Estimation of the Divergence field * * *

CBL > * * * Smoothing * * *

CBL > * * * Identification of voids * * *

CBL > Writing the divergence field in ../output/Divergence_output.dat ...

CBL > Number of cells with negative divergence: 173844
CBL > Number of voids: 919

CBL > * * * Voids rescaling * * *


CBL > Time spent to compute: 616.396409 seconds 
CBL > Time spent to compute: 10.273273 minutes 
CBL > Time spent to compute: 0.171221 hours 
```



# Configuracion del codigo deacuerdo al Ejemplpo/Script de lanzamiento


Esto se utiliza para definir los trazadores, es como definir un Box en VFTK

```c
cbl::catalogue::Catalogue tracer_catalogue {
    cbl::catalogue::ObjectType::_Halo_,
    cbl::CoordinateType::_comoving_,
    var_names_tracers,  # Definida Abajo
    columns_tracers,    # Definida Abajo
    {file_tracers},     # Definida Abajo
    0,
    nSub,
    scaleFact
    };
```



## Configuracion previa requerida

1. Definimos el path donde se ubica el catalogo

```c
const std::string file_tracers = "../input/halo_catalogue.txt";
```

2. Asocia la informacion del catalogo con variables X,Y,Z

```c
std::vector<cbl::catalogue::Var> var_names_tracers = {
    cbl::catalogue::Var::_X_,
    cbl::catalogue::Var::_Y_,
    cbl::catalogue::Var::_Z_
    };
```

3. Indicar que columnas del archivo se van a utilizar:

```c
std::vector<int> columns_tracers = {1, 2, 3};
```



# Firma de la funcion

## En el ejemplo

```c
Catalogue void_catalogue = Catalogue(
    VoidAlgorithm::_LaZeVo_,  # Opciones {"LaZeVo", "Exact"} , ver Catalogue.h
    tracer_catalogue,         # El que definimos mas arriba.
    random_catalogue,         # Otro catalogo random, si no se propociona se construye solo
    dir_output,               # String con el directorio de salida
    output,                   # output suffix
    cellsize,                 # se puede obtener como mean particle separation: tracer_catalogue.mps()
    n_rec,                    # int : number of reconstruction of the displacement field (?)
    step_size,                # cell size for the estimation of the divergence field (mps units) (?)
    threshold,                # double (0,1)threshold at which to stop the reconstruction of the displacement field 
    print                     # conditions for print the displacement and the divergence field vector of bool
    );
```


Firma de la Funcion

```c
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


# Que sucede dentro del algoritmo

Ver VoidCatalogue.cpp

1. Etapa de inicializacion de algunas variables:

```c
  const double start_time = omp_get_wtime();
  shared_ptr<Catalogue> tracer_cat, random_cat;
  vector<Catalogue> displacement_catalogue(n_rec);
  vector<vector<unsigned int>> randoms;
  cosmology::Cosmology cosm;
  CatalogueChainMesh ChainMesh_tracers;
  CatalogueChainMesh ChainMesh_randoms;
```

2. Ahora tenemos un if doble:
  - uno para: ```algorithm==VoidAlgorithm::_LaZeVo_```
  - y otro para: ```algorithm==VoidAlgorithm::_Exact_```

Dentro de la ejecucion de cada uno de los elementos tenemos 4 etapas:

1. ```* * * ChainMesh setting * * *```
  - Revisar CatalogueChainMesh.cpp
2. ```* * * Searching close particles * * *```
3. ```* * * Setting starting configuration * * *```
4. ```* * * Performing the iterations * * *```
5. ``` Writing the displacement field in ../output/Displacement_output.dat ...```


# Tareas iniciales
1. Revisar Object.h que contiene la definicion de catalogue y Catalogue
2. Revisar CatalogueChainMesh.h que contiene la definicion de chainmesh