# Parte 5: análisis de memoria con valgrind

## Ejercicio 1: pérdida de memoria

### Código original

Archivo `codigo/05_memory_leak.cpp`:

```cpp
#include <iostream>

void crear_arreglo() {
    int* datos = new int[10];

    for (int i = 0; i < 10; i++) {
        datos[i] = i * 2;
    }

    std::cout << "Primer elemento: " << datos[0] << std::endl;
}

int main() {
    crear_arreglo();
    return 0;
}
```

### Qué hace el programa

La función `crear_arreglo` reserva memoria dinámica para 10 enteros con `new int[10]`, los llena con los valores 0, 2, 4, ..., 18 e imprime el primero. `main` solo llama a esa función.

### Error de memoria

La memoria reservada con `new[]` nunca se libera con `delete[]`. Cuando la función termina, el puntero `datos` desaparece, pero la memoria sigue reservada y ya no hay forma de acceder a ella ni de liberarla. Eso es una pérdida de memoria (memory leak).

### Compilación y ejecución normal

```bash
g++ -g -o memory_leak codigo/05_memory_leak.cpp
./memory_leak
```

```text
Primer elemento: 0
```

El programa parece funcionar bien: no falla ni muestra ningún mensaje de error.

### Análisis con valgrind

```bash
valgrind --leak-check=full --show-leak-kinds=all ./memory_leak
```

```text
==28268== HEAP SUMMARY:
==28268==     in use at exit: 40 bytes in 1 blocks
==28268==   total heap usage: 3 allocs, 2 frees, 74,792 bytes allocated
==28268==
==28268== 40 bytes in 1 blocks are definitely lost in loss record 1 of 1
==28268==    at 0x48485C3: operator new[](unsigned long) (in /usr/libexec/valgrind/vgpreload_memcheck-amd64-linux.so)
==28268==    by 0x1091BE: crear_arreglo() (05_memory_leak.cpp:4)
==28268==    by 0x10923E: main (05_memory_leak.cpp:14)
==28268==
==28268== LEAK SUMMARY:
==28268==    definitely lost: 40 bytes in 1 blocks
==28268==    indirectly lost: 0 bytes in 0 blocks
==28268==      possibly lost: 0 bytes in 0 blocks
==28268==    still reachable: 0 bytes in 0 blocks
==28268==         suppressed: 0 bytes in 0 blocks
==28268==
==28268== ERROR SUMMARY: 1 errors from 1 contexts (suppressed: 0 from 0)
```

`valgrind` reportó que al terminar el programa quedaban 40 bytes en uso (10 enteros de 4 bytes cada uno) y que fueron reservados en `crear_arreglo()`, línea 4, llamada desde `main`. También muestra `3 allocs, 2 frees`: hubo una reserva más que liberaciones.

### Qué significa `definitely lost`

Significa que la memoria reservada ya no se puede alcanzar: al terminar el programa no queda ningún puntero que apunte a ese bloque, así que es una fuga segura y no hay forma de que el programa la libere.

### Corrección

Agregué `delete[] datos;` al final de la función, después de usar el arreglo:

```cpp
void crear_arreglo() {
    int* datos = new int[10];

    for (int i = 0; i < 10; i++) {
        datos[i] = i * 2;
    }

    std::cout << "Primer elemento: " << datos[0] << std::endl;

    delete[] datos;
}
```

### Evidencia después de corregir

```bash
g++ -g -o memory_leak codigo/05_memory_leak.cpp
valgrind --leak-check=full ./memory_leak
```

```text
Primer elemento: 0
==28788== HEAP SUMMARY:
==28788==     in use at exit: 0 bytes in 0 blocks
==28788==   total heap usage: 3 allocs, 3 frees, 74,792 bytes allocated
==28788==
==28788== All heap blocks were freed -- no leaks are possible
==28788==
==28788== ERROR SUMMARY: 0 errors from 0 contexts (suppressed: 0 from 0)
```

Ahora hay 3 reservas y 3 liberaciones, no queda memoria en uso y no hay errores.

### Preguntas de reflexión

1. **¿Qué es una pérdida de memoria?** Es cuando un programa reserva memoria dinámica y nunca la libera, y además pierde la referencia a ella, de modo que esa memoria queda ocupada sin poder usarse ni liberarse.
2. **¿Por qué el programa puede terminar aparentemente bien aunque tenga una pérdida de memoria?** Porque la fuga no causa un error inmediato: el programa sigue ejecutándose normalmente y, al terminar, el sistema operativo recupera toda la memoria del proceso. El problema no se ve en la salida, solo se nota con herramientas como `valgrind` o cuando el programa corre mucho tiempo.
3. **¿Qué significa liberar memoria dinámica?** Devolverle al sistema la memoria que se reservó con `new`, usando `delete` (o `delete[]`), para que pueda reutilizarse.
4. **¿Por qué se usa `delete[]` y no solo `delete`?** Porque la memoria se reservó como arreglo con `new[]`, y debe liberarse con `delete[]`. Usar solo `delete` en un arreglo es comportamiento indefinido, ya que no se liberan ni se destruyen correctamente todos los elementos.
5. **¿Qué tipo de problemas podrían aparecer en programas grandes si no se libera memoria?** El consumo de memoria crece con el tiempo, el programa se vuelve más lento, puede afectar a otros programas y, si corre mucho tiempo (como un servidor), puede quedarse sin memoria y fallar.