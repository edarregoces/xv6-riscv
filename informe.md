# Exploración de la planificación de procesos en XV6

**Curso:** Sistemas Operativos
**Integrantes:** Evan David Arregoces | Yoskar Alomia Alvarado
**Repositorio (fork):** https://github.com/edarregoces/xv6-riscv
**Fecha de entrega:** 16 de noviembre de 2026

---

## 1. Repositorio y entorno

### 1.1. Fork y commit de partida

**Fork:**

```
origin https://github.com/edarregoces/xv6-riscv.git (fetch)
origin https://github.com/edarregoces/xv6-riscv.git (push)
```

**Commit:**

```
fa2ddcb7ca87a77af579380a6eafd5aff163b44f
koki kitamura
Fri Oct 30 17:37:13 2026 +0900
usertests: bound the zero-length pipe read regression test
```

### 1.2. Versión de Linux

```
Distributor ID: Ubuntu
Description:    Ubuntu 26.04.1 LTS
Release:        26.04
Codename:       resolute
Kernel:         7.0.0-43-generic
```

### 1.3. Herramientas y versiones

**Herramientas:**

1. **riscv64-linux-gnu-gcc**: Sirve para: Compila el C a instrucciones RISC-V
2. **qemu-system-riscv64**: Sirve para: Emula el computador RISC-V donde corre XV6
3. **make**: Sirve para: Ejecuta el Makefile que automatiza la compilación
4. **gdb-multiarch**: Sirve para: Depurar

**Captura de versiones** (transcripción de la imagen original):

```
riscv64-linux-gnu-gcc (Ubuntu 15.2.0-16ubuntu1) 15.2.0
Copyright (C) 2025 Free Software Foundation, Inc.
This is free software; see the source for copying conditions. There is NO
warranty; not even for MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.

GNU ld (GNU Binutils for Ubuntu) 2.46
QEMU emulator version 10.2.1 (Debian 1:10.2.1+ds-1ubuntu3.3)
Copyright (c) 2003-2025 Fabrice Bellard and the QEMU Project developers
GNU gdb (Ubuntu 17.1-2ubuntu1) 17.1
GNU Make 4.4.1
git version 2.53.0
```

### 1.4. Cómo identificamos las herramientas necesarias (fuentes)

**1.4.1.** Empezamos por el README del repositorio oficial (https://github.com/mit-pdos/xv6-riscv), sección "BUILDING AND RUNNING XV6". Ahí vimos que se necesita un toolchain RISC-V y qemu para riscv64-softmmu, pero no indica cómo instalarlos.

**1.4.2.** Buscamos cómo instalarlos y encontramos la página "Tools Used in 6.1810" del curso del MIT (https://pdos.csail.mit.edu/6.S081/2024/tools.html). Allí aparece el comando para Debian/Ubuntu, que usamos tal cual:

```
sudo apt-get install git build-essential gdb-multiarch qemu-system-misc \
  gcc-riscv64-linux-gnu binutils-riscv64-linux-gnu
```

**1.4.3.** La misma página pide QEMU 7.2 o superior y recomienda Ubuntu 24 o posterior. Verificamos nuestras versiones con: `lsb_release -a`, `qemu-system-riscv64 --version`, `riscv64-linux-gnu-gcc --version`

**1.4.4. Fuente:** https://pdos.csail.mit.edu/6.S081/2024/tools.html

---

## 2. Compilación y ejecución de XV6

### 2.1. Compilación y evidencia de XV6 operativo (ls, cat)

**Compilación:**

```
xv6 kernel is booting

hart 1 starting
hart 2 starting
init: starting sh
```

**Evidencia:**

> [Imagen: captura de la compilación. Muestra `balloc: first 990 blocks have been allocated`, `balloc: write bitmap block at sector 46`, el comando `qemu-system-riscv64 -machine virt -bios none -kernel kernel/kernel -m 128M -smp 3 -nographic ...` y el arranque de xv6 con `init: starting sh`.]

**Salida de `ls`** (transcripción de la captura):

```
$ ls
.           1 1 1024
..          1 1 1024
README      2 2 2441
cat         2 3 36760
echo        2 4 35624
forktest    2 5 18168
grep        2 6 44216
init        2 7 36008
kill        2 8 35544
ln          2 9 35336
ls          2 10 42912
mkdir       2 11 35600
rm          2 12 35584
sh          2 13 58312
stressfs    2 14 36448
usertests   2 15 213544
grind       2 16 51856
wc          2 17 37760
zombie      2 18 34880
logstress   2 19 37592
forphan     2 20 36384
dorphan     2 21 35832
sync        2 22 34976
console     3 23 0
```

**Salida de `cat`:**

```
$ cat
hola
```

### 2.2. Ejecución con una sola CPU virtual

Consultamos el "Makefile" del repositorio con:

```
grep -n "CPUS" Makefile
```

Resultado:

```
174:ifndef CPUS
175:CPUS := 3
178:QEMUOPTS = -machine virt -bios none -kernel $K/kernel -m 128M -smp $(CPUS) -nographic
```

El Makefile define un valor por defecto para CPUS y lo pasa a qemu mediante la opción `-smp`, que es la opción de qemu que fija el número de procesadores virtuales. Como el valor se puede sobrescribir desde la línea de comandos, no fue necesario modificar ningún archivo del repositorio.

### 2.3. Comando o ajuste utilizado

```
make qemu CPUS=1
```

Con esto se ejecuta XV6 con una sola CPU virtual, sin editar el Makefile.

### 2.4. Evidencia

**Salida:**

```
qemu-system-riscv64 -machine virt -bios none -kernel kernel/kernel -m 128M -smp 1 -nographic -global virtio-mmio.force-legacy=false -drive file=fs.img,if=none,format=raw,id=x0 -device virtio-blk-device,drive=x0,bus=virtio-mmio-bus.0
```

> [Imagen: captura de la ejecución con `-smp 1`, mostrando `xv6 kernel is booting` e `init: starting sh`.]

---

## 3. Planificador original

### 3.1. Algoritmo por defecto y evidencia en el comportamiento

Para evidenciar empíricamente se creó dentro de la carpeta users un script llamado testSO.c, dicho script crea un proceso hijo mediante un fork() como visto en las sesiones anteriores, que imprime su PID, en un bucle donde se evidencia el cambio de prioridad

> [Imagen: explorador de archivos con la búsqueda "testSO" en `xv6-riscv/user`, mostrando testSO, testSO.asm, testSO.c, testSO.d, testSO.o y testSO.sym.]

**Código de testSO.c** (transcripción de la captura del editor nano):

```c
#include "kernel/types.h"
#include "kernel/stat.h"
#include "user/user.h"

int main(int argc, char *argv[]) {
    int pid = fork();

    if (pid < 0) {
        printf("sched_test: error en fork\n");
        exit(1);
    } else if (pid == 0) {
        // Proceso Hijo
        while(1) {
            printf("Proceso HIJO (PID: %d) ejecutandose...\n", getpid());
            for(int i = 0; i < 500000; i++); // Retraso artificial para notar la alternancia
        }
    } else {
        // Proceso Padre
        while(1) {
            printf("Proceso PADRE (PID: %d) ejecutandose...\n", getpid());
            for(int i = 0; i < 500000; i++); // Retraso artificial
        }
    }
    exit(0);
}
```

Este script se agregó manualmente a Makefile, posteriormente se procedería a correr el sistema XV6 restringida a una sola CPU virtual con los comandos `make clean, make qemu CPUS=1`, dentro del sistema operativo se inicia el script y se evidencia la siguiente salida

**Salida del script** (transcripción; las líneas repetidas se resumen entre corchetes):

```
Proceso PADRE (PID: 3) ejecutandose...
Proceso PADRE (PID: 3) ejecutandose...
Proceso PADRE (PIProceso HIJO (PID: 4) ejecutandose...
Proceso HIJO (PID: 4) ejecutandose...
[... línea "Proceso HIJO (PID: 4)" repetida unas 20 veces ...]
Proceso HIJO D: 3) ejecutandose...
Proceso PADRE (PID: 3) ejecutandose...
[... línea "Proceso PADRE (PID: 3)" repetida unas 25 veces ...]
Proceso PADRE (PID(PID: 4) ejecutandose...
Proceso HIJO (PID: 4) ejecutandose...
Proceso HIJO (PID: 4) ejecutandose...
```

Aquí queda en evidencia el algoritmo de planificación Round Robin, ya que los procesos PADRE e HIJO se intercalan equitativamente en la única CPU virtual disponible que les asigna bloques de tiempo iguales para su funcionamiento.

### 3.2. Archivo y función donde se selecciona el siguiente proceso

Navegando en la carpeta del sistema operativo educativo XV6, podemos ver la carpeta kernel, el corazón del SO, como dicho en la sección 1.1 del libro de XV6: 'The kernel associates a process identifier, or PID, with each process', aunque la sección del libro no habla o menciona un archivo o función de planificación en CPU , se puede encontrar en la ruta '/home/edajojo/xv6-riscv/kernel/proc.c' el script proc.c o process, donde hay una funcion 'scheduler' entre otras funciones que asignan los tiempos y se define el orden de procesos según lo descrito en la sección 1.1 del libro

**Función `scheduler()`** (transcripción de la captura de `kernel/proc.c`):

```c
void
scheduler(void)
{
  struct proc *p;
  struct cpu *c = mycpu();

  c->proc = 0;
  for(;;){
    // The most recent process to run may have had interrupts
    // turned off; enable them to avoid a deadlock if all
    // processes are waiting. Then turn them back off
    // to avoid a possible race between an interrupt
    // and wfi.
    intr_on();
    intr_off();

    int found = 0;
    for(p = proc; p < &proc[NPROC]; p++) {
      acquire(&p->lock);
      if(p->state == RUNNABLE) {
        // Switch to chosen process.  It is the process's job
        // to release its lock and then reacquire it
        // before jumping back to us.
        p->state = RUNNING;
        c->proc = p;
        swtch(&c->context, &p->context);

        // Don't re-enable interrupts on release.
        mycpu()->intena = 0;

        // Process is done running for now.
        // It should have changed its p->state before coming back.
        c->proc = 0;
        found = 1;
      }
      release(&p->lock);
    }
    if(found == 0) {
      // nothing to run; stop running on this core until an interrupt.
      asm volatile("wfi");
    }
  }
}
```

### 3.3. Relación con los estados de los procesos y los eventos (ceder/recuperar la CPU)

En el capítulo 2.5 del libro 'process overview', se describe como en XV6, los procesos son aislados en su propio kernel, CPU y memoria para mantener la integridad del propio proceso y de otros procesos dentro de XV6, esto lo hace mediante los espacios de direcciones y la asignación de tiempo a los hilos de los procesos.

Al revisar el capítulo 8 del libro 'scheduling', donde se describe como XV6 maneja procesos mediante multiplexing (hacer creer que todos tienen su propia CPU virtual), qué hace mediante switches voluntarios e involuntarios con calls (llamadas) read, wait, interrupt.

El hilo kernel suelta la CPU al cambiarlo contextualmente (mediante yield) con el hilo schedules de la misma CPU, este hilo scheduler entonces elige otro proceso del hilo kernel para correr/asignarle CPU.

### 3.4. Cómo se incorpora al sistema compilado (Makefile y construcción)

En la sección 2.6 del libro 'Code: starting xv6, the first process and system call' se describe la construcción e inicialización de XV6, aquí podemos ver que al iniciarse, el cargador de inicio o boot loader copia el kernel a la dirección de memoria física 0x80000000, aclarando que se inicia aquí debido a que las direcciones inferiores están reservadas para dispositivos de entrada/salida, posteriormente el loader salta a _entry, y RISC-V inicia con la paginación de hardware deshabilitada, o sea que las direcciones de memoria virtuales están directamente mapeadas a direcciones físicas de la memoria, las instrucciones en _entry configuran y habilitan (en stack0) el uso de C en XV6 al llamar el script start.c.

'start.c' también realiza unas configuraciones que solo se permiten en modo máquina, como programar el reloj para generar interrupciones medidas(como descrito arriba), posteriormente start usa la instrucción mret de RISC-V para cambiar y otorgar permisos de supervisor antes de pasar con main, que termina de inicializar varios subsistemas y crea el primer proceso llamado 'userinit', todos los nuevos procesos se empiezan a ejecutar en el kernel en forkret, después de ejecutar la llamada kexec, forkret retorna al espacio de usuario, donde finalmente se inicializa la consola de XV6.

---

## 4. MLFQ y su incorporación a XV6

### 4.1. Descripción inicial de MLFQ y dudas (antes de consultar a la IA)

El MLFQ (multi-level feedback queue) es un algoritmo basado en el uso de varias colas con distinta prioridad. El planificador (scheduling) siempre ejecuta los procesos que estén en la cola con mayor prioridad, si hay varios en la misma cola, se utiliza el algoritmo "Round Robin" se turnan en un quantum de misma duración. Un proceso nuevo entra en la cola de máxima prioridad ya que no conoce su tiempo de duración y dependiendo si gasta todo su tiempo en cpu o no será degradado o no y a su vez el sistema lleva un contador de cuantos ciclos ha estado un proceso en una cola, si pasa el máximo permitido será degradado también, estas son unas de las reglas de este algoritmo

**Dudas:**

1. Si un proceso es interrumpido a la mitad de su quantum debido a que llego otro de mayor prioridad, al volver de nuevo este seguirá con el quantum que llevaba o iniciará uno desde cero?
2. Qué le pasa a un proceso en la última cola cuando agota su quantum?
3. Cuántas colas conviene usar, que tan largo debe ser el quantum en cada una, y cada cuanto se debe hacer una subida de prioridad.

### 4.2. Prompt 1 (con nuestra descripción y síntesis)

Actúa como tutor socrático de una pareja de estudiantes de cuarto semestre de Ingeniería
de Sistemas. Conocemos C++ y hemos comenzado a explorar XV6 para RISC-V.
Queremos identificar y justificar qué se necesitaría para incorporar MLFQ a XV6. En esta
entrega no vamos a implementarlo.
Nuestra descripción inicial del algoritmo, escrita antes de esta conversación:

En la sección 4.1 describimos MLFQ antes de consultar a la IA y planteamos tres dudas.

Descripción: MLFQ usa varias colas con distinta prioridad y siempre ejecuta el proceso de la cola más alta. Dentro de una misma cola los procesos se turnan con Round Robin. Un proceso nuevo entra en la cola de máxima prioridad, se degrada si agota su quantum y sube o se mantiene según cuántos ciclos lleva en cada cola.

Dudas:

Si un proceso es interrumpido a mitad de su quantum por la llegada de otro de mayor prioridad, ¿retoma el quantum que le quedaba o empieza uno nuevo?
¿Qué pasa con un proceso en la última cola cuando agota su quantum?
¿Cuántas colas conviene usar, qué duración debe tener el quantum en cada una y cada cuánto se debe subir la prioridad?

Lo que averiguamos del planificador original y de su incorporación al sistema compilado:

3.1 Algoritmo: Creamos user/testSO.c, que hace fork() y, en un bucle, imprime el PID de padre e hijo. Con make qemu CPUS=1 (valor definido en el Makefile y sobrescribible desde la línea de comandos) vimos que ambos procesos se alternan en bloques de tiempo iguales, por lo que concluimos que el algoritmo es Round Robin.

3.2 Selección del proceso: Ubicamos la lógica en scheduler(), en kernel/proc.c. Esta función recorre proc[], toma el lock de cada proceso y, si encuentra uno en estado RUNNABLE, lo marca como RUNNING y cambia de contexto con swtch. Si no hay ninguno listo, ejecuta wfi hasta la siguiente interrupción.

3.3 Estados y eventos: Cada proceso tiene su propio espacio de direcciones, y la CPU se multiplexa entre ellos (capítulos 2.5 y 8 del libro). Un proceso cede la CPU voluntariamente al bloquearse, como en read o wait, que duermen con sleep(), o involuntariamente por la interrupción de reloj, que llama a yield(). En ambos casos el control vuelve a scheduler(), que elige el siguiente proceso.

3.4 Arranque: Revisamos en kernel/start.c cómo el cargador copia el kernel en 0x80000000 y salta a _entry. Desde start(), en modo máquina, se programa el reloj y se usa mret para pasar a modo supervisor. Luego main() inicializa los subsistemas y crea el primer proceso, userinit. Los procesos nuevos comienzan en forkret, que tras exec regresa al espacio de usuario.

Corrijo un punto de mi síntesis anterior: read y wait ceden la CPU con sleep(), no con yield(). Este último solo se usa en la interrupción de reloj.

Reglas de interacción:
1. Formula una sola pregunta principal por turno y espera nuestra respuesta. No presentes
una lista completa de cambios ni una arquitectura resuelta.
2. Empieza examinando nuestra descripción de MLFQ. Ayúdanos a detectar reglas ambiguas,
ausentes o contradictorias. No sustituyas nuestra descripción por una versión terminada
ni elijas silenciosamente una variante del algoritmo.
3. Pídenos que propongamos y justifiquemos las decisiones que faltan. Después, ayúdanos a
comprobarlas con ejemplos pequeños de procesos y decisiones de planificación que nosotros
debamos resolver.
4. Guíanos para relacionar cada regla con XV6. Antes de sugerir una modificación,
pregúntanos qué comportamiento necesitamos y qué evidencia del código respalda nuestra
propuesta.
5. No supongas que tienes acceso a nuestro repositorio. Pide los fragmentos necesarios y
distingue lo observado de lo que todavía sea una hipótesis. No inventes archivos,
funciones o comportamientos de nuestra versión.
6. No entregues código de implementación, parches, una lista cerrada de archivos por
modificar ni pseudocódigo que resuelva el planificador. Revisa nuestras propuestas y
señala problemas mediante preguntas y pistas graduales. Puedes explicar conceptos previos
con ejemplos mínimos.
7. Si nos bloqueamos, reduce la dificultad de la pregunta y ofrece una pista concreta. Si
pedimos la solución directa, conserva el modo socrático.
El objetivo es que nosotros logremos:
A. Redactar una descripción revisada y coherente de la variante de MLFQ considerada.
B. Explicar qué cambios necesitaría XV6 para soportarla, indicando para cada uno su
propósito, su relación con las reglas y la evidencia del código que lo justifica.
C. Proponer escenarios para comprobar una futura implementación del algoritmo MLFQ en el
sistema operativo XV6. Pídenos que expliquemos qué procesos participarían, qué harían,
qué regla comprobaríamos y qué comportamiento esperaríamos observar. No nos entregues los
escenarios resueltos. En esta tarea solo los describiremos; no vamos a programarlos ni
ejecutarlos.
Mantén el análisis dentro del alcance de una propuesta: no necesitamos implementar MLFQ ni construir un selector de planificadores. Ayúdanos a registrar las cuestiones que
queden pendientes de investigación.
Al finalizar, pídenos que redactemos nuestras conclusiones. Comprueba su consistencia
mediante una última pregunta aplicada a un caso concreto, sin escribir por nosotros el
reporte.
Comienza con una pregunta que permita comprobar cómo entendemos una regla de nuestra
descripción inicial de MLFQ.

### 4.3. Descripción revisada de la variante de MLFQ

Duda 1: la recomendación habitual (OSTEP, capítulo 8) es no reiniciar el contador. Se acumula el tiempo usado en cada nivel, para que un proceso no pueda evitar la degradación cediendo la CPU justo antes de agotar su quantum.
Duda 2: lo común es que se quede en la última cola y vuelva al final de su turno Round Robin. El boost periódico evita que quede sin CPU.
Duda 3: den valores iniciales y digan que son provisionales, por ejemplo 3 colas, quantum de 1, 2 y 4 ticks, y boost cada 50 ticks.

### 4.4. Propuesta de cambios en XV6

**Cambio 1:** Cambio 1 (datos): en kernel/proc.h, agreguen a struct proc un campo para el nivel de cola y otro para los ticks acumulados en ese nivel. Inicialícenlos en allocproc() de kernel/proc.c. Como fork() usa allocproc(), el hijo también entra en el nivel más alto.

**Cambio 2:** Cambio 2 (planificación y tiempo): en scheduler(), elijan el proceso RUNNABLE de menor nivel en vez del primero que encuentren. Como el for actual recorre en orden, necesitan una primera pasada para elegir el candidato y después tomar su p->lock y verificar que siga RUNNABLE. En el manejo del reloj de kernel/trap.c (donde usertrap() llama a yield() cuando which_dev == 2), incrementen el contador del proceso en ejecución, degrádenlo si superó el quantum de su nivel y reubiquen a todos en el nivel 0 cada S ticks. Respeten los locks al tocar campos de otros procesos.

### 4.5. Decisiones que requieren investigación adicional

Número de colas y quantum por nivel: se decide con experimentos. Pueden medir el tiempo de respuesta de un proceso interactivo con distintas configuraciones.
Periodo de boost: hay un intercambio entre inanición (si es muy largo) y respuesta a procesos interactivos (si es muy corto).
Procesos que se duermen antes de agotar el quantum: en la regla clásica conservan su nivel. Decidan qué hacer y justifíquenlo.
Cómo medir: uptime() devuelve los ticks desde el arranque y sleep(n) duerme n ticks. Pueden contar salidas de testSO por ventana de tiempo.
Fuente: OSTEP, capítulo 8, “Scheduling: The Multi-Level Feedback Queue” (Arpaci-Dusseau), gratuito en ostep.org. Cítenlo en el informe.

### 4.6. Conclusiones nuestras

El planificador original es Round Robin. La prueba con dos procesos (PADRE e HIJO) en una sola CPU virtual muestra que se turnan bloques de ejecución. La función scheduler() en kernel/proc.c busca el siguiente proceso RUNNABLE a partir de donde quedó la búsqueda, y como el ciclo continúa después de cada cambio de contexto, el turno pasa al siguiente proceso sin código adicional.
El cambio de proceso depende del tiempo y de las cesiones voluntarias. La interrupción de reloj llama a yield(), y un proceso también suelta la CPU al dormirse (sleep) o al terminar (exit). Las líneas de la salida con texto mezclado (“PIProceso HIJO”, “HIJO D: 3)”) muestran que el cambio puede ocurrir en medio de un printf, lo que indica que la asignación se hace por tiempo y no por línea de salida.
El MLFQ tocaría tres partes del kernel. La propuesta modificaría struct proc en kernel/proc.h, la inicialización en allocproc() y la selección en scheduler(), además del manejo de la interrupción de reloj en kernel/trap.c. El cambio más delicado es el de scheduler(), porque debe respetar el lock de cada proceso al elegirlo.
Lo que esperamos, sin medirlo aún. Con MLFQ esperamos que los procesos interactivos mantengan prioridad alta y respondan más rápido, mientras los de cómputo bajan de nivel y pierden prioridad frente a ellos. Este comportamiento está descrito en la literatura (OSTEP, capítulo 8), pero no lo verificamos. Queda pendiente para los escenarios de la sección 4.8.

### 4.7. Retroalimentación de la IA

Revisé el informe completo. La base es sólida: la estructura sigue el orden de la tarea, las fuentes están citadas, el análisis del Makefile con grep está bien justificado y son honestos al decir qué aún no han medido. Como la entrega es hoy, les sugiero atender primero lo que más puede costarles.

Errores que conviene corregir primero

El scheduler no “retoma donde quedó” (sección 4.6). La conclusión dice que scheduler() busca el siguiente RUNNABLE “a partir de donde quedó la búsqueda”, pero el código que ustedes mismos pegaron en 3.2 dice otra cosa. El for(;;) externo vuelve a empezar en p = proc en cada pasada, y no hay ninguna variable que guarde la posición anterior. La alternancia se produce porque, después de cada swtch, el for interno continúa con el siguiente índice. Revisen esa frase y expliquen con sus palabras por qué A y B se turnan aunque la búsqueda siempre empiece en el mismo lugar. Esto importa porque es justo la parte que van a modificar en 4.4.

Terminología que contradice la conclusión. En 3.1 el script “evidencia el cambio de prioridad”, pero Round Robin no tiene prioridades, solo alternancia por tiempo. En 5.2.2 aparece “sube de privilegios”, que debería decir “sube de prioridad”.

Imprecisiones en la sección 3. En 3.3 se dice que los procesos están aislados “en su propio kernel, CPU y memoria”. En xv6 la CPU se multiplexa entre procesos; no es de cada uno. El texto también mezcla “hilo” y “proceso” (cada proceso tiene un hilo de kernel, y el scheduler es un hilo por CPU), y menciona “interrupt” como si fuera una llamada al sistema, cuando es un evento de hardware. En 3.4, la frase “después de ejecutar la llamada kexec, forkret retorna al espacio de usuario” está mal ordenada: en el código la función es exec(), y forkret es la primera función que ejecuta un proceso nuevo en el kernel, desde donde vuelve a modo usuario por usertrapret(). La consola no la inicializa forkret: main() llama a consoleinit() y el proceso init la abre como descriptores 0, 1 y 2. Además, con -bios none es QEMU quien carga el kernel, no un bootloader. Por último, el título de 3.4 promete explicar el Makefile, pero la sección solo trata el arranque. Cambien el título o expliquen el papel del Makefile (lista de objetos y enlazado con kernel.ld).

Evidencias poco precisas. En 3.1 los conteos “unas 20 veces” y “unas 25 veces” son resúmenes manuales, así que no sirven para afirmar “bloques de tiempo iguales”. Presenten la observación como cualitativa o midan. La ruta /home/edajojo/xv6-riscv/kernel/proc.c es local y de otra cuenta; usen kernel/proc.c. En 3.1 dicen que el script está “dentro de la carpeta users” y luego aparece como user/, así que unifiquen.

Secciones incompletas o con problemas de estructura

La 4.7 está vacía. Es el lugar natural para documentar la retroalimentación de la IA: qué preguntas les hizo el tutor, qué respuestas aceptaron, cuáles corrigieron (como el punto sobre read y wait) y cuáles verificaron en OSTEP o en el código.

La sección 5 tiene poca evidencia. Solo aparece el nombre del modelo y las reflexiones. Falta describir qué parte de la conversación usó cada uno. Hay además una contradicción que conviene aclarar: Evan dice que dejó el uso de la IA a su compañero, pero 4.2 y 4.3 están en el informe compartido. Digan quién escribió qué. El texto de 4.3 también tiene tono de instrucción (“den valores iniciales y digan que son provisionales”), que no es el estilo de respuesta socrática que pidieron. Si viene de la IA, atribúyanlo; si es suyo, reescríbanlo como decisión propia y justifíquenla.

4.3 aún no es una descripción revisada. Los objetivos piden una descripción coherente de la variante, pero lo que hay son respuestas a tres dudas. Faltan en un solo lugar las reglas completas: colas, quantum por nivel, cuándo se degrada, cómo funciona el boost y qué pasa cuando un proceso se duerme (esto último queda pendiente en 4.5).

4.4 no cumple del todo el objetivo B. Para cada cambio hay que indicar propósito, regla que implementa y evidencia del código, y los cambios 1 y 2 no citan funciones o líneas concretas del comportamiento actual. El encabezado del Cambio 1 está duplicado (“Cambio 1: Cambio 1”). El Cambio 2 mezcla tres responsabilidades (contar ticks, degradar y hacer boost), y conviene separarlas. Además, 4.6 dice que el cambio toca tres partes del kernel, mientras que 4.4 describe dos cambios.

Hay también preguntas que el informe no responde desde el código y que son parte de la Duda 1 y del diseño:

Cuando despierta un proceso de mayor prioridad, ¿xv6 lo ejecuta de inmediato? Revisen qué hace wakeup() y dónde se llama yield().
Si el boost depende de ticks, revisen dónde se incrementa ese contador. Está en clockintr(), que solo corre en el CPU 0, así que debe avanzar una vez por tick y no una vez por CPU.
Si el reloj interrumpe código del kernel, el yield() no está en usertrap() sino en kerneltrap(). El cambio en trap.c debe contemplar ambos caminos.
La duración del tick define el tiempo real de los quantum. El comentario de start.c dice que el intervalo del temporizador es de alrededor de una décima de segundo en QEMU. Verifíquenlo en su copia, porque lo necesitan para los escenarios.
Escenarios (4.8)

El formato dado/cuando/entonces es claro, pero cada escenario debe nombrar la regla que comprueba; el primero habla de “cola alta” sin decir qué regla lo garantiza. En el escenario 2, “el de cómputo no recibe CPU” es demasiado absoluto: los interactivos duermen y dejan huecos, así que lo esperable es que reciba muy poca CPU y que mejore con el boost. Escríbanlo como predicción medible. Falta además un escenario para la regla anti-trampa de la Duda 1 (un proceso que se duerme justo antes de agotar su quantum no debe escapar de la degradación), que es una de las partes más interesantes de la variante. Y para comparar con Round Robin necesitan la misma carga corriendo con el planificador original; conviene que lo expliciten.

Formato, conversión y referencias

Quedan artefactos de la conversión desde PDF: “bajasegúnel contadordevecesquesemantenganen lamisma” y “cpù” en 5.2.2, “medidas(como” en 3.4, y “Cambio 1: Cambio 1” en 4.4. Revisen el texto completo. El encabezado de 2.1 dice “Compilación” pero muestra la salida de arranque, mientras que la descripción de la imagen menciona la salida de make, que no aparece transcrita.

Las imágenes están como placeholders. Si la entrega requiere evidencias visuales, inclúyanlas como archivos junto al .md. Falta también una bibliografía: citen OSTEP (capítulo 8, Arpaci-Dusseau, ostep.org), el libro de xv6 de Cox, Kaashoek y Morris con su versión, y la página de herramientas del MIT.

### 4.8. Escenarios de comprobación futura

**Escenario 1:** dado un proceso de cómputo intensivo (como el HIJO de testSO) y un proceso interactivo que duerme con sleep() periódicamente, cuando ambos corren con CPUS=1, entonces el interactivo debe mantenerse en la cola alta y responder más rápido que con Round Robin. Se mide con uptime() antes y después de cada operación.
**Escenario 2:** dado un proceso de cómputo en un nivel bajo y varios interactivos que se turnan en la cola alta, cuando pasan S ticks sin boost, entonces el de cómputo no recibe CPU. Con boost cada S ticks, debe avanzar. Se mide contando sus salidas por ventana de tiempo.

---

## 5. Evidencia del aprendizaje con IA

### 5.1. Servicio y modelo utilizado

Claude Haiku 5.5

### 5.2. Reflexión individual

#### 5.2.1. Evan Arregoces

Explorar el sistema operativo de aprendizaje fue una experiencia interesante y enriquecedora, el proceso de ensayo y error, presenciar los procesos de una manera más abstracta y luego complementar conocimientos con el libro proporcionado fue lo que más disfrute, no soy un fanatico de la IA, debido a mis propias creencias morales y a que considero que es una salida fácil que no siempre prefiero tomar, por eso mismo evitó su uso así que en esta entrega me limite a dejarle esa parte a mi compañero.

#### 5.2.2. Yoskar Alomia

Al revisar el sistema operativo de XV6 entendí que algoritmo, como se re rige y de qué depende la decisión del uso de la cpu, el scheduler del kernel y que el proceso cede la cp por interrupción (yield), al dormirse (sleep) o al terminar su trabajo (exit), esto se pudo visualizar mejor al usar una sola cpù virtual, lo que permitió analizar el algoritmo de Round Robin, pieza fundamental para este algoritmo, como se distribuye cada proceso y como permite que este sea eficiente en salida y en respuesta, como los sube de privilegios cada periodo o los bajasegúnel contadordevecesquesemantenganen lamisma prioridad.

#### 6. Referencias

Arpaci-Dusseau, R. H., y Arpaci-Dusseau, A. C. (2018). Operating systems: Three easy pieces. Arpaci-Dusseau Books. https://ostep.org (capítulo 8, “Scheduling: The Multi-Level Feedback Queue”)

Cox, R., Kaashoek, F., y Morris, R. (s. f.). xv6: A simple, Unix-like teaching operating system [Libro]. MIT. [Completen año y revisión de la copia que usaron.]

MIT PDOS. (2024). Tools used in 6.1810. https://pdos.csail.mit.edu/6.S081/2024/tools.html

mit-pdos. (s. f.). xv6-riscv [Repositorio de código]. GitHub. https://github.com/mit-pdos/xv6-riscv
