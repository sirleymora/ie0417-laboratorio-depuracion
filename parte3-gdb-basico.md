# Parte 3: introducción a gdb

## Código

Archivo `codigo/03_gdb_basico.cpp`:

```cpp
#include <iostream>

int duplicar(int x) {
    int resultado = x * 2;
    return resultado;
}

int sumar(int a, int b) {
    int resultado = a + b;
    return resultado;
}

int main() {
    int numero = 5;
    int doble = duplicar(numero);
    int total = sumar(doble, 10);

    std::cout << "Número: " << numero << std::endl;
    std::cout << "Doble: " << doble << std::endl;
    std::cout << "Total: " << total << std::endl;

    return 0;
}
```

## Compilación y ejecución normal

```bash
g++ -g -o gdb_basico codigo/03_gdb_basico.cpp
./gdb_basico
```

```text
Número: 5
Doble: 10
Total: 20
```

## Sesión con gdb

```bash
gdb -q ./gdb_basico
```

Antes de empezar desactivé dos avisos que interrumpían la sesión: la pregunta de `debuginfod` y la paginación de la salida (`--Type <RET> for more`).

```text
(gdb) set debuginfod enabled off
(gdb) set pagination off
(gdb) break main
Breakpoint 1 at 0x11cb: file codigo/03_gdb_basico.cpp, line 14.
(gdb) run
Starting program: /home/sirley/laboratorio-depuracion/gdb_basico
[Thread debugging using libthread_db enabled]
Using host libthread_db library "/lib/x86_64-linux-gnu/libthread_db.so.1".

Breakpoint 1, main ()
    at codigo/03_gdb_basico.cpp:14
14          int numero = 5;
(gdb) next
15          int doble = duplicar(numero);
(gdb) next
16          int total = sumar(doble, 10);
(gdb) print numero
$1 = 5
(gdb) next
18          std::cout << "Número: " << numero << std::endl;
(gdb) print doble
$2 = 10
(gdb) next
Número: 5
19          std::cout << "Doble: " << doble << std::endl;
(gdb) print total
$3 = 20
(gdb) continue
Continuing.
Doble: 10
Total: 20
[Inferior 1 (process 24600) exited normally]
(gdb) quit
```

## Qué hace cada comando

- **`-g` (al compilar):** agrega información de depuración al ejecutable, como los nombres de las variables y la relación entre las instrucciones y las líneas del código. Sin ella, `gdb` no podría mostrar los nombres ni los números de línea.
- **`break main`:** crea un breakpoint al inicio de la función `main`. El programa se va a detener ahí cuando se ejecute.
- **`run`:** inicia la ejecución del programa dentro de `gdb`. Se detiene en el primer breakpoint que encuentre.
- **`next`:** ejecuta la línea actual y se detiene en la siguiente, sin entrar a las funciones que se llamen en esa línea.
- **`print`:** muestra el valor de una variable en ese momento de la ejecución.
- **`continue`:** reanuda la ejecución hasta el siguiente breakpoint o hasta que el programa termine.

## Valores observados

| Variable | Valor | Momento en que la consulté |
|----------|-------|----------------------------|
| `numero` | 5 | Con el programa detenido en la línea 16 |
| `doble` | 10 | Con el programa detenido en la línea 18 |
| `total` | 20 | Con el programa detenido en la línea 19 |

Cada variable ya tenía su valor porque la línea que la asigna se había ejecutado antes de consultarla.

## Preguntas de reflexión

1. **¿Qué es un breakpoint?** Es un punto del programa donde se le pide al depurador que detenga la ejecución. Permite revisar el estado del programa (variables, línea actual) justo en ese punto.
2. **¿Qué diferencia hay entre ejecutar el programa normalmente y ejecutarlo dentro de `gdb`?** Normalmente el programa corre de principio a fin y solo se ve su salida final. Dentro de `gdb` puedo detenerlo, avanzar línea por línea e inspeccionar variables mientras se ejecuta.
3. **¿Qué ventaja tiene inspeccionar variables mientras el programa se ejecuta?** Permite ver el valor real de cada variable en cada paso, así que se puede encontrar en qué punto un valor deja de ser el esperado, en lugar de adivinar la causa solo con el resultado final.
4. **¿Por qué `next` no entra dentro de las funciones?** Porque `next` ejecuta la línea completa como un solo paso, incluyendo las funciones que llame, sin mostrar lo que pasa dentro de ellas. En la línea `int doble = duplicar(numero);`, `next` ejecutó `duplicar` completa y pasó a la siguiente línea de `main`. Para entrar a una función existe `step`.