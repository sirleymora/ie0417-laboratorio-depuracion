# Parte 4: step, next y backtrace

## Código original

Archivo `codigo/04_gdb_funciones.cpp`:

```cpp
#include <iostream>

int dividir(int a, int b) {
    int resultado = a / b;
    return resultado;
}

int calcular(int x) {
    int divisor = x - 5;
    int resultado = dividir(100, divisor);
    return resultado;
}

int main() {
    int valor = 5;
    int respuesta = calcular(valor);

    std::cout << "Respuesta: " << respuesta << std::endl;

    return 0;
}
```

## Compilación y ejecución normal

```bash
g++ -g -o gdb_funciones codigo/04_gdb_funciones.cpp
./gdb_funciones
```

## Error observado

```text
Floating point exception (core dumped)
```

El programa compila sin problemas, pero se detiene durante la ejecución con este mensaje. No dice en qué línea ni por qué ocurrió, así que usé `gdb` para investigarlo.

## Depuración con gdb

```bash
gdb -q ./gdb_funciones
```

```text
(gdb) break main
Breakpoint 1 at 0x11e1: file codigo/04_gdb_funciones.cpp, line 15.
(gdb) run
Starting program: /home/sirley/laboratorio-depuracion/gdb_funciones
[Thread debugging using libthread_db enabled]
Using host libthread_db library "/lib/x86_64-linux-gnu/libthread_db.so.1".

Breakpoint 1, main ()
    at codigo/04_gdb_funciones.cpp:15
15          int valor = 5;
(gdb) next
16          int respuesta = calcular(valor);
(gdb) step
calcular (x=5) at codigo/04_gdb_funciones.cpp:9
9           int divisor = x - 5;
(gdb) step
10          int resultado = dividir(100, divisor);
(gdb) print x
$1 = 5
(gdb) print divisor
$2 = 0
(gdb) step
dividir (a=100, b=0)
    at codigo/04_gdb_funciones.cpp:4
4           int resultado = a / b;
(gdb) print a
$3 = 100
(gdb) print b
$4 = 0
(gdb) backtrace
#0  dividir (a=100, b=0)
    at codigo/04_gdb_funciones.cpp:4
#1  0x00005555555551cd in calcular (x=5)
    at codigo/04_gdb_funciones.cpp:10
#2  0x00005555555551f2 in main ()
    at codigo/04_gdb_funciones.cpp:16
(gdb) quit
```

## Valores observados

| Variable | Valor | Dónde |
|----------|-------|-------|
| `valor` | 5 | en `main` |
| `x` | 5 | en `calcular` |
| `divisor` | 0 | en `calcular`, resultado de `x - 5` |
| `a` | 100 | en `dividir` |
| `b` | 0 | en `dividir` |

## Qué encontré con gdb

El valor 5 de `valor` se pasa a `calcular` como `x`. Ahí, `divisor` se calcula como `x - 5`, que es 0. Ese 0 se le pasa a `dividir` como `b`, y la línea `int resultado = a / b;` intenta dividir 100 entre 0. Esa división entera entre cero es lo que detiene el programa con el mensaje que vi.

## Diferencia entre `next` y `step`

- `next` ejecuta la línea actual completa y pasa a la siguiente, sin entrar a las funciones que se llamen en ella.
- `step` entra dentro de la función llamada en esa línea y se detiene en su primera línea.

En la sesión, el primer `next` avanzó de la línea 15 a la 16 dentro de `main`. Después, un `step` en la línea 16 (`calcular(valor)`) me llevó a la primera línea dentro de `calcular`, y otro `step` en la línea 10 (`dividir(100, divisor)`) me llevó dentro de `dividir`. Con `next` habría ejecutado esas funciones sin poder ver sus valores internos.

## Resultado de `backtrace`

```text
#0  dividir (a=100, b=0)           (codigo/04_gdb_funciones.cpp:4)
#1  calcular (x=5)                 (codigo/04_gdb_funciones.cpp:10)
#2  main ()                        (codigo/04_gdb_funciones.cpp:16)
```

Muestra la cadena de llamadas hasta el punto donde estaba detenido: `main` llamó a `calcular`, y `calcular` llamó a `dividir`, donde está el problema.

## Corrección

Agregué una validación en `dividir` para no dividir cuando el divisor es cero:

```cpp
#include <iostream>

int dividir(int a, int b) {
    if (b == 0) {
        std::cout << "Error: no se puede dividir entre cero." << std::endl;
        return 0;
    }
    int resultado = a / b;
    return resultado;
}

int calcular(int x) {
    int divisor = x - 5;
    int resultado = dividir(100, divisor);
    return resultado;
}

int main() {
    int valor = 5;
    int respuesta = calcular(valor);

    std::cout << "Respuesta: " << respuesta << std::endl;

    return 0;
}
```

## Resultado final

```bash
g++ -g -o gdb_funciones codigo/04_gdb_funciones.cpp
./gdb_funciones
```

```text
Error: no se puede dividir entre cero.
Respuesta: 0
```

El programa ya no falla: detecta el divisor cero, avisa con un mensaje y continúa.

## Preguntas de reflexión

1. **¿Qué diferencia observó entre `next` y `step`?** `next` ejecuta la línea sin entrar a las funciones, y `step` sí entra. Para ver qué pasa dentro de `calcular` y `dividir` tuve que usar `step`.
2. **¿Para qué sirve `backtrace`?** Muestra la pila de llamadas, es decir, qué funciones se fueron llamando una a otra hasta llegar al punto donde está detenido el programa. Sirve para saber quién llamó a la función donde ocurre el problema y con qué valores.
3. **¿Cuál fue la causa del error?** `divisor` valía 0 (porque `x - 5` con `x = 5` da 0) y ese valor llegó a `dividir`, que dividió 100 entre 0.
4. **¿Por qué este error es de tiempo de ejecución?** Porque el código es válido y compila bien. El problema solo aparece al ejecutarlo, cuando el valor de `divisor` resulta ser 0, y el compilador no puede saberlo de antemano.
5. **¿Cómo podría prevenirse este tipo de error desde el diseño del programa?** Validando los datos antes de usarlos (en este caso, comprobar que el divisor no sea cero antes de dividir), definiendo claramente qué valores acepta cada función y decidiendo cómo se reportan los errores, por ejemplo con mensajes, códigos de error o excepciones.