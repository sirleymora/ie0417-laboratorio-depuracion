# Parte 7: análisis de hilos y condiciones de carrera

## Código con condición de carrera

Archivo `codigo/08_race_condition.cpp`:

```cpp
#include <iostream>
#include <thread>

int contador = 0;

void incrementar() {
    for (int i = 0; i < 100000; i++) {
        contador++;
    }
}

int main() {
    std::thread hilo1(incrementar);
    std::thread hilo2(incrementar);

    hilo1.join();
    hilo2.join();

    std::cout << "Contador final: " << contador << std::endl;

    return 0;
}
```

Dos hilos ejecutan la misma función, que incrementa 100000 veces la variable global `contador`.

## Resultado esperado

Cada hilo suma 100000, así que el valor final esperado es 2 × 100000 = **200000**.

## Ejecución normal

```bash
g++ -g -pthread -o race_condition codigo/08_race_condition.cpp
./race_condition
```

| Ejecución | Valor esperado | Valor obtenido |
|-----------|----------------|----------------|
| 1 | 200000 | 200000 |
| 2 | 200000 | 166199 |
| 3 | 200000 | 200000 |
| 4 | 200000 | 200000 |
| 5 | 200000 | 200000 |

En 4 de las 5 ejecuciones el resultado fue el esperado, pero en la segunda dio 166199, es decir, se perdieron 33801 incrementos. El resultado no siempre es igual, y que casi siempre salga bien no significa que el programa sea correcto.

## ThreadSanitizer

```bash
g++ -g -fsanitize=thread -pthread -o race_condition_tsan codigo/08_race_condition.cpp
./race_condition_tsan
```

```text
==================
WARNING: ThreadSanitizer: data race (pid=38160)
  Read of size 4 at 0x587edc690154 by thread T2:
    #0 incrementar() codigo/08_race_condition.cpp:8
    (...)

  Previous write of size 4 at 0x587edc690154 by thread T1:
    #0 incrementar() codigo/08_race_condition.cpp:8
    (...)

  Location is global 'contador' of size 4 at 0x587edc690154

  Thread T2 created by main thread at:
    #2 main codigo/08_race_condition.cpp:14
  Thread T1 created by main thread at:
    #2 main codigo/08_race_condition.cpp:13

SUMMARY: ThreadSanitizer: data race codigo/08_race_condition.cpp:8 in incrementar()
==================
==================
WARNING: ThreadSanitizer: data race (pid=38160)
  Write of size 4 at 0x587edc690154 by thread T2:
    #0 incrementar() codigo/08_race_condition.cpp:8
    (...)

  Previous write of size 4 at 0x587edc690154 by thread T1:
    #0 incrementar() codigo/08_race_condition.cpp:8
    (...)

  Location is global 'contador' of size 4 at 0x587edc690154
  (...)

SUMMARY: ThreadSanitizer: data race codigo/08_race_condition.cpp:8 in incrementar()
==================
Contador final: 200000
ThreadSanitizer: reported 2 warnings
```

(Se recortaron con `(...)` las líneas internas de la librería de C++ y de los hilos.)

ThreadSanitizer reportó 2 condiciones de carrera (`data race`), ambas en la línea 8 (`contador++`), sobre la variable global `contador`:

- Una lectura hecha por un hilo (T2) que choca con una escritura del otro (T1).
- Una escritura de T2 que choca con una escritura de T1.

También indica en qué líneas de `main` se crearon los hilos (13 y 14). En esa ejecución el resultado fue 200000, pero la herramienta detectó el problema de todas formas, porque el acceso sin sincronización existe aunque el resultado salga bien.

**Nota sobre el entorno:** en WSL2, ThreadSanitizer puede fallar con `FATAL: ThreadSanitizer: unexpected memory mapping`. Se resolvió con:

```bash
sudo sysctl vm.mmap_rnd_bits=28
```

## Helgrind

```bash
valgrind --tool=helgrind ./race_condition
```

```text
==38269== Possible data race during read of size 4 at 0x10C154 by thread #3
==38269== Locks held: none
==38269==    at 0x10929A: incrementar() (08_race_condition.cpp:8)
==38269==    (...)
==38269==
==38269== This conflicts with a previous write of size 4 by thread #2
==38269== Locks held: none
==38269==    at 0x1092A3: incrementar() (08_race_condition.cpp:8)
==38269==    (...)
==38269==  Address 0x10c154 is 0 bytes inside data symbol "contador"
==38269==
==38269== Possible data race during write of size 4 at 0x10C154 by thread #3
==38269== Locks held: none
==38269==    at 0x1092A3: incrementar() (08_race_condition.cpp:8)
==38269==    (...)
==38269==
==38269== This conflicts with a previous write of size 4 by thread #2
==38269== Locks held: none
==38269==    at 0x1092A3: incrementar() (08_race_condition.cpp:8)
==38269==    (...)
==38269==  Address 0x10c154 is 0 bytes inside data symbol "contador"
==38269==
Contador final: 200000
==38269== ERROR SUMMARY: 2 errors from 2 contexts (suppressed: 1 from 1)
```

Helgrind reportó 2 posibles condiciones de carrera (`Possible data race`), una de lectura y otra de escritura, ambas en la línea 8 y sobre `contador` (`0 bytes inside data symbol "contador"`). Indica que cada hilo accedió a la variable sin ningún candado (`Locks held: none`) y que ese acceso choca con una escritura previa del otro hilo.

## Qué es una condición de carrera

Es un error que ocurre cuando dos o más hilos acceden a la misma variable compartida al mismo tiempo, al menos uno de ellos la modifica y no hay sincronización entre ellos. El resultado depende del orden en que el sistema intercala la ejecución de los hilos, y por eso puede cambiar de una ejecución a otra.

## Corrección con `std::mutex`

Archivo `codigo/09_race_condition_corregido.cpp`:

```cpp
#include <iostream>
#include <thread>
#include <mutex>

int contador = 0;
std::mutex mtx;

void incrementar() {
    for (int i = 0; i < 100000; i++) {
        std::lock_guard<std::mutex> lock(mtx);
        contador++;
    }
}

int main() {
    std::thread hilo1(incrementar);
    std::thread hilo2(incrementar);

    hilo1.join();
    hilo2.join();

    std::cout << "Contador final: " << contador << std::endl;

    return 0;
}
```

Agregué un `std::mutex` global y, dentro del ciclo, un `std::lock_guard` que lo bloquea antes de hacer `contador++` y lo libera automáticamente al terminar cada iteración. Así solo un hilo a la vez puede modificar `contador`.

## Resultado después de corregir

Ejecución normal:

```bash
g++ -g -pthread -o race_condition_corregido codigo/09_race_condition_corregido.cpp
./race_condition_corregido
```

```text
Contador final: 200000
```

Con Helgrind:

```bash
valgrind --tool=helgrind ./race_condition_corregido
```

```text
==39534== Helgrind, a thread error detector
==39534== Command: ./race_condition_corregido
==39534== 
Contador final: 200000
==39534== 
==39534== ERROR SUMMARY: 0 errors from 0 contexts (suppressed: 300005 from 8)
```

Con ThreadSanitizer:

```bash
g++ -g -fsanitize=thread -pthread -o race_condition_corregido_tsan codigo/09_race_condition_corregido.cpp
./race_condition_corregido_tsan
```

```text
Contador final: 200000
```

El resultado es 200000, Helgrind reporta 0 errores y ThreadSanitizer no emite ningún `WARNING`. Los avisos suprimidos de Helgrind (`suppressed`) son falsos positivos conocidos de las librerías internas y no corresponden al código del laboratorio.

## Preguntas de reflexión

1. **¿Por qué `contador++` no es una operación segura entre varios hilos?** Porque no es una sola operación: se compone de leer el valor, sumarle 1 y escribirlo de nuevo. Si dos hilos leen el mismo valor antes de que alguno escriba, ambos escriben el mismo resultado y se pierde un incremento, como pasó con el valor 166199.
2. **¿Qué significa que dos hilos accedan a la misma variable compartida?** Que ambos leen o modifican la misma posición de memoria (aquí `contador`, una variable global), por lo que lo que haga uno afecta lo que ve el otro.
3. **¿Qué problema resuelve `std::mutex`?** Garantiza exclusión mutua: solo un hilo a la vez puede ejecutar la parte protegida del código, así que no se mezclan los accesos a la variable compartida.
4. **¿Qué hace `std::lock_guard`?** Bloquea el mutex al crearse y lo desbloquea automáticamente al salir del bloque donde se declaró, incluso si hay una excepción, para que no se olvide liberarlo.
5. **¿Por qué los errores de concurrencia pueden ser difíciles de reproducir?** Porque dependen de cómo el sistema operativo intercala los hilos, que cambia en cada ejecución. En mis pruebas, 4 de 5 ejecuciones dieron el resultado correcto aunque el error existía, así que el problema puede aparecer solo a veces.
6. **¿Cuál herramienta le pareció más clara para analizar este caso: ThreadSanitizer o Helgrind?**
ThreadSanitizer lo declara como `data race` e indica la variable global y las líneas donde se crearon los hilos, mientras Helgrind dice "posible" y destaca que no había candados (`Locks held: none`). 