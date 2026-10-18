---
title: "Bandit: nivel 5 → 6"
date: 2026-10-18 17:00:00 +0200
categories: [Linux, Bandit]
tags: [bandit, overthewire, linux, find, tr]
description: Veinte directorios y un fichero escondido que cumple tres condiciones. Combinamos filtros de find por tamaño y permisos, y limpiamos la salida con tr.
ref: bandit-5-6
serie: bandit
nivel: 6
---

En el nivel anterior buscábamos entre diez ficheros. Ahora el fichero está escondido en uno de **veinte directorios**, y el enunciado nos da su "retrato robot": tres características que tiene que cumplir. Es el momento de sacarle partido a `find` combinando varios filtros a la vez.

> Las contraseñas aparecen como `[REDACTED]`, siguiendo las [normas de OverTheWire](https://overthewire.org/rules/).
{: .prompt-warning }

## Conexión al nivel 5

```console
$ sshpass -p '[REDACTED]' ssh bandit5@bandit.labs.overthewire.org -p 2220
bandit5@bandit:~$ export TERM=xterm
```

## El enunciado

La [página del nivel 6](https://overthewire.org/wargames/bandit/bandit6.html) dice que la contraseña está en un fichero dentro de `inhere` que cumple tres cosas:

- es ***human-readable***,
- ocupa **1033 bytes**,
- **no es ejecutable**.

```console
bandit5@bandit:~$ ls
inhere
bandit5@bandit:~$ cd inhere
bandit5@bandit:~/inhere$ ls
maybehere00  maybehere01  maybehere02  ...  maybehere19
```

Veinte directorios, cada uno con varios ficheros dentro. Revisarlos a mano sería eterno, pero tenemos el retrato robot, y `find` sabe buscar por características.

## La solución: `find` con varios filtros

```console
bandit5@bandit:~/inhere$ find . -type f -size 1033c ! -executable
./maybehere07/.file2
```

Un único resultado. Por partes:

- **`find .`**: busca desde el directorio actual, **recorriendo todos los subdirectorios**. No tenemos que entrar en los veinte uno a uno.
- **`-type f`**: solo ficheros.
- **`-size 1033c`**: que ocupen **exactamente 1033 bytes**. La **`c`** es importante y la explico abajo.
- **`! -executable`**: el `!` **niega** la condición que viene detrás, así que se queda con los ficheros que **no** son ejecutables. También se puede escribir `-not -executable`, que se lee mejor.

Cuando pones varios filtros seguidos, `find` los combina con un **Y lógico**: el fichero tiene que cumplir **todos** a la vez. Por eso, con tres condiciones, de entre docenas de ficheros solo queda uno.

> **La trampa de `-size`:** sin sufijo, `find` no mide en bytes sino en **bloques de 512 bytes**. `-size 1033` buscaría ficheros de unos 528 KB. Para bytes hay que usar **`c`**. Otros sufijos útiles: `k` (KiB), `M` (MiB), `G` (GiB). Y con `+` o `-` delante buscas "más de" o "menos de": `-size +1M` encuentra los ficheros de más de un mega.
{: .prompt-danger }

Fíjate también en el resultado: **`.file2` empieza por punto**, así que es un fichero oculto, como en el [nivel 3 → 4](/posts/bandit-nivel-3-4/). `ls` lo habría escondido, pero **`find` muestra los ocultos por defecto**. Otra razón para usar `find` cuando buscas algo de verdad.

### Ruta relativa, no absoluta

`./maybehere07/.file2` es una **ruta relativa**: empieza por `./`, que significa "desde el directorio en el que estoy". `find` construye las rutas a partir del punto de partida que le das. Como le dijimos `.`, todas empiezan por `./`. Si hubiéramos lanzado `find /home/bandit5/inhere ...`, nos habría devuelto rutas **absolutas**, que empiezan por `/` y funcionan desde cualquier sitio.

## Leer el fichero y limpiar la salida

```console
bandit5@bandit:~/inhere$ cat ./maybehere07/.file2 | tr -d ' '
[REDACTED]
```

Al abrir el fichero con `cat` a secas, la contraseña aparece seguida de un **enorme espacio en blanco**. Tiene sentido: la contraseña ocupa muy poco y el fichero tiene que medir 1033 bytes, así que está **relleno de espacios** hasta llegar a ese tamaño.

**`tr -d ' '`** lo limpia. `tr` (*translate*) transforma caracteres, y con **`-d`** (*delete*) **borra** todos los caracteres que le indiques, en este caso el espacio. La contraseña no tiene espacios, así que no se toca, y desaparece todo el relleno.

## Otras formas de resolverlo

**Añadir la condición de *human-readable*.** Mi `find` funcionó porque solo un fichero cumplía tamaño y permisos, pero no comprueba la tercera condición. Podemos hacerlo con `file`, como en el [nivel 4 → 5](/posts/bandit-nivel-4-5/):

```console
bandit5@bandit:~/inhere$ find . -type f -size 1033c ! -executable -exec file {} + | grep text
```

**Buscar y leer en un solo paso.** No hace falta copiar la ruta: `find` puede pasarle el resultado directamente a `cat`:

```console
bandit5@bandit:~/inhere$ find . -type f -size 1033c ! -executable -exec cat {} + | tr -d ' '
```

**Otras formas de quitar el relleno:**

```console
bandit5@bandit:~/inhere$ cat ./maybehere07/.file2 | sed 's/ //g'
bandit5@bandit:~/inhere$ cat ./maybehere07/.file2 | awk '{print $1}'
```

- **`sed 's/ //g'`**: sustituye (`s`) cada espacio por nada, en toda la línea (`g`, *global*). Hace lo mismo que `tr -d ' '`, pero `sed` sirve para sustituciones mucho más complejas.
- **`awk '{print $1}'`**: imprime solo el primer campo de cada línea. `awk` separa por espacios, así que se queda con la contraseña y descarta el resto. Es el hermano del `$NF` que usamos en el [nivel 0 → 1](/posts/bandit-nivel-0-1/).

> `-executable` y `-readable` comprueban los permisos **del usuario que ejecuta `find`**, no los del dueño del fichero. Si quieres filtrar por los bits de permiso exactos, por ejemplo "ejecutable por cualquiera", está la opción `-perm`. Nos va a hacer falta más adelante.
{: .prompt-tip }

## Pasar al nivel 6

```console
bandit5@bandit:~/inhere$ exit
$ sshpass -p '[REDACTED]' ssh bandit6@bandit.labs.overthewire.org -p 2220
```

## Resumen

| Comando / concepto | Para qué sirve |
|---|---|
| `find . -type f` | Buscar ficheros recorriendo todos los subdirectorios |
| `-size 1033c` | Tamaño exacto en bytes (`c`); sin sufijo son bloques de 512 bytes |
| `-size +1M` / `-size -10k` | Más de / menos de un tamaño |
| `! -executable` / `-not -executable` | Negar una condición |
| Varios filtros seguidos | Se combinan con Y: hay que cumplirlos todos |
| `find ... -exec cat {} +` | Leer directamente lo que se encuentra |
| `tr -d ' '` | Borrar todos los espacios de la salida |
| `sed 's/ //g'` | Sustituir texto; aquí, quitar los espacios |

{% include series-nav.html %}
