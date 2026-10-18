---
title: "Bandit: nivel 4 → 5"
date: 2026-10-18 16:00:00 +0200
categories: [Linux, Bandit]
tags: [bandit, overthewire, linux, find, file, xargs]
description: Diez ficheros y solo uno contiene texto legible. Usamos find, file y xargs para encontrarlo sin abrirlos uno a uno.
ref: bandit-4-5
serie: bandit
nivel: 5
---

Aquí empieza lo interesante: ya no basta con abrir un fichero, hay que **encontrar** el bueno entre varios. Podríamos abrirlos uno a uno, pero la gracia es que lo haga la terminal por nosotros. Para eso combinamos tres herramientas que vas a usar muchísimo: `find`, `file` y `xargs`.

> Las contraseñas aparecen como `[REDACTED]`, siguiendo las [normas de OverTheWire](https://overthewire.org/rules/).
{: .prompt-warning }

## Conexión al nivel 4

```console
$ sshpass -p '[REDACTED]' ssh bandit4@bandit.labs.overthewire.org -p 2220
bandit4@bandit:~$ export TERM=xterm
```

## El enunciado

La [página del nivel 5](https://overthewire.org/wargames/bandit/bandit5.html) dice que la contraseña está en **el único fichero *human-readable*** del directorio `inhere`:

```console
bandit4@bandit:~$ ls
inhere
bandit4@bandit:~$ cd inhere
bandit4@bandit:~/inhere$ ls
-file00  -file01  -file02  -file03  -file04  -file05  -file06  -file07  -file08  -file09
```

Diez ficheros, y todos empiezan por guion, así que ya sabemos desde el [nivel 1 → 2](/posts/bandit-nivel-1-2/) que habrá que tener cuidado con ellos.

> **Ojo con *human-readable*.** No significa "con permisos de lectura", sino "que contiene **texto que una persona puede leer**". Los otros nueve ficheros tienen datos binarios: si los abres con `cat` verás caracteres raros. Entender bien el enunciado es la mitad del nivel.
{: .prompt-info }

## La pregunta: ¿cómo lo encuentro sin abrirlos uno a uno?

Abrir diez ficheros a mano es viable, pero no escala: si fueran mil, no lo harías así. Además, hacer `cat` de un fichero binario puede dejar la terminal llena de basura. La propia página del nivel avisa: si se te descontrola la terminal, prueba el comando `reset`.

Lo que necesitamos es una herramienta que nos diga **qué tipo de contenido** tiene cada fichero sin tener que abrirlo.

## La herramienta clave: `file`

`file` analiza el contenido de un fichero y te dice qué es. No se fía de la extensión, que en Linux no significa nada: lee los primeros bytes y los compara con firmas conocidas. Así distingue un texto plano (`ASCII text`), una imagen, un ejecutable o datos binarios sin formato reconocible (`data`).

## Mi solución: `find | xargs file`

```console
bandit4@bandit:~/inhere$ find . -type f -readable | xargs file
```

La salida muestra una línea por fichero con su tipo. Nueve salen como **`data`** y solo uno como **`ASCII text`**: ese es el nuestro.

Por partes:

1. **`find .`** busca a partir del directorio actual (`.`).
2. **`-type f`** se queda solo con ficheros, no con directorios.
3. **`-readable`** se queda con los ficheros que **nuestro usuario puede leer**. Es un filtro de **permisos**, no de contenido. Aquí no descarta nada, porque podemos leer todos; quítalo y verás el mismo resultado. Es útil en otros escenarios, cuando buscas por un sistema entero y no quieres que se te llene la pantalla de ficheros que no puedes abrir.
4. **`| xargs file`**: `find` imprime una lista de rutas, una por línea. `xargs` coge esa lista y la convierte en **argumentos** para `file`. Es decir, ejecuta `file ./-file00 ./-file01 ...` con todas las rutas a la vez.

¿Por qué hace falta `xargs`? Porque `file` no lee la lista de ficheros de la entrada estándar: espera recibirlos como argumentos. Con `find ... | file` a secas no funcionaría. `xargs` hace de puente entre "salida de un comando" y "argumentos de otro".

Y un detalle que lo hace funcionar: `find` devuelve las rutas como `./-file00`, **con `./` delante**. Es el mismo truco del nivel 1 → 2, así que `file` no confunde los nombres con opciones aunque empiecen por guion.

Con el fichero localizado, lo leemos. No te digo cuál es, compruébalo tú:

```console
bandit4@bandit:~/inhere$ cat ./-file0X
[REDACTED]
```

## Otras formas de resolverlo

```console
bandit4@bandit:~/inhere$ file ./*
bandit4@bandit:~/inhere$ find . -type f -exec file {} +
bandit4@bandit:~/inhere$ find . -type f | xargs file | grep text
```

- **`file ./*`**: la más corta. El `*` lo expande la shell a todos los ficheros del directorio. Fíjate en el `./`: con `file *` a secas, la shell pasaría `-file00`, `-file01`… y `file` los tomaría por opciones. Otra vez el truco del nivel 1 → 2.
- **`find . -type f -exec file {} +`**: hace lo mismo que `xargs` sin salir de `find`. `{}` se sustituye por las rutas encontradas y `+` le dice que las pase todas juntas en una sola ejecución.
- **`... | grep text`**: añade un filtro final para que solo aparezca la línea que interesa. Con diez ficheros da igual, pero con mil es la diferencia entre leer una línea o mil.

> **`xargs` y los espacios.** `xargs` separa los argumentos por espacios, así que se rompería con nombres como el `--spaces in this filename--` del [nivel 2 → 3](/posts/bandit-nivel-2-3/). La versión robusta es `find . -type f -print0 | xargs -0 file`, que separa las rutas con un carácter nulo en vez de con espacios. O directamente `-exec`, que no tiene ese problema.
{: .prompt-tip }

## Pasar al nivel 5

```console
bandit4@bandit:~/inhere$ exit
$ sshpass -p '[REDACTED]' ssh bandit5@bandit.labs.overthewire.org -p 2220
```

## Resumen

| Comando / concepto | Para qué sirve |
|---|---|
| *human-readable* | Contenido legible por una persona (texto), no "con permisos de lectura" |
| `file fichero` | Identificar el tipo de contenido por sus bytes, no por la extensión |
| `find . -type f` | Listar todos los ficheros a partir de un directorio |
| `-readable` | Filtrar por permisos de lectura del usuario actual |
| `xargs comando` | Convertir la salida de un comando en argumentos de otro |
| `find ... -exec cmd {} +` | Ejecutar un comando sobre lo encontrado sin usar `xargs` |
| `reset` | Recuperar la terminal si se descontrola tras mostrar datos binarios |

{% include series-nav.html %}
