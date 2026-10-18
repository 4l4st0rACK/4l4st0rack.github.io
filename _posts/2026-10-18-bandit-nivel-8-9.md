---
title: "Bandit: nivel 8 → 9"
date: 2026-10-18 21:15:00 +0200
categories: [Linux, Bandit]
tags: [bandit, overthewire, linux, sort, uniq, pipes]
description: Miles de líneas repetidas y una sola que aparece una vez. Combinamos sort y uniq -u para encontrarla, y entendemos por qué uniq no sirve sin ordenar antes.
ref: bandit-8-9
serie: bandit
nivel: 9
---

En el [nivel anterior](/posts/bandit-nivel-7-8/) buscábamos una línea por su **contenido**: la que tenía la palabra `millionth`. Ahora no tenemos ninguna palabra que buscar. La única pista es **cuántas veces aparece** la línea: todas las demás están repetidas y la contraseña es la única que sale una sola vez. Para eso, `grep` no nos sirve. Toca presentar a otra pareja clásica: `sort` y `uniq`.

> Las contraseñas aparecen como `[REDACTED]`, siguiendo las [normas de OverTheWire](https://overthewire.org/rules/).
{: .prompt-warning }

## Conexión al nivel 8

```console
$ sshpass -p '[REDACTED]' ssh bandit8@bandit.labs.overthewire.org -p 2220
bandit8@bandit:~$ export TERM=xterm
```

## El enunciado

Según el [enunciado del nivel 9](https://overthewire.org/wargames/bandit/bandit9.html), la contraseña está en el fichero `data.txt` y es **la única línea que aparece una sola vez**.

```console
bandit8@bandit:~$ ls
data.txt
```

## La solución: sort + uniq -u

```console
bandit8@bandit:~$ cat data.txt | sort | uniq -u
[REDACTED]
```

Tres comandos encadenados con tuberías, como en el nivel anterior. Veamos qué aporta cada uno:

| Parte | Qué hace |
|---|---|
| `cat data.txt` | Vuelca el contenido del fichero. |
| `sort` | Ordena las líneas alfabéticamente, de forma que las repetidas quedan **juntas**. |
| `uniq -u` | Muestra solo las líneas que **no** tienen ninguna copia justo al lado (*unique*). |

### ¿Por qué hace falta `sort`?

Este es el detalle clave del nivel: **`uniq` solo compara cada línea con la inmediatamente anterior**. No recorre el fichero entero recordando lo que ha visto; solo mira a su vecina. Por eso, si dos copias de la misma línea están separadas por otras, `uniq` las trata como distintas.

Un ejemplo sencillo, con un fichero inventado de cinco líneas:

```text
pera
manzana
pera
kiwi
manzana
```

Aquí `uniq -u` sin ordenar mostraría las cinco líneas, porque ninguna tiene una copia **justo al lado**. Pero si ordenamos primero:

```text
kiwi
manzana
manzana
pera
pera
```

Ahora las repetidas están juntas, y `uniq -u` deja solo `kiwi`, que es la única que aparece una vez. Exactamente lo que necesitamos con `data.txt`.

> Regla práctica: casi siempre que uses `uniq`, ponle un `sort` delante. Sin ordenar, el resultado depende del orden del fichero y casi nunca es lo que esperas.
{: .prompt-tip }

### Las opciones de `uniq`

| Comando | Qué muestra |
|---|---|
| `uniq` | Una copia de cada línea (elimina los duplicados consecutivos) |
| `uniq -u` | Solo las líneas que aparecen **una vez** |
| `uniq -d` | Solo las líneas que están **repetidas** (una copia de cada) |
| `uniq -c` | Cada línea precedida del **número de veces** que aparece |

## Otras formas

### 1. Sin `cat`

```console
bandit8@bandit:~$ sort data.txt | uniq -u
[REDACTED]
```

Igual que con `grep` en el nivel anterior, `sort` acepta el fichero como argumento, así que el `cat` sobra. Mismo resultado, un proceso menos.

### 2. Contando las apariciones con `uniq -c`

```console
bandit8@bandit:~$ sort data.txt | uniq -c | sort -n | head -n 1
```

Este camino es más largo, pero te enseña **qué hay dentro** del fichero:

- `uniq -c` pone delante de cada línea cuántas veces aparece.
- `sort -n` ordena por ese número de forma **numérica** (sin `-n`, `sort` compararía como texto y `10` iría antes que `9`).
- `head -n 1` se queda con la primera línea: la que tiene el contador más bajo, es decir, la que aparece una sola vez, con un `1` delante.

Si quitas el `head`, verás también cuántas veces se repite cada una de las otras líneas, que es una buena forma de entender cómo está construido el reto.

### 3. Solo con awk, sin ordenar

```console
bandit8@bandit:~$ awk '{c[$0]++} END {for (l in c) if (c[l] == 1) print l}' data.txt
[REDACTED]
```

Volvemos a `awk`, que conocimos en el nivel anterior, pero ahora usando un **array asociativo** (un diccionario):

- `{c[$0]++}` se ejecuta en cada línea: usa la línea completa (`$0`) como clave y suma uno a su contador.
- `END {...}` se ejecuta una sola vez, cuando ya se ha leído todo el fichero.
- `for (l in c) if (c[l] == 1) print l` recorre todas las claves e imprime las que tienen el contador a 1.

A diferencia de `uniq`, `awk` **sí recuerda** todas las líneas que ha visto, así que no necesita que el fichero esté ordenado.

## Pasar al nivel siguiente

```console
bandit8@bandit:~$ exit
$ sshpass -p '[REDACTED]' ssh bandit9@bandit.labs.overthewire.org -p 2220
```

## Resumen

| Comando / concepto | Para qué sirve |
|---|---|
| `sort` | Ordenar líneas, dejando juntas las repetidas |
| `sort -n` | Ordenar numéricamente |
| `uniq -u` | Mostrar solo las líneas que aparecen una vez |
| `uniq -c` | Contar cuántas veces aparece cada línea |
| `uniq -d` | Mostrar solo las líneas repetidas |
| `head -n 1` | Quedarse con la primera línea |
| `awk` con arrays | Contar apariciones sin necesidad de ordenar |
| `uniq` sin `sort` | Solo compara líneas consecutivas: casi siempre necesita `sort` delante |

{% include series-nav.html %}
