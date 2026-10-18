---
title: "Bandit: nivel 1 → 2"
date: 2026-10-18 13:00:00 +0200
categories: [Linux, Bandit]
tags: [bandit, overthewire, linux, cat, stdin, paths]
description: Un fichero que se llama simplemente "-". Vemos por qué cat no lo abre, qué es la entrada estándar y cómo usar rutas relativas para leerlo.
ref: bandit-1-2
serie: bandit
nivel: 2
---

En este nivel la contraseña está en un fichero con un nombre muy poco habitual: **`-`**, un guion. Parece una tontería, pero sirve para entender cómo interpreta la terminal los argumentos y qué es la **entrada estándar**.

> Las contraseñas aparecen como `[REDACTED]`, siguiendo las [normas de OverTheWire](https://overthewire.org/rules/).
{: .prompt-warning }

## Conexión al nivel 1

Entramos con la contraseña que conseguimos en el [nivel anterior](/posts/bandit-nivel-0-1/) y arreglamos la terminal, como en cada nivel:

```console
$ sshpass -p '[REDACTED]' ssh bandit1@bandit.labs.overthewire.org -p 2220
bandit1@bandit:~$ export TERM=xterm
```

## El enunciado

La [página del nivel 2](https://overthewire.org/wargames/bandit/bandit2.html) dice que la contraseña está en un fichero llamado **`-`** en el directorio *home*. Lo comprobamos:

```console
bandit1@bandit:~$ ls
-
```

Ahí está. Así que hacemos lo mismo que en el nivel 0, ¿no?

## Por qué `cat -` no funciona

```console
bandit1@bandit:~$ cat -
```

El comando no muestra nada y se queda esperando, como si se hubiera colgado. No es un error: `cat` está haciendo exactamente lo que le hemos pedido.

En Linux hay una convención que siguen muchísimos programas: **un guion solo (`-`) como argumento significa "lee de la entrada estándar"** (*stdin*) en lugar de un fichero. La entrada estándar, cuando no hay nada encadenado, es el teclado. Así que `cat -` está esperando a que escribamos algo para repetirlo en pantalla.

Si escribes cualquier cosa y pulsas Enter, verás que `cat` te la devuelve. Para salir pulsa **Ctrl+C** (cancelar) o **Ctrl+D** (fin de la entrada).

Esa convención existe para poder encadenar comandos. Por ejemplo, `echo hola | cat -` hace que `cat` lea lo que le llega por la tubería. El problema es que nuestro fichero se llama justo igual que esa palabra reservada.

## La solución: `cat ./-`

```console
bandit1@bandit:~$ cat ./-
[REDACTED]
```

La clave es que `./-` ya no es un guion solo, sino una **ruta**:

- `.` significa "el directorio en el que estoy ahora".
- `/` es el separador de directorios.
- `-` es el nombre del fichero.

`./-` se lee como "el fichero llamado `-` que está en este directorio". Apunta exactamente al mismo fichero que `-`, pero como el argumento ya no es un guion suelto, `cat` no lo confunde con la entrada estándar y lo abre como un fichero normal.

> El mismo truco sirve para cualquier fichero cuyo nombre empiece por guion. Un fichero llamado `-v` haría que muchos comandos lo tomaran por una opción. Con `./-v` se acaba el problema.
{: .prompt-tip }

## Otras formas de leerlo

Hay más de una manera de resolverlo, y cada una enseña algo:

```console
bandit1@bandit:~$ cat /home/bandit1/-
bandit1@bandit:~$ cat < -
```

- **`cat /home/bandit1/-`**: la misma idea que `./-`, pero con la **ruta absoluta**, desde la raíz del sistema.
- **`cat < -`**: aquí `cat` no recibe ningún argumento. Es la **shell** la que abre el fichero `-` con la redirección `<` y se lo pasa a `cat` como entrada estándar. Como la shell no aplica la convención del guion, abre el fichero sin problema.

> Seguramente pienses en `cat -- -`, porque `--` se usa para decir "aquí terminan las opciones". **No funciona en este caso.** `--` sirve para nombres que *empiezan* por guion, como `-v`, pero un guion solo sigue significando entrada estándar aunque vaya después de `--`.
{: .prompt-info }

## Pasar al nivel 2

```console
bandit1@bandit:~$ exit
$ sshpass -p '[REDACTED]' ssh bandit2@bandit.labs.overthewire.org -p 2220
```

## Resumen

| Concepto | Qué hemos aprendido |
|---|---|
| `-` como argumento | Muchos programas lo interpretan como "leer de la entrada estándar" |
| Entrada estándar (*stdin*) | Lo que recibe un programa: el teclado o lo que llega por una tubería |
| `Ctrl+C` / `Ctrl+D` | Cancelar un comando / indicar fin de la entrada |
| `./fichero` | Ruta relativa al directorio actual |
| `/home/usuario/fichero` | Ruta absoluta, desde la raíz |
| `comando < fichero` | La shell abre el fichero y lo pasa como entrada estándar |

{% include series-nav.html %}
