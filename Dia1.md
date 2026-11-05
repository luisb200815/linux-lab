# Notas del Día 1: Linux, terminal y primeros pasos en DevOps

Resumen de lo esencial que practiqué: entorno, shell, redirecciones, globs, Git y errores comunes.

## 1. Mi entorno de trabajo

- VM con Ubuntu en VirtualBox/VMware, conectada por **SSH desde VS Code** (Remote-SSH).
- Trabajo en la **terminal de VS Code** (Terminal → New Terminal), no en la ventana negra de la VM. Es más cómoda y permite copiar/pegar y dividir la pantalla.
- Pantalla dividida: **Windows + flecha izquierda/derecha** (con el cursor fuera de la ventana de la VM). El video puede ir en Imagen en imagen.
- Anotar siempre **usuario, IP de la VM y contraseña** en un lugar seguro (nunca en GitHub).
- Hábitos: hacer un **snapshot** de la VM cuando todo funciona, y apagarla con `sudo shutdown now`.
- Dato útil: `whoami` muestra el usuario y `hostname` el nombre de la máquina.

## 2. Ideas clave de DevOps

- **DevOps:** crea el camino seguro y automático para que el software llegue a los usuarios y siga funcionando (CI/CD, infraestructura como código, monitoreo).
- **Pipeline:** cadena automática de pasos (build → pruebas → análisis → publicación → despliegue). Si un paso falla, se detiene.
- **CI:** integrar y probar cada cambio automáticamente. **CD:** entregar o desplegar lo que pasó las pruebas.
- **Imagen de Docker:** paquete con la app y todo lo que necesita (como un molde). **Contenedor:** esa imagen en ejecución (las galletas). Se guardan en un **registry**.
- **Dockerfile:** archivo de texto con las instrucciones para construir una imagen.

## 3. Cómo funciona el shell

- Ubuntu usa **bash** por defecto (`echo $SHELL` → `/bin/bash`). `sh` en Ubuntu es un enlace a **dash**.
- **PATH:** lista de carpetas (separadas por `:`) donde Linux busca los programas. `echo $PATH` la muestra.
- `which comando` dice dónde está un programa. `which -a comando` muestra **todas** las ubicaciones, en orden.
- **bin** guarda programas ejecutables: `/bin`, `/usr/bin`, `/usr/local/bin` (instalados a mano), `/sbin` (administración).
- `command not found` suele significar que su carpeta no está en el PATH.
- **Raíz `/`:** todo cuelga de ahí. `/home` (usuarios), `/etc` (configuración), `/var/log` (logs), `/usr` (programas), `/tmp` (temporales), `/proc` (info de procesos en vivo).
- Moverse: `pwd`, `cd /`, `cd ~`, `cd ..`, `cd -`, `ls -l`.
- Para saber qué hace un comando: `man comando`, `comando --help`, `whatis comando`, `apropos "tema"`.
- No hace falta memorizar todos los archivos del sistema: lo importante es saber dónde buscar y cómo averiguar.

## 4. Comandos que usé

| Comando | Para qué sirve |
| --- | --- |
| `mkdir -p ruta` | Crea carpetas; no falla si ya existe y crea las intermedias |
| `touch archivo` | Crea un archivo vacío o actualiza su fecha |
| `touch -d "40 days ago" archivo` | Le pone una fecha de modificación inventada |
| `echo "texto" > archivo` | Escribe (sobrescribe) en un archivo |
| `echo "texto" >> archivo` | Añade al final sin borrar |
| `cat archivo` | Muestra el contenido de un archivo de texto |
| `less`, `head`, `tail` | Ver archivos largos, el inicio o el final |
| `find . -type f -name "*.txt"` | Busca archivos por nombre |
| `grep "texto" archivo` | Busca texto dentro de archivos |
| `rm`, `rm -r` | Borra (sin papelera, con cuidado) |
| `pkill nombre` | Detiene procesos por nombre |
| `nvim archivo` / `nano archivo` | Editores de texto en la terminal |

## 5. Argumentos, globs y regex

- **Argumentos:** valores que recibe un comando (`cp a.txt b.txt`). Las **opciones** (`-l`, `-p`) son argumentos con guion que cambian el comportamiento.
- **Glob (comodín):** patrón para elegir **nombres de archivo**. Lo expande **bash** antes de ejecutar el comando.
  - `*` cualquier cantidad de caracteres, `?` exactamente uno, `[ab]` uno de esos, `{a,b,c}` genera varias opciones (brace expansion).
- **Regex:** patrón para buscar **texto dentro de archivos**. Lo interpreta el programa (`grep`, `sed`, `awk`).
  - `.` cualquier carácter, `*` repite lo anterior, `^` inicio de línea, `$` final, `[0-9]` un dígito.
- **Trampa:** el `*` significa cosas distintas en glob ("cualquier cosa") y en regex ("repite lo anterior"; para "cualquier cosa" se usa `.*`).
- `echo *.txt` muestra lo que realmente recibe el comando tras la expansión.
- Con `find`, poner el patrón **entre comillas** y con `-name`, para que lo interprete `find` y no bash.

## 6. Comillas

- `'simples'`: todo literal, no se expande nada.
- `"dobles"`: expande `$VARIABLE` y `$(comando)`; `\n` no se convierte en salto de línea.
- `$'ANSI'`: interpreta `\n`, `\t` y `\'`, pero no expande variables.
- Ejemplo con `$` literal, `!` y salto de línea: `echo $'Cuesta $5\n¡Ojo!'`.
- El `!` en la terminal interactiva puede activar el historial (`event not found`); se desactiva con `set +H`.

## 7. Flujos estándar y redirecciones

- **stdin (0)** entrada, **stdout (1)** salida normal, **stderr (2)** errores.
- `cmd > salida.txt 2> errores.txt` separa salida y errores en dos archivos.
- `cmd > todo.txt 2>&1` (o `&> todo.txt`) manda ambos al mismo archivo.
- **El orden importa:** `2>&1 > archivo` no hace lo mismo que `> archivo 2>&1`, porque bash procesa de izquierda a derecha.
- `2>/dev/null` descarta los errores. `|` conecta la salida de un comando con la entrada del siguiente.

## 8. Scripts: lo que entendí

- `#!/bin/bash` (shebang) indica con qué intérprete se ejecuta el script.
- `set -euo pipefail`: se detiene ante errores (`-e`), falla con variables inexistentes (`-u`) y detecta fallos dentro de pipes (`pipefail`).
- `comando &` lo manda a segundo plano; `$!` guarda el PID del último proceso en segundo plano.
- `$(comando)` inserta su resultado en un texto. `while comando; do ... done` repite mientras el comando tenga éxito.
- `stress` satura la CPU a propósito para provocar fallos en pruebas intermitentes (flaky tests) que dependen del tiempo. Usar con cuidado en mi VM (por ejemplo `stress --cpu 1 --timeout 60`).

## 9. Git y GitHub

- Configuración única: `git config --global user.name "..."` y `user.email "..."`.
- Conexión con GitHub por **llave SSH**. Se comprueba con `ssh -T git@github.com` (**T mayúscula**; la `-t` minúscula da el error `PTY allocation request failed`). El saludo "Hi usuario!" significa éxito.
- Flujo básico: `git clone` → crear o copiar archivos → `git status` → `git add archivo` → `git commit -m "mensaje"` → `git push`.
- Errores típicos: `Permission denied (publickey)` (falta la llave en GitHub), `rejected ... fetch first` (hacer `git pull`), `nothing to commit` (falta `git add`).

## 10. Errores que cometí y qué aprendí

| Qué pasó | Causa | Lección |
| --- | --- | --- |
| La VM se puso lenta y SSH mostró "reconnecting" | Dejé un `grep -r` corriendo sobre `/` | Detener con **Ctrl + C** o `pkill grep`; buscar en carpetas concretas y excluir `/proc` |
| Cientos de `Permission denied` | `/proc` no se puede leer sin privilegios | Es normal; usar `2>/dev/null` |
| `find: paths must precede expression` | Faltaba `-name` | Cada condición de `find` lleva su opción |
| `ls /nonexistent/tmp` no mostró dos cosas | Escribí la ruta sin espacio | Los argumentos se separan por espacios |
| No me dejaba escribir en la terminal de VS Code | La terminal no tenía el foco | Hacer clic dentro de ella |
| Ventana de login al usar Windows + flecha | La VM capturaba el teclado | Trabajar desde VS Code, no desde la ventana de la VM |
| Avisos del panel Problems | Archivos internos de `~/.vscode-server` | Son recomendaciones, se pueden ignorar; abrir una carpeta de trabajo concreta |

## 11. Reglas de seguridad que me llevo

- Practicar dentro de mi carpeta personal (`~`), no en carpetas del sistema.
- Pensar dos veces antes de usar `sudo`, `rm -r` o comandos recursivos sobre `/`.
- No copiar comandos que no entienda: revisarlos con `man` o explainshell.com.
- Nunca compartir la llave privada SSH (`id_ed25519`), solo la `.pub`.

## 12. Extra: trigger de auditoría en SQL Server

- Un **trigger** se ejecuta automáticamente tras un `UPDATE`. Dentro tiene las tablas especiales `inserted` (valores nuevos) y `deleted` (valores anteriores).
- `UPDATE(columna)` es verdadero si el `UPDATE` incluía esa columna; el filtro `d.col <> i.col` asegura que solo se registren cambios reales.
- Guarda quién (`SYSTEM_USER`), cuándo (`GETDATE()`), qué cambió y el valor anterior/nuevo. Ojo: con valores `NULL` la comparación `<>` no detecta el cambio.
- Mismo concepto de auditoría que verá en DevOps: logs, historial de Git, registro de cambios.

## 13. Próximos pasos

- Terminar los ejercicios de **Course Overview + Introduction to the Shell** (Missing Semester).
- Seguir con **Command-line Environment**.
- Practicar `man`, `grep`, `find` y redirecciones con ejemplos propios.
- Complementar con Linux Journey y OverTheWire: Bandit.
- Subir estas notas a mi repositorio y repasar mañana los comandos de memoria.
