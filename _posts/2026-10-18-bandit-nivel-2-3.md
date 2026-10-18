---
title: "Bandit: nivel 2 → 3"
date: 2026-10-18 14:00:00 +0200
categories: [Linux, Bandit]
tags: [bandit, overthewire, linux, cat, bash, quoting]
description: Un fichero con espacios y guiones en el nombre. Vemos cómo la shell trocea los argumentos por los espacios y cómo evitarlo con comillas, escapes y el autocompletado.
ref: bandit-2-3
serie: bandit
nivel: 3
---

Después del fichero llamado `-`, este nivel sube la apuesta: un fichero que se llama **`--spaces in this filename--`**. Tiene guiones al principio y **espacios** en medio, y los espacios son lo que nos va a dar guerra.

> Las contraseñas aparecen como `[REDACTED]`, siguiendo las [normas de OverTheWire](https://overthewire.org/rules/).
{: .prompt-warning }

## Conexión al nivel 2

```console
$ sshpass -p '[REDACTED]' ssh bandit2@bandit.labs.overthewire.org -p 2220
bandit2@bandit:~$ export TERM=xterm
```

## El enunciado

La [página del nivel 3](https://overthewire.org/wargames/bandit/bandit3.html) dice que la contraseña está en un fichero llamado `--spaces in this filename--`, en el directorio *home*:

```console
bandit2@bandit:~$ ls
--spaces in this filename--
```

## Primer intento: el truco del nivel anterior

Como el nombre empieza por guiones, es razonable pensar que `cat` a secas lo confundirá con una opción. En el [nivel anterior](/posts/bandit-nivel-1-2/) lo resolvimos poniendo `./` delante, así que probamos lo mismo:

```console
bandit2@bandit:~$ cat ./--spaces in this filename--
cat: ./--spaces: No such file or directory
cat: in: No such file or directory
cat: this: No such file or directory
cat: filename--: No such file or directory
```

Cuatro errores, uno por cada palabra del nombre. `cat` ha intentado abrir **cuatro ficheros distintos**: `./--spaces`, `in`, `this` y `filename--`. Ninguno existe.

## Por qué pasa: la shell trocea por los espacios

Antes de ejecutar un comando, la **shell** (bash, en este caso) lee la línea y la divide en palabras usando los espacios como separador. La primera palabra es el comando y el resto son sus argumentos. A `cat` no le llega la línea tal cual escribimos, sino ya troceada:

| Lo que escribimos | Lo que recibe `cat` |
|---|---|
| `cat ./--spaces in this filename--` | 4 argumentos: `./--spaces`, `in`, `this`, `filename--` |

`cat` no tiene forma de saber que esas cuatro palabras eran en realidad un solo nombre. Hace lo que tiene que hacer: intenta abrir cuatro ficheros. El `./` arregló el problema de los guiones, pero el de los espacios es otro distinto.

## La solución: comillas

```console
bandit2@bandit:~$ cat ./"--spaces in this filename--"
[REDACTED]
```

Las **comillas** le dicen a la shell: "todo lo que hay aquí dentro es **una sola palabra**, aunque tenga espacios". Ahora `cat` recibe un único argumento, `./--spaces in this filename--`, y abre el fichero correcto.

Las comillas no tienen por qué rodear la palabra entera. Pueden ir solo sobre la parte problemática, como aquí, o sobre todo: `cat "./--spaces in this filename--"` funciona igual. La shell quita las comillas y junta lo que hay a los lados en una misma palabra.

> **Comillas dobles o simples:** para este caso dan igual. La diferencia es que dentro de las **dobles** (`"..."`) la shell sigue interpretando algunos caracteres especiales, como `$` para las variables, y dentro de las **simples** (`'...'`) todo se toma literalmente. Si el nombre de un fichero tuviera un `$`, habría que usar simples.
{: .prompt-info }

## ¿Y por qué sigue haciendo falta el `./`?

Las comillas solucionan los espacios, pero no los guiones. Sin el `./`:

```console
bandit2@bandit:~$ cat "--spaces in this filename--"
cat: unrecognized option '--spaces in this filename--'
```

Ahora `cat` recibe una sola palabra, pero empieza por `--`, así que la toma por una opción larga que no existe. Necesitamos las dos cosas: **comillas para los espacios y `./` para los guiones**.

## Otras formas de resolverlo

```console
bandit2@bandit:~$ cat ./--spaces\ in\ this\ filename--
bandit2@bandit:~$ cat -- "--spaces in this filename--"
```

- **Escapar los espacios con `\`**: la barra invertida le dice a la shell que el carácter siguiente es literal y no un separador. Es justo lo que hace bash cuando autocompletas.
- **`--` para terminar las opciones**: en el nivel anterior vimos que `--` no servía para un guion solo. Aquí **sí funciona**, porque el nombre *empieza* por guiones pero no es solo un guion. Todo lo que va después de `--` se trata como fichero, aunque empiece por `-`.

> **El truco más práctico es el tabulador.** Escribe `cat ./--sp` y pulsa **Tab**. Bash completa el nombre y escapa los espacios él solo. Así se evitan errores al escribir nombres raros, y además ves cómo lo habría escrito un experto.
{: .prompt-tip }

## Pasar al nivel 3

```console
bandit2@bandit:~$ exit
$ sshpass -p '[REDACTED]' ssh bandit3@bandit.labs.overthewire.org -p 2220
```

## Resumen

| Concepto | Qué hemos aprendido |
|---|---|
| División en palabras | La shell separa la línea por los espacios antes de ejecutar el comando |
| `"..."` | Agrupa en un solo argumento; dentro aún se interpretan cosas como `$` |
| `'...'` | Agrupa en un solo argumento; todo se toma literalmente |
| `\ ` | Escapa un espacio para que no actúe como separador |
| `--` | Marca el final de las opciones: lo que sigue es un fichero aunque empiece por `-` |
| Tab | Autocompleta y escapa los nombres por ti |

{% include series-nav.html %}
