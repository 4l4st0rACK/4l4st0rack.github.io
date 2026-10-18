---
title: "Bandit: nivel 3 → 4"
date: 2026-10-18 15:00:00 +0200
categories: [Linux, Bandit]
tags: [bandit, overthewire, linux, ls, cd, hidden-files]
description: Un directorio que parece vacío pero no lo está. Aprendemos a movernos con cd y a ver los ficheros ocultos con ls -la.
ref: bandit-3-4
serie: bandit
nivel: 4
---

En este nivel la contraseña está en un directorio que, a primera vista, **parece vacío**. Nos sirve para aprender a movernos entre directorios con `cd` y para descubrir los **ficheros ocultos** de Linux, que se usan muchísimo y son de lo primero que conviene revisar al explorar un sistema.

> Las contraseñas aparecen como `[REDACTED]`, siguiendo las [normas de OverTheWire](https://overthewire.org/rules/).
{: .prompt-warning }

## Conexión al nivel 3

```console
$ sshpass -p '[REDACTED]' ssh bandit3@bandit.labs.overthewire.org -p 2220
bandit3@bandit:~$ export TERM=xterm
```

## El enunciado

La [página del nivel 4](https://overthewire.org/wargames/bandit/bandit4.html) dice que la contraseña está en un **fichero oculto** dentro del directorio `inhere`. Miramos qué hay en nuestro *home*:

```console
bandit3@bandit:~$ ls
inhere
```

## Entrar en el directorio: `cd`

Para movernos a `inhere` usamos `cd` (*change directory*):

```console
bandit3@bandit:~$ cd inhere
bandit3@bandit:~/inhere$
```

Fíjate en que el **prompt ha cambiado**: antes ponía `~` y ahora pone `~/inhere`. El prompt siempre muestra en qué directorio estás, así que es la forma más rápida de saber dónde te encuentras sin tener que hacer `pwd`.

## Primer intento: `ls`

```console
bandit3@bandit:~/inhere$ ls
bandit3@bandit:~/inhere$
```

Nada. `ls` no muestra ningún fichero. ¿Está vacío? El enunciado ya nos daba la pista: el fichero está **oculto**.

## Qué es un fichero oculto en Linux

En Linux no hay un atributo especial para ocultar ficheros como en Windows. La regla es mucho más sencilla: **cualquier fichero o directorio cuyo nombre empiece por punto (`.`) es oculto**, y `ls` no lo muestra por defecto.

Es una convención pensada para no llenar el listado de ficheros de configuración. Tu *home* está lleno de ellos: `.bashrc`, `.profile`, `.ssh/`… No los ves con un `ls` normal, pero están ahí.

> Que un fichero esté oculto **no lo protege en absoluto**. Cualquiera con permisos para listar el directorio lo ve con una sola opción de `ls`. Por eso, al explorar un sistema, revisar los ficheros ocultos es de lo primero que se hace: es donde suelen estar configuraciones, historiales y, a veces, credenciales olvidadas.
{: .prompt-danger }

## La solución: `ls -la`

`ls` tiene opciones para mostrar también los ocultos. La combinación más habitual es `-la`:

```console
bandit3@bandit:~/inhere$ ls -la
```

Entre las líneas de la salida aparece por fin nuestro fichero: **`...Hiding-From-You`**. Las dos opciones hacen cosas distintas:

- **`-a`** (*all*) muestra **todos** los ficheros, también los que empiezan por punto.
- **`-l`** (*long*) muestra el formato largo, con una línea por fichero: permisos, propietario, grupo, tamaño, fecha y nombre.

Si solo quieres ver los nombres, `ls -a` es más limpio:

```console
bandit3@bandit:~/inhere$ ls -a
.  ..  ...Hiding-From-You
```

Aquí aparecen tres cosas, y conviene no confundirlas:

| Nombre | Qué es |
|---|---|
| `.` | El directorio actual. Lo usamos en el [nivel 1 → 2](/posts/bandit-nivel-1-2/) con `./-` |
| `..` | El directorio padre: `cd ..` te sube un nivel |
| `...Hiding-From-You` | Nuestro fichero. Empieza por punto, así que está oculto |

El nombre está elegido con mala idea: empieza por **tres puntos**, así que de un vistazo rápido se confunde con `.` y `..`.

## Leer el fichero

```console
bandit3@bandit:~/inhere$ cat ...Hiding-From-You
[REDACTED]
```

Aquí no hace falta ningún truco. El nombre no empieza por guion ni tiene espacios, así que `cat` lo abre sin problema. Lo raro del fichero es solo que está oculto.

## Otras formas de resolverlo

```console
bandit3@bandit:~$ ls -A inhere
bandit3@bandit:~$ cat inhere/...Hiding-From-You
bandit3@bandit:~$ find inhere -type f
```

- **`ls -A`**: como `-a`, pero sin mostrar `.` y `..`. Solo deja los ficheros reales, ocultos incluidos.
- **`cat inhere/...Hiding-From-You`**: no hace falta entrar con `cd`. Puedes leer el fichero desde el *home* usando su ruta relativa.
- **`find inhere -type f`**: `find` recorre el directorio y lista todos los ficheros (`-type f`), estén ocultos o no. Con un solo fichero da igual, pero cuando un directorio tiene subdirectorios y cientos de ficheros, `find` es la herramienta. Nos va a hacer mucha falta en los próximos niveles.

## Pasar al nivel 4

```console
bandit3@bandit:~/inhere$ exit
$ sshpass -p '[REDACTED]' ssh bandit4@bandit.labs.overthewire.org -p 2220
```

## Resumen

| Comando / concepto | Para qué sirve |
|---|---|
| `cd directorio` | Moverse a otro directorio |
| `cd ..` | Subir al directorio padre |
| Fichero oculto | Cualquier nombre que empiece por `.`; es una convención, no una protección |
| `ls -a` / `ls -A` | Listar también los ocultos (`-A` sin `.` ni `..`) |
| `ls -l` | Formato largo: permisos, propietario, tamaño, fecha |
| `find dir -type f` | Buscar todos los ficheros dentro de un directorio |

{% include series-nav.html %}
