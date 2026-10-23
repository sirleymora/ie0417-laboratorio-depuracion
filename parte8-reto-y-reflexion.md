# Parte 8

## Mini reto de depuración

### Código inicial

Archivo `codigo/10_reto.cpp` (versión original, antes de corregir):

```cpp
#include <iostream>

int obtener_mayor(int* datos, int tamano) {
    int mayor = 0;

    for (int i = 0; i <= tamano; i++) {
        if (datos[i] > mayor) {
            mayor = datos[i];
        }
    }

    return mayor;
}

double calcular_promedio(int* datos, int tamano) {
    int suma = 0;

    for (int i = 0; i < tamano; i++) {
        suma += datos[i];
    }

    return suma;
}

int main() {
    int tamano = 5;
    int* datos = new int[tamano];

    datos[0] = -10;
    datos[1] = -20;
    datos[2] = -5;
    datos[3] = -30;
    datos[4] = -15;

    int mayor = obtener_mayor(datos, tamano);
    double promedio = calcular_promedio(datos, tamano);

    std::cout << "Mayor: " << mayor << std::endl;
    std::cout << "Promedio: " << promedio << std::endl;

    return 0;
}
```

Los datos son -10, -20, -5, -30 y -15. El resultado esperado es **Mayor: -5** y **Promedio: -16** (la suma es -80 y -80 / 5 = -16).

### Resultado inicial

```bash
g++ -g -o reto codigo/10_reto.cpp
./reto
```

```text
Mayor: 0
Promedio: -80
```

Los dos resultados son incorrectos y el programa no muestra ningún error ni se detiene. Este es el problema de los errores lógicos y de memoria: el programa compila y se ejecuta, pero produce resultados malos.

### Herramientas usadas

Ejecución normal, `valgrind`, AddressSanitizer y `gdb`.

### Análisis con valgrind

```bash
valgrind --leak-check=full ./reto
```

```text
==44546== Invalid read of size 4
==44546==    at 0x10921C: obtener_mayor(int*, int) (10_reto.cpp:7)
==44546==    by 0x10932C: main (10_reto.cpp:35)
==44546==  Address 0x4e1e094 is 0 bytes after a block of size 20 alloc'd
==44546==    at 0x48485C3: operator new[](unsigned long) (...)
==44546==    by 0x1092D5: main (10_reto.cpp:27)
==44546==
Mayor: 0
Promedio: -80
==44546==
==44546== HEAP SUMMARY:
==44546==     in use at exit: 20 bytes in 1 blocks
==44546==   total heap usage: 3 allocs, 2 frees, 74,772 bytes allocated
==44546==
==44546== 20 bytes in 1 blocks are definitely lost in loss record 1 of 1
==44546==    at 0x48485C3: operator new[](unsigned long) (...)
==44546==    by 0x1092D5: main (10_reto.cpp:27)
==44546==
==44546== LEAK SUMMARY:
==44546==    definitely lost: 20 bytes in 1 blocks
==44546== ERROR SUMMARY: 2 errors from 2 contexts (suppressed: 0 from 0)
```

Valgrind encontró dos problemas:

- Una lectura inválida (`Invalid read of size 4`) en la línea 7, dentro de `obtener_mayor`. Indica que la dirección está "0 bytes after a block of size 20", es decir, justo después del final del arreglo de 5 enteros (5 × 4 = 20 bytes).
- Una pérdida de memoria (`definitely lost: 20 bytes in 1 blocks`) del bloque reservado en la línea 27 de `main`.

### Análisis con AddressSanitizer

```bash
g++ -g -fsanitize=address -o reto_asan codigo/10_reto.cpp
./reto_asan
```

```text
=================================================================
==44690==ERROR: AddressSanitizer: heap-buffer-overflow on address 0x503000000054 ...
READ of size 4 at 0x503000000054 thread T0
    #0 0x55656471733a in obtener_mayor(int*, int) codigo/10_reto.cpp:7
    #1 0x5565647175db in main codigo/10_reto.cpp:35
    (...)

0x503000000054 is located 0 bytes after 20-byte region [0x503000000040,0x503000000054)
allocated by thread T0 here:
    #0 0x7f57fd6fe6c8 in operator new[](unsigned long) (...)
    #1 0x556564717465 in main codigo/10_reto.cpp:27
    (...)

SUMMARY: AddressSanitizer: heap-buffer-overflow codigo/10_reto.cpp:7 in obtener_mayor(int*, int)
(...)
==44690==ABORTING
```

AddressSanitizer detuvo el programa con un `heap-buffer-overflow`: una lectura de 4 bytes en la línea 7 de `obtener_mayor`, justo después de una región de 20 bytes reservada en la línea 27 de `main`. Coincide con lo que mostró valgrind.

### Análisis con gdb

```bash
gdb -q ./reto
```

```gdb
break 7 if i == 5
run
print i
print tamano
print datos[i]
backtrace
quit
```

```text
Breakpoint 1, obtener_mayor (datos=0x55555556b2b0, tamano=5) at codigo/10_reto.cpp:7
7               if (datos[i] > mayor) {
(gdb) print i
$1 = 5
(gdb) print tamano
$2 = 5
(gdb) print datos[i]
$3 = 0
(gdb) backtrace
#0  obtener_mayor (datos=0x55555556b2b0, tamano=5) at codigo/10_reto.cpp:7
#1  0x000055555555532d in main () at codigo/10_reto.cpp:35
```

Con un breakpoint condicional (`break 7 if i == 5`) el programa se detuvo solo cuando `i` valía 5. Ahí se ve que `i` es igual a `tamano`, y que `datos[5]` devolvió `0`, un valor que no pertenece al arreglo. El programa no falló, pero leyó memoria que no le pertenece. El `backtrace` confirma que la llamada viene de `main`, línea 35.

### Errores encontrados

| # | Error | Dónde | Cómo se encontró |
|---|-------|-------|------------------|
| 1 | Error lógico en el mayor: `mayor` empieza en 0 | `obtener_mayor`, línea 4 | Ejecución normal: imprimió `Mayor: 0` y el mayor real es -5. Como todos los datos son negativos, ninguno supera a 0 y `mayor` nunca cambia. |
| 2 | Acceso fuera de límites: `i <= tamano` | `obtener_mayor`, línea 6 | valgrind (`Invalid read of size 4`), AddressSanitizer (`heap-buffer-overflow`) y gdb (`i = 5` con `tamano = 5`). |
| 3 | Promedio incorrecto: se devuelve la suma sin dividir | `calcular_promedio`, línea 17 | Ejecución normal: imprimió `Promedio: -80` cuando debía ser -16. Se devuelve `suma` en lugar de `suma / tamano`. |
| 4 | Pérdida de memoria: falta `delete[]` | `main`, línea 27 | valgrind (`definitely lost: 20 bytes in 1 blocks`). |

### Correcciones realizadas

Código corregido en `codigo/10_reto.cpp`:

```cpp
#include <iostream>

int obtener_mayor(int* datos, int tamano) {
    int mayor = datos[0];

    for (int i = 1; i < tamano; i++) {
        if (datos[i] > mayor) {
            mayor = datos[i];
        }
    }

    return mayor;
}

double calcular_promedio(int* datos, int tamano) {
    int suma = 0;

    for (int i = 0; i < tamano; i++) {
        suma += datos[i];
    }

    return static_cast<double>(suma) / tamano;
}

int main() {
    int tamano = 5;
    int* datos = new int[tamano];

    datos[0] = -10;
    datos[1] = -20;
    datos[2] = -5;
    datos[3] = -30;
    datos[4] = -15;

    int mayor = obtener_mayor(datos, tamano);
    double promedio = calcular_promedio(datos, tamano);

    std::cout << "Mayor: " << mayor << std::endl;
    std::cout << "Promedio: " << promedio << std::endl;

    delete[] datos;
    datos = nullptr;

    return 0;
}
```

1. **Mayor:** `mayor` ahora empieza con `datos[0]` y el ciclo arranca en `i = 1`. Así el máximo se compara siempre con un valor real del arreglo y funciona con números negativos.
2. **Fuera de límites:** la condición pasó de `i <= tamano` a `i < tamano`, para recorrer solo las posiciones 0 a 4.
3. **Promedio:** se divide la suma entre `tamano`, convirtiendo antes `suma` a `double` con `static_cast<double>` para que no se pierdan decimales ni se haga una división entera.
4. **Fuga:** se agregó `delete[] datos;` (con `[]`, porque se reservó con `new[]`) antes de terminar `main`, y `datos = nullptr;` para no dejar un puntero colgante.

### Resultado final

```bash
g++ -g -o reto codigo/10_reto.cpp
./reto
```

```text
Mayor: -5
Promedio: -16
```

Con valgrind:

```bash
valgrind --leak-check=full ./reto
```

```text
Mayor: -5
Promedio: -16
==48214== HEAP SUMMARY:
==48214==     in use at exit: 0 bytes in 0 blocks
==48214==   total heap usage: 3 allocs, 3 frees, 74,772 bytes allocated
==48214==
==48214== All heap blocks were freed -- no leaks are possible
==48214== ERROR SUMMARY: 0 errors from 0 contexts (suppressed: 0 from 0)
```

Con AddressSanitizer:

```bash
g++ -g -fsanitize=address -o reto_asan codigo/10_reto.cpp
./reto_asan
```

```text
Mayor: -5
Promedio: -16
```

Los resultados son los esperados, valgrind no reporta errores ni fugas, y AddressSanitizer no detecta ningún problema.

### Qué herramienta fue más útil y por qué

La más útil fue valgrind, porque con una sola ejecución (`valgrind --leak-check=full ./reto`) encontró dos de los cuatro errores: el acceso fuera de límites (`Invalid read of size 4`, justo después del bloque de 20 bytes) y la pérdida de memoria (`definitely lost: 20 bytes`). Además indicó en qué línea se reservó la memoria, y eso no lo muestra ninguna otra herramienta que usé.

## Reflexión final

1. **¿Qué entiende ahora por depuración?** Es el proceso de encontrar y entender la causa de un error en un programa y corregirla, siguiendo un orden: ejecutar, observar el síntoma, reproducir el error, usar una herramienta adecuada, identificar la causa y corregir. No consiste en cambiar el código al azar hasta que funcione.
2. **¿Por qué no basta con que un programa compile?** Porque el compilador solo revisa que el código sea válido, no que haga lo correcto. Los programas del laboratorio compilaban y aun así tenían una división por cero, un promedio mal calculado, fugas de memoria, accesos fuera de límites y condiciones de carrera.
3. **¿Cuál fue la diferencia entre un error de sintaxis y un error lógico?** El error de sintaxis impide compilar y el compilador indica el archivo y la línea. El error lógico compila y se ejecuta sin quejas, pero da un resultado equivocado, y hay que descubrirlo comparando contra lo que se esperaba.
4. **¿Qué ventaja tiene usar `gdb`?** Permite detener el programa en un punto, avanzar línea por línea y ver el valor de las variables mientras corre (`print`, `next`, `step`, `backtrace`), lo que ayuda a entender por qué ocurre un error y no solo dónde.
5. **¿Qué ventaja tiene usar `valgrind`?** Detecta errores de memoria sin recompilar el programa: fugas (`definitely lost`), lecturas y escrituras inválidas, y usa memoria ya liberada. También indica dónde se reservó la memoria. Además, con Helgrind permite analizar carreras de datos.
6. **¿Qué ventaja tiene usar AddressSanitizer?** Se activa al compilar con `-fsanitize=address` y detiene el programa en cuanto ocurre el error, con la línea exacta (por ejemplo `heap-buffer-overflow` o `heap-use-after-free`). Es más rápido que valgrind, aunque requiere recompilar.
7. **¿Qué aprendió sobre errores de memoria?** Que pueden existir sin que el programa falle: una fuga o una lectura fuera de límites pueden pasar desapercibidas y dar resultados raros solo a veces. Cada `new` necesita su `delete` (`new[]` con `delete[]`), no se debe usar memoria después de liberarla y hay que respetar los límites del arreglo.
8. **¿Qué aprendió sobre errores de concurrencia?** Que dos hilos que modifican la misma variable sin sincronización producen resultados impredecibles, que pueden aparecer solo a veces, y que `std::mutex` con `std::lock_guard` lo resuelve. ThreadSanitizer y Helgrind detectan el problema aunque el resultado salga bien.
9. **¿Cuál fue el error más fácil de encontrar?** El error de sintaxis, porque el compilador no dejó compilar el programa y señaló el archivo, la línea y la columna del problema. No hizo falta ninguna herramienta adicional.
10. **¿Cuál fue el error más difícil de entender?** La condición de carrera. El programa casi siempre daba el resultado correcto (4 de 5 ejecuciones dieron 200000) y solo una dio 166199, así que sin ThreadSanitizer o Helgrind habría sido fácil pensar que el programa estaba bien. Además, el resultado cambia en cada ejecución y no se puede reproducir a voluntad.
11. **¿Qué herramienta usaría primero si un programa produce un resultado incorrecto?** Primero ejecutaría el programa con datos conocidos para reproducir el error y luego usaría `gdb` para seguir el valor de las variables paso a paso y encontrar en qué punto el cálculo deja de ser el esperado.
12. **¿Qué herramienta usaría primero si un programa falla con segmentation fault?** AddressSanitizer, porque indica la línea exacta y el tipo de acceso inválido. Si quiero ver el estado del programa en el momento de la falla, usaría `gdb` con `backtrace`.
13. **¿Qué herramienta usaría primero si sospecha una pérdida de memoria?** `valgrind --leak-check=full`, porque muestra cuántos bytes se perdieron y en qué línea se reservaron.
14. **¿Qué herramienta usaría primero si sospecha un problema con hilos?** ThreadSanitizer (`-fsanitize=thread`), porque reporta la variable, la línea y los hilos involucrados. Helgrind es una alternativa que no requiere recompilar.
15. **¿Cómo aplicaría estas herramientas en proyectos futuros?** Compilaría siempre con `-g` y probaría con AddressSanitizer durante el desarrollo para detectar errores de memoria apenas aparezcan. Antes de entregar un programa pasaría valgrind para revisar fugas, y usaría ThreadSanitizer en todo programa con hilos. Cuando el resultado no coincida con lo esperado usaría gdb para seguir las variables paso a paso. 
