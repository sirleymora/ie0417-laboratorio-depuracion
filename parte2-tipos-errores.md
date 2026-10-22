# Parte 2: tipos de errores

En esta parte trabajé con un error de sintaxis y un error lógico. El primero lo detecta el compilador antes de ejecutar el programa. El segundo no lo detecta nadie automáticamente: el programa compila y corre, pero da un resultado incorrecto.

---

## Ejercicio 1: error de sintaxis

### Código original

Archivo `codigo/01_error_sintaxis.cpp`:

```cpp
#include <iostream>

int main() {
    std::cout << "Hola mundo" << std::endl
    return 0;
}
```

### Comando de compilación

```bash
g++ -g -o error_sintaxis codigo/01_error_sintaxis.cpp
```

### Resultado observado

El programa no compiló y `g++` mostró este mensaje:

```text
codigo/01_error_sintaxis.cpp: In function ‘int main()’:
codigo/01_error_sintaxis.cpp:4:43: error: expected ‘;’ before ‘return’
    4 |     std::cout << "Hola mundo" << std::endl
      |                                           ^
      |                                           ;
    5 |     return 0;
      |     ~~~~~~
```

Al intentar ejecutar `./error_sintaxis` apareció `No such file or directory`, porque como la compilación falló nunca se generó el ejecutable.

### Herramienta usada

El compilador `g++`.

### Análisis

El error está en la línea 4, columna 43: falta el punto y coma (`;`) al final de la instrucción `std::cout << "Hola mundo" << std::endl`. En C++ cada instrucción debe terminar en `;`. El compilador lo reportó "antes de `return`" porque se da cuenta de que falta cuando llega a la siguiente instrucción, y además indicó con un `;` dónde debía ir.

### Corrección

Agregué el `;` al final de la línea 4:

```cpp
#include <iostream>

int main() {
    std::cout << "Hola mundo" << std::endl;
    return 0;
}
```

### Evidencia

```bash
g++ -g -o error_sintaxis codigo/01_error_sintaxis.cpp
./error_sintaxis
```

```text
Hola mundo
```

### Reflexión

1. **¿Este error fue detectado antes o durante la ejecución?** Antes de la ejecución, durante la compilación. El programa ni siquiera llegó a ejecutarse.
2. **¿Qué herramienta detectó el error?** El compilador `g++`.
3. **¿Por qué este tipo de error suele ser más fácil de corregir que un error lógico?** Porque el compilador indica el archivo, la línea, la columna y qué falta, y además el programa no avanza hasta que se corrige, así que el error no puede pasar desapercibido. En cambio, un error lógico no genera ningún aviso y hay que descubrirlo comparando resultados.

---

## Ejercicio 2: error lógico

### Código original

Archivo `codigo/02_error_logico.cpp`:

```cpp
#include <iostream>
#include <vector>

double calcular_promedio(const std::vector<int>& notas) {
    int suma = 0;

    for (int nota : notas) {
        suma += nota;
    }

    return suma;
}

int main() {
    std::vector<int> notas = {80, 90, 100};

    double promedio = calcular_promedio(notas);

    std::cout << "Promedio: " << promedio << std::endl;

    return 0;
}
```

### Comando de compilación y ejecución

```bash
g++ -g -o error_logico codigo/02_error_logico.cpp
./error_logico
```

### Resultado observado

El programa compiló sin errores ni advertencias y se ejecutó, pero imprimió:

```text
Promedio: 270
```

- Resultado esperado: (80 + 90 + 100) / 3 = **90**
- Resultado obtenido: **270**

### Herramienta usada

Ejecución normal del programa y comparación manual contra el resultado calculado a mano.

### Análisis

La función `calcular_promedio` suma todas las notas, pero termina con `return suma;`, así que devuelve el total y nunca lo divide entre la cantidad de notas. El 270 es la suma de 80, 90 y 100, y es justo 3 veces el promedio esperado (90), lo que me ayudó a notar que faltaba dividir entre 3.

El compilador no lo detecta porque el código es válido: devolver un `int` en una función que retorna `double` es una conversión permitida. El error está en la lógica, no en las reglas del lenguaje.

### Corrección

Se divide la suma entre la cantidad de elementos de la lista:

```cpp
double calcular_promedio(const std::vector<int>& notas) {
    int suma = 0;

    for (int nota : notas) {
        suma += nota;
    }

    return static_cast<double>(suma) / notas.size();
}
```

Convertí `suma` a `double` antes de dividir para que la división conserve los decimales. Sin esa conversión, C++ haría una división entera y descartaría la parte decimal (por ejemplo, con las notas 85 y 90 daría 87 en vez de 87.5).

### Evidencia

```bash
g++ -g -o error_logico codigo/02_error_logico.cpp
./error_logico
```

```text
Promedio: 90
```

### Reflexión

1. **¿Por qué el compilador no detectó este error?** Porque el código cumple todas las reglas del lenguaje: los tipos son compatibles y no hay errores de sintaxis. El compilador no sabe qué quería calcular el programador, solo verifica que el código sea válido.
2. **¿Por qué este error se considera lógico?** Porque el programa hace lo que se le escribió, pero no lo que se quería: la función se llama "promedio" y devuelve una suma. El problema está en el razonamiento del algoritmo y no en la forma de escribirlo.
3. **¿Qué estrategia usó para encontrarlo?** Calculé a mano el resultado esperado (90), lo comparé con el obtenido (270), noté que 270 es 3 veces 90 y revisé la función para ver qué devolvía realmente, hasta encontrar que faltaba la división entre la cantidad de notas.
4. **¿Qué pruebas adicionales podría hacer?**
   - Una lista con una sola nota, donde el promedio debe ser esa misma nota.
   - Notas cuyo promedio tenga decimales, como `{85, 90}`, que debe dar 87.5.
   - Una lista vacía, que haría dividir entre cero y debería manejarse de forma especial.
   - Listas con notas iguales o con ceros.