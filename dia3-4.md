# Notas de los días 3 y 4: xargs, curl, jq, awk y SSH con NAT

Borrador para reescribir con mis propias palabras antes de subirlo a GitHub. Complementa `Dia2.md`.

## 1. xargs: convertir líneas en argumentos

- Un pipe (`|`) entrega el texto por **stdin**, pero comandos como `cat`, `wc` o `rm` esperan **nombres de archivo como argumentos**. `xargs` hace esa conversión.
- Experimento:
  - `echo /etc/hostname | cat` imprime el texto `/etc/hostname` (cat solo repite su entrada).
  - `echo /etc/hostname | xargs cat` ejecuta `cat /etc/hostname` y muestra el nombre de la máquina (`lab`).
- **Ruta relativa vs absoluta:** `etc/hostname` (sin barra) busca dentro de la carpeta actual y da `No such file or directory`; `/etc/hostname` parte de la raíz.
- `xargs` por defecto separa por **espacios y saltos de línea**, así que un archivo llamado `mi script.sh` se parte en `./mi` y `script.sh` y da errores.
- Solución para nombres con espacios: **`find ... -print0 | xargs -0 ...`**. `-print0` termina cada nombre con un carácter nulo (el único que no puede estar en un nombre de archivo) y `-0` le dice a `xargs` que lea con ese separador. Las dos opciones van siempre juntas.

```bash
find . -type f -name "*.sh" -print0 | xargs -0 wc -l
```

- Resultado con mis archivos de prueba: `3 ./mi script.sh`, `1 ./a.sh`, `2 ./b.sh` y `6 total`.
- Con `-printf '%f\n'` solo salen los nombres sin ruta, y `wc` no encuentra archivos que estén en subcarpetas. Para pasar archivos a otro comando conviene conservar la ruta (`-print` o `-print0`).

## 2. print, printf y wc -l

- `print` (en `find` y `awk`) imprime y añade salto de línea. `printf` imprime con formato y **no añade salto de línea**: hay que poner `\n`.
- En `find`: `-print` (ruta + salto de línea, es el valor por defecto), `-print0` (ruta + carácter nulo) y `-printf 'formato'` (con códigos `%`, por ejemplo `%f` para el nombre sin ruta).
- En bash: `printf "%s tiene %d años\n" Luis 20`. Con comillas simples, el `\n` le llega intacto a `printf`.
- `wc -l` cuenta **saltos de línea**, no líneas visibles: `printf 'hola\nmundo' | wc -l` da 1.
- Otras opciones de `wc`: `-w` (palabras), `-c` (bytes), `-m` (caracteres).
- El punto de `find .` es **dónde empieza a buscar** (la carpeta actual); `find ~` busca en mi carpeta personal.

## 3. curl y grep: contar las clases de la web del curso

- `curl -s URL` descarga el HTML en silencio (sin barra de progreso), ideal para usar en un pipe.
- **`head` va al final de la cadena, nunca en medio.** `curl -s URL | head | grep ...` solo busca en las primeras 10 líneas y no encuentra nada. `head` sirve para mirar mientras exploro y se quita cuando paso a filtrar o contar.
- Para encontrar el patrón: buscar el título de una clase con `grep -n`, comparar la línea con la de otra clase y quedarme con lo que se repite en todas. Las clases son líneas como:
  `<a href="/2026/course-shell/">Course Overview + Introduction to the Shell</a>`
- Patrón que funciona: `grep -n '<a href="/2026/'`. Va entre **comillas simples** porque el texto lleva comillas dobles dentro (con dobles quedaba una comilla sin cerrar y bash mostraba el prompt `>`).
- Con `-n` se ven 9 clases (líneas 114, 123, 132, ..., 186).
- `grep -c` cuenta las **líneas** que coinciden y muestra solo el número. `-nc` da lo mismo que `-c`, porque con `-c` no se muestran líneas a las que numerar. `grep -c patrón` equivale a `grep patrón | wc -l`.
- Me salió **10** y no 9: probablemente un enlace del menú que también lleva `/2026/`. Pendiente de afinar el patrón.

## 4. jq: procesar JSON

- JSON es texto con estructura. Si empieza con `[`, es una lista; cada elemento es un objeto con campos (`name`, `version`...).
- `jq .` reescribe el JSON legible. `.[0]` es el primer elemento. `.[]` recorre la lista elemento por elemento.
- `.[] | .name` muestra el nombre de cada elemento (entre comillas, porque son texto). Con `jq -r` salen sin comillas.
- **`select(condición)`** deja pasar solo los elementos que cumplen la condición. Dentro, `.` es el elemento actual.
- Estructura del ejercicio: `jq '.[] | select(.version > 6) | .name'`.
- Para verificarlo, mostrar `[.name, .version]` y comprobar que todas las versiones son mayores que 6. Si `version` fuera texto con comillas, `jq` lo consideraría mayor que cualquier número y el filtro no filtraría.
- **Por qué no `grep`:** ve líneas de texto y no entiende números ni campos. No sabe comparar `7.2 > 6` ni relacionar `version` con el `name` del mismo elemento.
- Si falta `jq`: `sudo apt update && sudo apt install jq`.

## 5. awk: filtrar y manipular columnas

- **awk** viene de las iniciales de sus creadores: Aho, Weinberger y Kernighan.
- Parte cada línea en columnas por espacios: `$1`, `$2`, `$3`... (`$0` es la línea completa, `$NF` la última). Con `-F.` el separador es un punto.
- Forma general: `awk 'CONDICIÓN {ACCIÓN}'`. Solo condición imprime las líneas que la cumplen.
- Mi filtro, bien escrito (una sola pareja de comillas simples que contiene condición y acción):

```bash
printf 'a 50 x\nb 150 y\nc 200 z\n' | awk '$2 > 100 {print}'
```

- Salida: `b 150 y` y `c 200 z`.
- Errores de comillas que cometí: poner dos `printf` en la misma línea y dejar `{print}` fuera de las comillas de `awk`.
- Pendiente: imprimir esas líneas con la primera y la tercera columna intercambiadas (`print` con las columnas en otro orden).

## 6. Red de la VM: de modo puente a NAT con reenvío de puerto

- **Problema:** en modo puente (*Bridged Adapter*) la VM pide IP al router de cada red, así que al cambiar de wifi la IP cambiaba y VS Code daba `The connection timed out`.
- **Solución:** adaptador en **NAT** con reenvío de puertos en VirtualBox: Name `ssh`, TCP, Host IP `127.0.0.1`, **Host Port 2222**, Guest IP vacía, **Guest Port 22**.
- Un puerto es como una extensión de centralita. `127.0.0.1` significa "este mismo equipo" y no cambia con ningún wifi. Con NAT, la VM queda en una red privada (`10.0.2.15`).
- Probar desde Windows: `ssh -p 2222 luismostro@127.0.0.1`. Aquí `-p` (minúscula) es el puerto; con `scp` se usa `-P`.
- Archivo de configuración en Windows, `C:\Users\luisb\.ssh\config` (sin extensión `.txt`):

```
Host lab
  HostName 127.0.0.1
  Port 2222
  User luismostro
```

- Con eso, `ssh lab` funciona y VS Code se conecta con **Connect to Host... → lab**. Abajo a la izquierda debe decir `SSH: lab`.
- `ssh lab` se ejecuta **en Windows**. Si lo escribo dentro de la VM, se conecta a sí misma (`127.0.1.1`) y pregunta por una huella desconocida.
- `Connection refused` en `127.0.0.1:2222`: la VM estaba **apagada**. VirtualBox solo abre ese puerto mientras la VM está encendida (*Running*).
- Rutina diaria: encender la VM → esperar al login → abrir VS Code → conectar a `lab` → abrir `/home/luismostro/linux-lab`.
- Si hay dos ventanas de VS Code, la conectada es la que muestra `SSH: lab`; la otra se puede cerrar.

## 7. Errores que cometí y qué aprendí

| Qué pasó | Causa | Lección |
|---|---|---|
| `Bad port 'luismostro@127.0.0.1'` | Escribí `ssh -p usuario@ip` sin el número de puerto | `-p` necesita un número justo después |
| `Connection refused` | La VM estaba apagada | Encender la VM antes de conectar |
| `cat: etc/hostname: No such file` | Ruta sin la barra inicial | Ruta absoluta empieza con `/` |
| `wc: ./mi: No such file` | `xargs` partió `mi script.sh` por el espacio | Usar `-print0` y `-0` juntos |
| `grep` no encontraba nada con `curl \| head \| grep` | `head` en medio cortaba los datos | `head` va al final |
| Prompt `>` esperando más texto | Comilla doble sin cerrar | Usar comillas simples si el texto lleva comillas dobles |
| `grep -c` dio 10 y no 9 | El patrón también atrapa un enlace del menú | Revisar las líneas con `-n` antes de contar |
| Línea de awk sin funcionar | Dos `printf` y `{print}` fuera de las comillas | Condición y acción dentro de las mismas comillas |
| `ssh lab` pedía confirmar la huella | Lo ejecuté dentro de la VM | Se ejecuta desde Windows |

## 8. Cómo estudio los ejercicios

1. Convertir la pista en una lista de herramientas.
2. Aprender cada una con ejemplos mínimos (`man`, `--help`, `whatis`).
3. Describir en español la transformación antes de escribir el comando.
4. Construir el pipe de izquierda a derecha y mirar la salida de cada etapa.
5. Pedir una pista, no la solución, después de intentarlo.

## 9. Pendiente

- [ ] Afinar el patrón de `curl | grep` para que dé 9 clases y escribir la versión con `| wc -l`.
- [ ] Terminar el filtro de `jq` con `select(.version > 6)` y verificarlo con `[.name, .version]`.
- [ ] Imprimir con `awk` las líneas filtradas con las columnas 1 y 3 intercambiadas.
- [ ] Desarmar el pipeline del log de SSH de la clase (copiarlo de las notas de la lección).
- [ ] Construir un pipe con `~/.bash_history` para ver qué comandos uso más.
- [ ] Subir estas notas y los scripts a `linux-lab` en GitHub.
