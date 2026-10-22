# Parte 6: AddressSanitizer

En esta parte usé AddressSanitizer (ASan), una herramienta que se activa al compilar con `-fsanitize=address` y detecta errores de memoria mientras el programa se ejecuta. Además, comparé su reporte con el de `valgrind`.

## Código original

Archivo `codigo/07_use_after_free.cpp`:

```cpp
#include <iostream>

int main() {
    int* numero = new int;
    *numero = 42;

    delete numero;

    std::cout << "Número: " << *numero << std::endl;

    return 0;
}
```

El programa reserva un entero, le asigna 42, libera la memoria con `delete` y después intenta leer el valor a través del mismo puntero. Ese puntero ya apunta a memoria liberada.

## Ejecución sin sanitizer

```bash
g++ -g -o use_after_free codigo/07_use_after_free.cpp
./use_after_free
```

```text
Número: -2140179392
```

El programa compiló, se ejecutó y terminó sin ningún aviso, pero imprimió un valor basura (`-2140179392`) en vez de 42. No hubo ningún mensaje de error: el programa pareció funcionar, pero el dato era incorrecto.

## Ejecución con AddressSanitizer

```bash
g++ -g -fsanitize=address -o use_after_free_asan codigo/07_use_after_free.cpp
./use_after_free_asan
```

```text
=================================================================
==31529==ERROR: AddressSanitizer: heap-use-after-free on address 0x502000000010 at pc 0x5c9d9a32734e bp 0x7fff4cce52b0 sp 0x7fff4cce52a0
READ of size 4 at 0x502000000010 thread T0
    #0 0x5c9d9a32734d in main codigo/07_use_after_free.cpp:9
    (...)

0x502000000010 is located 0 bytes inside of 4-byte region [0x502000000010,0x502000000014)
freed by thread T0 here:
    #0 0x744a574ff5e8 in operator delete(void*, unsigned long) ../../../../src/libsanitizer/asan/asan_new_delete.cpp:164
    #1 0x5c9d9a3272f9 in main codigo/07_use_after_free.cpp:7
    (...)

previously allocated by thread T0 here:
    #0 0x744a574fe548 in operator new(unsigned long) ../../../../src/libsanitizer/asan/asan_new_delete.cpp:95
    #1 0x5c9d9a32729e in main codigo/07_use_after_free.cpp:4
    (...)

SUMMARY: AddressSanitizer: heap-use-after-free codigo/07_use_after_free.cpp:9 in main
==31529==ABORTING
```

ASan detuvo el programa en el momento del error y mostró tres cosas:

- **Dónde se usó la memoria ya liberada:** línea 9, una lectura (`READ of size 4`) en `main`.
- **Dónde se liberó:** línea 7, en el `delete numero;`.
- **Dónde se reservó:** línea 4, en `new int`.

## Qué significa `heap-use-after-free`

Significa que el programa usó (leyó o escribió) un bloque de memoria del heap después de haberlo liberado. El puntero sigue guardando la dirección vieja, pero esa memoria ya no pertenece al programa y puede estar vacía o reutilizada por otra cosa.

## Comparación con valgrind

También ejecuté el programa original con `valgrind`:

```bash
valgrind ./use_after_free
```

```text
==31594== Invalid read of size 4
==31594==    at 0x109223: main (07_use_after_free.cpp:9)
==31594==  Address 0x4e1e080 is 0 bytes inside a block of size 4 free'd
==31594==    at 0x484A61D: operator delete(void*, unsigned long) (in /usr/libexec/valgrind/vgpreload_memcheck-amd64-linux.so)
==31594==    by 0x109202: main (07_use_after_free.cpp:7)
==31594==  Block was alloc'd at
==31594==    at 0x4846FA3: operator new(unsigned long) (in /usr/libexec/valgrind/vgpreload_memcheck-amd64-linux.so)
==31594==    by 0x1091DE: main (07_use_after_free.cpp:4)
==31594==
Número: 42
==31594== ERROR SUMMARY: 1 errors from 1 contexts (suppressed: 0 from 0)
```

`valgrind` reportó lo mismo con otras palabras: una lectura inválida (`Invalid read of size 4`) en la línea 9, sobre un bloque liberado en la línea 7 y reservado en la línea 4. La diferencia es que `valgrind` dejó que el programa terminara (imprimió `Número: 42`) y ASan lo detuvo en el primer error.

## Corrección

Usé el valor antes de liberar la memoria y después asigné `nullptr` al puntero:

```cpp
#include <iostream>

int main() {
    int* numero = new int;
    *numero = 42;

    std::cout << "Número: " << *numero << std::endl;

    delete numero;
    numero = nullptr;

    return 0;
}
```

## Resultado final

```bash
g++ -g -fsanitize=address -o use_after_free_asan codigo/07_use_after_free.cpp
./use_after_free_asan
```

```text
Número: 42
```

El programa imprime el valor correcto y ASan ya no reporta ningún error.

## Preguntas de reflexión

1. **¿Qué significa usar memoria después de liberarla?** Que el programa accede, a través de un puntero, a memoria que ya devolvió al sistema con `delete`. Ese puntero queda "colgante": todavía guarda la dirección, pero la memoria ya no es válida para el programa.
2. **¿Por qué este error puede ser difícil de detectar sin herramientas?** Porque el programa no falla de forma evidente. Aquí compiló y terminó normalmente, pero el resultado fue basura (`-2140179392`) en una ejecución y `42` bajo `valgrind`. Como el resultado depende de qué haya pasado con esa memoria, el error puede pasar desapercibido o aparecer solo a veces.
3. **¿Qué ventaja tiene AddressSanitizer sobre ejecutar el programa normalmente?** Detecta el error en el momento en que ocurre, detiene el programa e indica la línea exacta del uso inválido, de la liberación y de la reserva. Ejecutando normalmente no se ve ningún aviso, solo un resultado incorrecto que hay que sospechar.
4. **¿Qué diferencia observó entre el reporte de `valgrind` y el de AddressSanitizer?** Ambos señalan las mismas tres líneas (uso, liberación y reserva). ASan se activa al compilar con `-fsanitize=address`, detiene el programa en el primer error y muestra además un volcado del estado de la memoria. `valgrind` analiza el ejecutable sin cambiar la compilación, deja que el programa termine y reporta el error con su propio formato (`Invalid read`, `free'd`, `alloc'd`).
5. **¿Por qué es buena práctica asignar `nullptr` después de liberar un puntero?** Porque después de `delete` el puntero sigue apuntando a memoria inválida. Con `nullptr`, un uso accidental provoca un fallo inmediato y fácil de detectar (en vez de leer datos basura sin avisar) y además permite comprobar si el puntero es válido (`if (numero != nullptr)`).