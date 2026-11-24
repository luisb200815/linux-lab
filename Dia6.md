# Día 6: Debugging y Profiling

> *"El código no hace lo que esperas que haga, sino lo que le dices que haga."* — Regla de oro de la programación.

---

## Parte 1: Debugging (Depuración)

### 1. Printf Debugging y Logging
* **Print debugging:** Primera línea de defensa. Consiste en insertar sentencias de impresión estratégicas para observar el estado del programa.
* **Logging ("imprimir con más cuidado"):** Usa frameworks de registro que ofrecen:
  * Redirección de salidas (archivos, consola, servicios remotos).
  * Niveles de severidad (`DEBUG`, `INFO`, `WARN`, `ERROR`) para filtrar mensajes.
  * **Logging estructurado** (por ejemplo, JSON o CSV) para facilitar el análisis posterior.
* **Buena práctica:** Cuando resuelvas un error usando `print`, considera convertirlos en *logs* permanentes antes de borrarlos.

### 2. Debuggers (`gdb`, `lldb`, `pdb`, `jdb`)
Útiles cuando no sabes qué imprimir, el error es difícil de reproducir o reiniciar el programa es costoso. Permiten pausar la ejecución, avanzar paso a paso e inspeccionar memoria, pila (*stack*) y registros.

| Comando GDB | Acción |
| :--- | :--- |
| `run` | Inicia el programa |
| `b {función}` o `b {archivo}:{línea}` | Establece un punto de ruptura (*breakpoint*) |
| `c` | Continúa la ejecución |
| `step` / `next` / `finish` | Entrar en función / Pasar por encima / Salir de función |
| `p {variable}` | Imprime el valor de una variable |
| `bt` | Muestra el *backtrace* (pila de llamadas) |
| `watch {expresión}` | Pausa cuando cambia el valor de una expresión |

### 3. Record-Replay Debugging (`rr`)
Ideal para **Heisenbugs** (errores que cambian o desaparecen al intentar observarlos, como condiciones de carrera o fallos intermitentes).

```bash
rr record ./mi_programa   # Graba una ejecución determinista
rr replay                 # Reproduce la grabación dentro de GDB
```

Comandos de depuración inversa en `rr replay`:
* `reverse-continue` (`rc`): Retrocede hasta un *breakpoint* o *watchpoint*.
* `reverse-step` (`rs`): Retrocede una línea entrando en funciones.
* `reverse-next` (`rn`): Retrocede una línea saltando llamadas a funciones.
* `reverse-finish`: Retrocede hasta el inicio de la función actual.

*Nota sobre concurrencia:* `rr` serializa la planificación de hilos para lograr determinismo, por lo que algunas condiciones de carrera pueden requerir varios intentos de grabación hasta capturar el fallo.

### 4. Trazado de Llamadas al Sistema (System Call Tracing)
Permite ver cómo interactúa un programa con el kernel (abrir archivos, asignar memoria, crear procesos).

* **`strace` (Linux) / `dtruss` (macOS):** Rápido de usar para un proceso puntual.
  ```bash
  strace ./mi_programa               # Todas las llamadas al sistema
  strace -e trace=file ./mi_programa # Solo operaciones de archivos
  strace -f ./mi_programa            # Seguir procesos hijos
  strace -p <PID>                    # Adjuntarse a un proceso activo
  strace -T ./mi_programa            # Mostrar tiempo de cada llamada
  ```
* **`bpftrace` y eBPF:** Ejecuta programas seguros dentro del kernel con muy bajo impacto (*overhead*). Requiere `root` y es ideal para agregaciones, conteos y monitoreo global o filtrado:
  ```bash
  sudo bpftrace -e 'tracepoint:syscalls:sys_enter_* /pid == cpid/ { @[probe] = count(); }' -c 'ls -la'
  ```

### 5. Debugging de Red
* **`tcpdump` y Wireshark:** Capturan y analizan paquetes de red (`sudo tcpdump -i any -w capture.pcap`).
* **Tráfico HTTPS:** Al estar cifrado, `tcpdump` es limitado. Conviene usar `mitmproxy` o la pestaña **Network** de las herramientas de desarrollador del navegador.

### 6. Debugging de Memoria
* **Sanitizers (en tiempo de compilación):** Rápidos e ideales para desarrollo y CI.
  * **ASan** (`-fsanitize=address`): *Buffer overflows*, *use-after-free*, fugas de memoria.
  * **TSan** (`-fsanitize=thread`): *Data races* en código multihilo.
  * **MSan** (`-fsanitize=memory`): Lecturas de memoria no inicializada.
  * **UBSan** (`-fsanitize=undefined`): Comportamiento indefinido (ej. *integer overflow*).
* **Valgrind:** No requiere recompilar (útil para binarios sin código fuente o librerías de terceros), aunque es más lento:
  ```bash
  valgrind --leak-check=full ./mi_programa
  ```

### 7. IA (LLMs) para Debugging
* **Fortalezas:** Explicar errores crípticos de compilación (C++, Rust), depurar entre fronteras de lenguajes/FFI, correlacionar síntomas vagos con causas raíz y analizar *stack traces*.
* **Limitaciones:** Pueden alucinar explicaciones plausibles pero incorrectas o sugerir parches que solo ocultan el error. Siempre verifica con herramientas reales.

---

## Parte 2: Profiling (Análisis de Rendimiento)

> *"La optimización prematura es la raíz de todo mal."* Antes de optimizar, mide dónde se consumen realmente el tiempo y los recursos.

### 1. Medición de Tiempo (`time`)
* **Real:** Tiempo de reloj de pared (*wall clock*) de inicio a fin, incluyendo esperas (red, disco, otros procesos).
* **User:** Tiempo de CPU ejecutando código propio del programa.
* **Sys:** Tiempo de CPU ejecutando llamadas dentro del kernel.

### 2. Monitoreo de Recursos
* **General:** `htop`, `btop` (`F6` ordenar, `t` árbol, `h` hilos).
* **Disco (I/O):** `iotop`.
* **Memoria:** `free`.
* **Archivos abiertos:** `lsof`.
* **Red:** `ss` (puertos y conexiones), `nethogs` e `iftop` (consumo de red por proceso).

### 3. Visualización de Datos
Registrar métricas en formatos limpios (CSV o JSON) permite graficarlas y detectar patrones invisibles en tablas:
* **`gnuplot`:** Gráficos rápidos desde la terminal.
* **`matplotlib` (Python) y `ggplot2` (R):** Exploración iterativa y gráficos por facetas.

### 4. Profilers de CPU y Memoria
* **Sampling Profilers (`perf`):** Muestrean periódicamente la pila del programa. Bajo *overhead*, ideales para producción. Se visualizan fácilmente con **Flame Graphs** (eje Y = jerarquía de llamadas, eje X = proporción de tiempo).
* **Tracing Profilers (`valgrind --tool=callgrind`):** Registran cada llamada exacta e instrucciones. Más lentos, pero exactos; se analizan con `callgrind_annotate` o `kcachegrind`.
* **Memory Profilers (`valgrind --tool=massif`):** Miden el uso de memoria *heap* a lo largo del tiempo (`ms_print massif.out.<pid>`).

### 5. Benchmarking (`hyperfine`)
Compara de forma estadística el rendimiento de dos o más comandos o implementaciones, soportando rondas de calentamiento (`--warmup`) para evitar sesgos por caché fría.

---

## Parte 3: Resolución de los Ejercicios de Profiling

### Ejercicio 1: Estadísticas básicas con `perf stat`

**Objetivo:** Obtener contadores de rendimiento de un programa y entender qué significa cada métrica.

**Problema encontrado y solución:**
Al ejecutar `perf stat` directamente, apareció el error `Unable to find PMU or event on a PMU of 'cpu_atom'` (común en VMs, WSL o kernels que no exponen métricas híbridas Intel *topdown*). Se solucionó especificando los eventos manualmente (o usando `--cputype core`):
```bash
perf stat -e task-clock,context-switches,cpu-migrations,page-faults,cycles,instructions <comando>
```

**Significado de los contadores principales:**
* **`task-clock` y `CPUs utilized`:** Tiempo total de CPU consumido por el proceso. Si es cercano al tiempo *elapsed*, usó un núcleo casi al $100\%$.
* **`context-switches`:** Veces que el sistema operativo pausó el proceso para ceder la CPU a otro.
* **`cpu-migrations`:** Veces que el proceso fue movido de un núcleo físico a otro (un valor de `0` es ideal porque conserva la caché del núcleo).
* **`page-faults`:** No son errores; ocurren cuando el programa accede por primera vez a memoria virtual y el kernel la mapea en RAM física bajo demanda.
* **`cycles` e `instructions`:** Latidos del reloj de la CPU frente a instrucciones de máquina completadas.
* **`insn per cycle` (IPC):** Instrucciones ejecutadas por cada ciclo. Un valor bajo ($< 1$) indica que la CPU pasó tiempo esperando datos (memoria o disco).
* **`user` vs `sys`:** Tiempo en código del programa frente a tiempo en llamadas al kernel (por ejemplo, `find` pasa mucho tiempo en `sys` leyendo directorios).

---

### Ejercicio 2: `perf record`, `perf report` y Flame Graph con `slow.c`

**1. Preparación y compilación:**
Guardamos `slow.c` corrigiendo los saltos de línea después de los `#include` y compilamos con símbolos de depuración (`-g`), optimización (`-O2`) y librería matemática (`-lm`):
```bash
gcc -g -O2 slow.c -o slow -lm
```

**2. Problema de permisos encontrado y solución:**
Al ejecutar `perf record -g ./slow`, el sistema bloqueó el acceso porque `kernel.perf_event_paranoid` estaba en `3`. Lo relajamos temporalmente (se restablece al reiniciar):
```bash
sudo sysctl -w kernel.perf_event_paranoid=1
perf record -g ./slow
```
*(Los avisos sobre `/proc/kallsyms` se pueden ignorar porque `slow` trabaja en espacio de usuario).*

**3. Análisis con `perf report`:**
Al abrir `perf report` comparamos:
* **`Children`:** Tiempo total de la función más todo lo que ella llama.
* **`Self`:** Tiempo gastado exclusivamente dentro de la propia función.

Confirmamos que el cuello de botella real no era el bucle en sí, sino las llamadas matemáticas (`sin`/`cos` en `libm.so.6`, como `__cos_fma` / `__sin_avx` y rutinas internas de reducción de argumentos).

**4. Generación del Flame Graph:**
Usamos los scripts de Brendan Gregg sobre el archivo `perf.data` generado:
```bash
git clone https://github.com/brendangregg/FlameGraph.git
perf script | ./FlameGraph/stackcollapse-perf.pl | ./FlameGraph/flamegraph.pl > flamegraph.svg
```
Luego abrimos `flamegraph.svg` en el navegador para inspeccionar visualmente las barras de llamadas.

---

### Ejercicio 3: Benchmarking con `hyperfine`

**Objetivo:** Comparar el rendimiento de dos implementaciones de la misma tarea.

Compilamos una versión sin optimizar (`-O0`) de `slow.c` para compararla contra la versión optimizada (`-O2`) usando 3 ejecuciones previas de calentamiento:
```bash
gcc -O0 slow.c -o slow_O0 -lm
hyperfine --warmup 3 './slow' './slow_O0'
```
`hyperfine` ejecutó múltiples repeticiones, calculó media, desviación estándar ($\sigma$), rango mínimo/máximo y mostró en el resumen cuántas veces más rápida resultó la versión ganadora.

---

### Ejercicio 4: Monitoreo con `htop` y afinidad de CPU con `taskset`

**Objetivo:** Observar el consumo de CPU en tiempo real y entender por qué `taskset --cpu-list 0,2 stress -c 3` no utiliza tres CPUs.

**Ejecución (con dos terminales divididas):**
* Terminal 1: `htop`
* Terminal 2: `taskset --cpu-list 0,2 stress -c 3`

**¿Qué hace cada parte?**
* `taskset --cpu-list 0,2`: Restringe la **afinidad de CPU** del proceso y sus hijos únicamente a los núcleos `0` y `2`.
* `stress -c 3`: Lanza 3 procesos trabajadores intentando saturar el $100\%$ de CPU cada uno.

**¿Por qué `stress` no usa tres CPUs?**
Porque el planificador (*scheduler*) de Linux respeta estrictamente la afinidad impuesta por `taskset` y tiene prohibido asignar los procesos a otros núcleos libres. Por lo tanto, obliga a los 3 trabajadores a turnarse dentro de los núcleos permitidos:
* En una máquina con 4 o más núcleos, los 3 procesos comparten los núcleos `0` y `2` ($\frac{200\%}{3} \approx 66.6\%$ de CPU cada uno).
* En un entorno con solo 2 núcleos (`0` y `1`, como el que vimos en `htop`), el núcleo `2` no existe, por lo que los 3 procesos quedan confinados únicamente al núcleo `0` ($\frac{100\%}{3} \approx 33.3\%$ cada uno).

---

### Ejercicio 5: Descubrir y terminar un proceso que ocupa un puerto

**Objetivo:** Identificar qué proceso escucha en un puerto ocupado y cerrarlo de forma limpia.

**Paso a paso (con dos terminales):**
1. **Levantar un servidor de prueba en el puerto `4444` (Terminal 1):**
   ```bash
   python3 -m http.server 4444
   ```
2. **Localizar el proceso propietario del puerto (Terminal 2):**
   ```bash
   ss -tlnp | grep 4444
   ```
   * `-t`: Conexiones TCP.
   * `-l`: Sockets en estado *Listening*.
   * `-n`: Puertos en formato numérico (sin resolver nombres).
   * `-p`: Muestra el nombre del proceso y su `pid=`.
3. **Terminar el proceso (Terminal 2):**
   ```bash
   kill <PID>
   ```
   Al ejecutar `kill`, el servidor de la Terminal 1 se detiene (`Terminated`) y `ss -tlnp | grep 4444` deja de devolver resultados, confirmando que el puerto quedó libre.