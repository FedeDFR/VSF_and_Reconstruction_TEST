# VSF_and_Reconstruction_TEST

## Pasos para compilar la libreria

1. Clonar la libreria que esta en:
https://github.com/federicomarulli/CosmoBolognaLib#

2. En el archivo Func/Func.cpp cambiar la linea 539

```cpp
delete ydata;
```

por:

```cpp
delete[] ydata;
```


3. Creo un entorno virtual:

```bash
python3 -m venv venv
```

Lo activamos

```bash
source venv/bin/activate
```

4. Instalamos las dependencias:

```bash
pip install -r requirements.txt
```

5. Despues de ver muchos problemas de Werror gracias a la herencia saque la flag -Werror del Makefile, asi compila pero con warnings. 

En la linea 79 del Makefile sacamos -Werror. Y ejecutamos el make:

```bash
make
```

## Probrar el Example BitVF_LaZeVo

1. Iniciamos el entorno virtual

```bash
source venv/bin/activate
```

2. Cambia todo el Makefile que se encuentra en Examples/cosmicVoids/codes por:

```makefile
CXX = g++

FLAGS0 = -std=c++17 -fopenmp 
FLAGS = -O3 -unroll -Wall -Wextra -pedantic -Wfatal-errors -Werror

dirLib = $(PWD)/../../../
dirH = $(dirLib)Headers/
dir_Eigen = $(dirLib)External/Eigen/eigen-3.4.0/
dir_CCfits = $(dirLib)External/CCfits/include
dirCUBA = $(dirLib)External/Cuba-4.2.1/

FLAGS_LIB = -Wl,-rpath,$(HOME)/lib/ -Wl,-rpath,$(dirLib) -L$(dirLib) -lCBL
FLAGS_INC = -I$(HOME)/include/  -I$(dirH) -I$(dirCUBA) -I$(dir_Eigen) -I$(dir_CCfits) 

OBJ1 = sizeFunction.o
OBJ2 = cleanVoidCatalogue.o
OBJ3 = modelling_VoidAbundances.o
OBJ4 = voidFinder.o
OBJ7 = createCatalogue.o
OBJ8 = BitVF_LaZeVo.o   # <-- agregado

ES = so

SYS:=$(shell uname -s)

ifeq ($(SYS),Darwin)
        ES = dylib
endif

sizeFunction: $(OBJ1) 
	$(CXX) $(OBJ1) -o sizeFunction $(FLAGS_LIB)

cleanVoidCatalogue: $(OBJ2) 
	$(CXX) $(OBJ2) -o cleanVoidCatalogue $(FLAGS_LIB)

modelling_VoidAbundances: $(OBJ3) 
	$(CXX) $(OBJ3) -o modelling_VoidAbundances $(FLAGS_LIB)

voidFinder: $(OBJ4) 
	$(CXX) $(OBJ4) -o voidFinder $(FLAGS_LIB)

createCatalogue: $(OBJ7)
	$(CXX) $(OBJ7) -o createCatalogue $(FLAGS_LIB)

BitVF_LaZeVo: $(OBJ8)   # <-- nuevo ejecutable
	$(CXX) $(OBJ8) -o BitVF_LaZeVo $(FLAGS_LIB)

clean:
	rm -f *.o sizeFunction cleanVoidCatalogue modelling_VoidAbundances voidFinder createCatalogue BitVF_LaZeVo *~ \#* temp* core*

sizeFunction.o: sizeFunction.cpp makefile $(dirLib)*.$(ES)
	$(CXX) $(FLAGS0) $(FLAGS) $(FLAGS_INC) -c sizeFunction.cpp

cleanVoidCatalogue.o: cleanVoidCatalogue.cpp makefile $(dirLib)*.$(ES)
	$(CXX) $(FLAGS0) $(FLAGS) $(FLAGS_INC) -c cleanVoidCatalogue.cpp

modelling_VoidAbundances.o: modelling_VoidAbundances.cpp makefile $(dirLib)*.$(ES)
	$(CXX) $(FLAGS0) $(FLAGS) $(FLAGS_INC) -c modelling_VoidAbundances.cpp

voidFinder.o: voidFinder.cpp makefile $(dirLib)*.$(ES)
	$(CXX) $(FLAGS0) $(FLAGS) $(FLAGS_INC) -c voidFinder.cpp

createCatalogue.o: createCatalogue.cpp makefile $(dirLib)*.$(ES)
	$(CXX) $(FLAGS0) $(FLAGS) $(FLAGS_INC) -c createCatalogue.cpp

BitVF_LaZeVo.o: BitVF_LaZeVo.cpp makefile $(dirLib)*.$(ES)
	$(CXX) $(FLAGS0) $(FLAGS) $(FLAGS_INC) -c BitVF_LaZeVo.cpp
```

3. Cambia en el archivo BitVF_LaZeVo.cpp en la linea 60:

```cpp
vector<bool> print={true,true}
```

 por:

```cpp
vector<bool> print={true,true};
```

4. Compilamos:

```bash
make
make BitVF_LaZeVo
```

Ejecutamos el archivo compilado

```bash
./BitVF_LaZeVo
```
