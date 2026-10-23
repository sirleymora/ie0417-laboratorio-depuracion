# Laboratorio: depuración con gdb, valgrind y sanitizers

## Estudiante

Nombre: Sirley Mora Chavarría

Carné: C15059

Curso: IE0417 Diseño de Software para Ingeniería, Universidad de Costa Rica

## Descripción

Este laboratorio practica la depuración de programas en C++ con distintas herramientas: el compilador `g++`, el depurador `gdb`, `valgrind` (Memcheck y Helgrind) y los sanitizers de GCC (AddressSanitizer y ThreadSanitizer). Se trabajan errores de sintaxis, de ejecución y lógicos, pérdidas de memoria, accesos inválidos, uso de memoria liberada, condiciones de carrera y un mini reto final con varios errores combinados.

Cada ejercicio documenta el código original, el error observado, la herramienta usada, la explicación del problema, la corrección y la evidencia de que el programa corregido funciona.

## Entorno

| Elemento | Versión |
|----------|---------|
| Sistema | Ubuntu 24.04.3 LTS en WSL2 (Windows) |
| Compilador | g++ 13.3.0 |
| Depurador | gdb 15.1 |
| Valgrind | 3.22.0 |
| Editor | Visual Studio Code con la extensión WSL |

## Índice

| Parte | Tema | Documento | Código |
|-------|------|-----------|--------|
| 1 | Instalación y preparación del ambiente | [parte1-instalacion.md](parte1-instalacion.md) | [prueba.cpp](codigo/prueba.cpp) |
| 2 | Tipos de errores: sintaxis, ejecución y lógicos | [parte2-tipos-errores.md](parte2-tipos-errores.md) | [01_error_sintaxis.cpp](codigo/01_error_sintaxis.cpp), [02_error_logico.cpp](codigo/02_error_logico.cpp) |
| 3 | Uso básico de gdb | [parte3-gdb-basico.md](parte3-gdb-basico.md) | [03_gdb_basico.cpp](codigo/03_gdb_basico.cpp) |
| 4 | gdb con funciones | [parte4-gdb-funciones.md](parte4-gdb-funciones.md) | [04_gdb_funciones.cpp](codigo/04_gdb_funciones.cpp) |
| 5 | Valgrind: pérdidas de memoria y accesos inválidos | [parte5-valgrind.md](parte5-valgrind.md) | [05_memory_leak.cpp](codigo/05_memory_leak.cpp), [06_invalid_access.cpp](codigo/06_invalid_access.cpp) |
| 6 | AddressSanitizer: uso de memoria liberada | [parte6-asan.md](parte6-asan.md) | [07_use_after_free.cpp](codigo/07_use_after_free.cpp) |
| 7 | Hilos y condiciones de carrera (ThreadSanitizer y Helgrind) | [parte7-hilos.md](parte7-hilos.md) | [08_race_condition.cpp](codigo/08_race_condition.cpp), [09_race_condition_corregido.cpp](codigo/09_race_condition_corregido.cpp) |
| 8 | Mini reto de depuración y reflexión final | [parte8-reto-y-reflexion.md](parte8-reto-y-reflexion.md) | [10_reto.cpp](codigo/10_reto.cpp) |

## Estructura del repositorio

```text
laboratorio-depuracion/
├── README.md
├── parte1-instalacion.md
├── parte2-tipos-errores.md
├── parte3-gdb-basico.md
├── parte4-gdb-funciones.md
├── parte5-valgrind.md
├── parte6-asan.md
├── parte7-hilos.md
├── parte8-reto-y-reflexion.md
└── codigo/
    ├── prueba.cpp
    ├── 01_error_sintaxis.cpp
    ├── 02_error_logico.cpp
    ├── 03_gdb_basico.cpp
    ├── 04_gdb_funciones.cpp
    ├── 05_memory_leak.cpp
    ├── 06_invalid_access.cpp
    ├── 07_use_after_free.cpp
    ├── 08_race_condition.cpp
    ├── 09_race_condition_corregido.cpp
    └── 10_reto.cpp
```

## Resumen de errores y herramientas

| Archivo | Error | Herramienta principal | Corrección |
|---------|-------|-----------------------|------------|
| `01_error_sintaxis.cpp` | Falta un `;` | Compilador (`g++`) | Agregar el `;` |
| `02_error_logico.cpp` | Promedio incorrecto por división entera | Ejecución normal | Convertir a `double` antes de dividir |
| `03_gdb_basico.cpp` | Sin error | `gdb` | No requiere corrección |
| `04_gdb_funciones.cpp` | División entre cero | `gdb` | Validar que el divisor no sea cero |
| `05_memory_leak.cpp` | Pérdida de memoria | `valgrind` | Agregar `delete[]` |
| `06_invalid_access.cpp` | Acceso fuera de los límites de un arreglo | `valgrind` | Corregir la condición del ciclo (`i < 5`) |
| `07_use_after_free.cpp` | Uso de memoria después de liberarla | AddressSanitizer | Usar el valor antes de hacer `delete` |
| `08_race_condition.cpp` | Condición de carrera sobre `contador` | ThreadSanitizer y Helgrind | `std::mutex` con `std::lock_guard` (`09_race_condition_corregido.cpp`) |
| `10_reto.cpp` | Mayor mal calculado, acceso fuera de límites, promedio incorrecto y pérdida de memoria | `valgrind`, AddressSanitizer y `gdb` | Inicializar con `datos[0]`, `i < tamano`, dividir la suma entre `tamano`, `delete[]` |

## Compilación y ejecución

Los comandos se ejecutan desde la carpeta `laboratorio-depuracion`.

Compilación normal, con información de depuración:

```bash
g++ -g -o programa codigo/archivo.cpp
./programa
```

Con `gdb`:

```bash
gdb -q ./programa
```

Con `valgrind`:

```bash
valgrind --leak-check=full ./programa
valgrind --tool=helgrind ./programa
```

Con AddressSanitizer:

```bash
g++ -g -fsanitize=address -o programa_asan codigo/archivo.cpp
./programa_asan
```

Con ThreadSanitizer:

```bash
g++ -g -fsanitize=thread -pthread -o programa_tsan codigo/archivo.cpp
./programa_tsan
```

Los ejecutables generados no se incluyen en el repositorio.

## Notas sobre el entorno (WSL2)

- ThreadSanitizer puede fallar en WSL2 con `FATAL: ThreadSanitizer: unexpected memory mapping`. Se resolvió con `sudo sysctl vm.mmap_rnd_bits=28`.
- Para que `gdb` no se detenga con mensajes interactivos, se creó un archivo `~/.gdbinit` con `set debuginfod enabled off` y `set pagination off`.

## Conclusiones

Este laboratorio me mostró que un programa que compila no necesariamente es correcto. Varios de los ejercicios se ejecutaban sin problemas aparentes, pero daban resultados incorrectos o tenían errores de memoria que el programa no reportaba por sí solo. Cada herramienta resultó útil para un tipo distinto de problema. `gdb` sirvió para seguir la ejecución línea por línea y ver el valor de las variables, lo que ayuda a entender en qué punto algo deja de ser lo esperado. `valgrind` permitió encontrar pérdidas de memoria y accesos inválidos sin recompilar. AddressSanitizer detuvo el programa justo en el error, con la línea exacta, y ThreadSanitizer y Helgrind detectaron condiciones de carrera aunque el resultado saliera bien en esa ejecución.
