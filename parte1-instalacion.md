# Parte 1: instalación y verificación del ambiente

## Ambiente utilizado

Trabajé en Ubuntu 24.04.3 LTS ejecutándose en WSL2 sobre Windows, ya que herramientas como `valgrind` y los sanitizers necesitan un entorno Linux.

Versiones instaladas:

| Herramienta | Versión |
|-------------|---------|
| g++ | 13.3.0 |
| gdb | 15.1 |
| valgrind | 3.22.0 |

Comandos usados para instalar y verificar:

```bash
sudo apt update
sudo apt install build-essential gdb valgrind

g++ --version
gdb --version
valgrind --version
cat /etc/os-release
```

## Programa de prueba

Archivo `codigo/prueba.cpp`:

```cpp
#include <iostream>

int main() {
    std::cout << "Ambiente listo para depuración" << std::endl;
    return 0;
}
```

Comando de compilación:

```bash
g++ -g -o prueba codigo/prueba.cpp
```

Comando de ejecución:

```bash
./prueba
```

Resultado obtenido:

```text
Ambiente listo para depuración
```

El programa compiló sin errores ni advertencias y se ejecutó correctamente, por lo que el ambiente está listo para el resto del laboratorio.

## Para qué sirve cada herramienta

- **g++:** es el compilador de C++. Traduce el código fuente (`.cpp`) a un programa ejecutable que el computador puede correr.
- **gdb:** es un depurador. Permite ejecutar un programa paso a paso, detenerlo en puntos específicos (breakpoints), revisar el valor de las variables y ver la pila de llamadas mientras corre.
- **valgrind:** es una herramienta de análisis que ejecuta el programa y vigila cómo usa la memoria. Detecta, por ejemplo, pérdidas de memoria y accesos inválidos, y también puede analizar problemas con hilos (Helgrind).

## Preguntas de reflexión

1. **¿Para qué sirve `g++`?** Para compilar programas escritos en C++, es decir, convertir el código fuente en un ejecutable. También avisa de errores de sintaxis y permite activar opciones como los símbolos de depuración o los sanitizers.

2. **¿Para qué sirve `gdb`?** Para depurar un programa mientras se ejecuta: detenerlo en un punto, avanzar línea por línea, inspeccionar variables y ver qué funciones lo llevaron hasta ahí. Es útil para entender por qué falla o por qué da un resultado incorrecto.

3. **¿Para qué sirve `valgrind`?** Para detectar errores de memoria en tiempo de ejecución, como memoria que se reserva y nunca se libera, lecturas o escrituras fuera de los límites de un arreglo y uso de memoria ya liberada. También tiene herramientas para detectar condiciones de carrera en programas con hilos.

4. **¿Por qué se recomienda compilar con `-g` al depurar?** Porque `-g` agrega al ejecutable información de depuración: los nombres de variables y funciones y la correspondencia entre las instrucciones y las líneas del código fuente. Sin ella, `gdb` y `valgrind` solo podrían mostrar direcciones de memoria, y no los nombres de las variables ni los números de línea donde ocurre el problema.

5. **¿Qué diferencia hay entre compilar un programa y depurarlo?** Compilar es generar el ejecutable a partir del código y detectar errores de sintaxis. Depurar es analizar el programa ya compilado cuando falla o da resultados incorrectos, para encontrar la causa. Que un programa compile no significa que sea correcto: puede tener errores de lógica o de memoria que solo se descubren al ejecutarlo y depurarlo.