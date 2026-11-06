# Notas del Día 2: códigos de salida, scripts, set y pipes

Borrador para reescribir con mis propias palabras antes de subirlo a GitHub como `dia2.md`.

## 1. Códigos de salida

- Todo comando deja un número al terminar: **0 = éxito**, cualquier otro = fallo. `$?` guarda el del último comando.
- Ejemplos: `true` → 0, `false` → 1, `ls /nonexistent` → 2, comando que no existe → 127, sin permiso de ejecución → 126, cortado con Ctrl + C → 130.
- Cada programa decide qué número usa; se documenta en su manual (`man ls`, sección *Exit status*).
- **No confundir** con los descriptores de archivo (0 stdin, 1 stdout, 2 stderr), que van pegados a `<` y `>` como en `2>&1`.
- `echo $?` hay que escribirlo **justo después** del comando, porque el propio `echo` cambia el valor.

## 2. Unir comandos: `;`, `&&`, `||` y `|`

| Operador | Qué hace |
| --- | --- |
| `;` | Ejecuta uno y luego el otro, siempre |
| `&&` | El segundo solo si el primero tuvo éxito (código 0) |
| `\|\|` | El segundo solo si el primero falló |
| `\|` | Pasa la **salida** del primero como **entrada** del segundo (es otro mecanismo) |
| `&` | Manda el comando a segundo plano (no confundir con `&&`) |

- Ejercicio: crear una carpeta solo si no existe: `[ -d /tmp/mydir ] || mkdir /tmp/mydir`.
- `[ -d ruta ]` sale con 0 si existe y es carpeta; `[ -f ruta ]` si es un archivo normal; `[ -z "$x" ]` si el texto está vacío. Los espacios dentro de los corchetes son obligatorios.
- `mkdir -p` hace lo mismo en un solo comando.

## 3. Scripts y condiciones

- `$1`, `$2`... son los argumentos que recibe un script. Entrecomillar siempre: `"$1"`.
- Estructura:

```bash
#!/bin/bash
if [ -f "$1" ]; then
    echo "El archivo $1 existe"
else
    echo "El archivo $1 no existe"
fi
```

- En una sola línea hay que separar con `;`: `if [ -f x ]; then echo a; else echo b; fi`. Sin los `;`, bash muestra el prompt `>` esperando que se cierre el `if` (se cancela con Ctrl + C).
- **Permisos:** un archivo nuevo no se puede ejecutar. `ls -l` muestra `-rw-rw-r--` (sin `x`) y `./script.sh` da `Permission denied` (código 126). Con `chmod +x script.sh` aparecen las `x` y ya funciona.
- `bash script.sh` funciona aunque no tenga `x`, porque bash solo necesita **leer** el archivo.
- `./` es necesario porque la carpeta actual no está en el `PATH`.

## 4. Sustitución de comandos

- `$(comando)` ejecuta el comando y pone su resultado en ese lugar: `echo "Hoy es $(date)"`.
- `date +%Y-%m-%d` da la fecha como `2026-11-06`.
- Backup con fecha: `cp notes.txt "notes_$(date +%Y-%m-%d).txt"`.
- Con comillas dobles se ejecuta el `$( )`; con comillas simples sale literal.
- `cp -i` pregunta antes de sobrescribir; `cp -n` no sobrescribe nunca.

## 5. Parámetros especiales

| Parámetro | Qué contiene |
| --- | --- |
| `$?` | Código de salida del último comando |
| `$#` | Número de argumentos |
| `"$@"` | Todos los argumentos, cada uno por separado (el que casi siempre se quiere) |
| `"$*"` | Todos los argumentos unidos en una sola palabra |
| `$0` | Nombre del script o de la shell |
| `$$` | PID de la shell actual |
| `$!` | PID del último proceso en segundo plano |
| `$-` | Opciones de la shell activas |

## 6. Shell, bash y comandos internos

- **Shell** es el tipo de programa; **bash** es una shell concreta (otras: sh/dash, zsh, fish, PowerShell). La terminal es la ventana; la shell corre dentro.
- `cd` es un **builtin** porque debe cambiar el directorio de la propia shell. Un programa externo es un proceso hijo y solo cambiaría el suyo. Lo mismo con `export`, `source`, `exit` y `set`.
- `type nombre` dice si es builtin, alias o programa; `help cd` explica un builtin.

## 7. El comando `set`

- `set` **cambia las reglas de la shell**. `-` enciende una regla y `+` la apaga.
- `set -x` muestra cada comando antes de ejecutarlo (con `+` delante). `bash -x script.sh` hace lo mismo solo para esa ejecución.
- `set -u` da error con variables sin definir. `set -e` detiene la shell al primer fallo (**no probarlo directo en la terminal**; usar `bash -c 'set -e; false; echo no sale'`).
- `set -euo pipefail` es el modo estricto que se pone como segunda línea de casi todo script. Sin la palabra `set` delante no funciona, porque bash tomaría `-euo` como un comando.
- `set -- a b` guarda valores en `$1` y `$2`; `set` solo lista las variables.
- Ojo: `set -x` imprime las variables ya sustituidas, así que puede dejar contraseñas en los logs.
- Cada `-x` o `-p` significa algo distinto según el comando (`chmod +x`, `mkdir -p`, `ps -p`, `tar -x`).

## 8. Mi pipe de las 5 extensiones más comunes

```bash
find ~ -type f -printf "%f\n" | awk -F. '{print $NF}' | sort | uniq -c | sort -nr | head -n 5
```

- `find ~ -type f` lista los archivos de mi carpeta personal.
- `-printf "%f\n"` imprime solo el nombre (sin ruta), con salto de línea (el `\n` va **dentro** de las comillas).
- `awk -F. '{print $NF}'` parte cada nombre por los puntos y se queda con el último trozo (`$NF`).
- `sort` agrupa las extensiones iguales; `uniq -c` las cuenta (solo cuenta líneas seguidas, por eso va después de `sort`).
- `sort -nr` ordena por número, de mayor a menor; `head -n 5` se queda con las 5 primeras.
- **Pendiente de afinar:** un nombre sin punto (`known_hosts`) o uno oculto (`.gitconfig`) también pasan por `awk` y pueden colarse como "extensiones".

## 9. Errores que cometi y qué aprendí

| Qué pasó | Causa | Lección |
| --- | --- | --- |
| `Luis: command not found` | Escribí `nombre= Luis` con un espacio | En bash las asignaciones van sin espacios: `nombre=Luis` |
| El `if` en una línea se quedó en `>` | Faltaban los `;` antes de `else` y `fi` | Separar con `;` o con saltos de línea |
| `PTY allocation request failed` | Usé `ssh -t` en vez de `ssh -T` | Las mayúsculas y minúsculas cambian el significado de una opción |
| `ls /nonexistent/tmp` no salió como esperaba | Sin espacio era una sola ruta | Los argumentos se separan con espacios |
| `find: paths must precede expression` | Faltó `-name` / puse `%p` suelto | Cada condición de `find` lleva su opción; los `%` solo funcionan dentro de `-printf` |
| Los nombres salieron todos pegados | `-printf` no añade salto de línea | Añadir `\n` dentro de las comillas |
| Bash se comía el `\n` | Estaba fuera de las comillas | La barra invertida fuera de comillas escapa el siguiente carácter |

## 10. Dudas pendientes

- Entender mejor cuándo conviene cada opción de `set`.
- Decidir qué hacer con los archivos sin extensión y los ocultos en el conteo.
- Práctica con expresiones regulares (`grep`, `sed`), todavía casi sin usar.
- Subir los scripts `check.sh` y `existe.sh` a `linux-lab/scripts`.

## 11. Cómo trabajo los ejercicios

1. Convertir la pista en una lista de herramientas.
2. Aprender cada una con ejemplos mínimos (`man`, `--help`, `whatis`, explainshell.com).
3. Describir en español qué transformación necesito antes de escribir el comando.
4. Construir el pipe de izquierda a derecha, una pieza a la vez, mirando la salida de cada etapa.
5. Pedir ayuda solo después de un intento real, y pidiendo una pista en vez de la solución.
