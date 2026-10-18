---
title: "Bandit: nivel 7 → 8"
date: 2026-10-18 21:00:00 +0200
categories: [Linux, Bandit]
tags: [bandit, overthewire, linux, grep, awk, pipes]
description: Un fichero enorme y una sola línea que nos interesa. Encadenamos grep y awk con tuberías para quedarnos solo con la contraseña junto a la palabra millionth.
ref: bandit-7-8
serie: bandit
nivel: 8
---

En el [nivel anterior](/posts/bandit-nivel-6-7/) teníamos que encontrar un fichero perdido en todo el servidor. Ahora sabemos exactamente en qué fichero está la contraseña, pero el problema es otro: ese fichero es **enorme**, y lo que buscamos es una sola línea entre miles. Es el momento de presentar a dos herramientas básicas para trabajar con texto en Linux: `grep` para filtrar líneas y `awk` para quedarnos con la parte de la línea que nos interesa.

> Las contraseñas aparecen como `[REDACTED]`, siguiendo las [normas de OverTheWire](https://overthewire.org/rules/).
{: .prompt-warning }

## Conexión al nivel 7

```console
$ sshpass -p '[REDACTED]' ssh bandit7@bandit.labs.overthewire.org -p 2220
bandit7@bandit:~$ export TERM=xterm
```

## El enunciado

Según el [enunciado del nivel 8](https://overthewire.org/wargames/bandit/bandit8.html), la contraseña está en el fichero `data.txt`, **al lado de la palabra `millionth`**.

```console
bandit7@bandit:~$ ls
data.txt
```

Si probamos a abrirlo directamente con `cat data.txt`, la terminal se llena de líneas y más líneas: cada una tiene una palabra seguida de una cadena que parece una contraseña. Leerlo a ojo no es una opción. Necesitamos **filtrar**.

> Para hacerte una idea del tamaño de un fichero antes de abrirlo, `wc -l data.txt` te dice cuántas líneas tiene. Así evitas inundar la terminal.
{: .prompt-tip }

## Filtrar con grep

```console
bandit7@bandit:~$ cat data.txt | grep "millionth"
millionth	[REDACTED]
```

Aquí entran en juego dos piezas:

- **La tubería `|`** conecta la salida estándar (stdout) del comando de la izquierda con la entrada estándar (stdin) del de la derecha. `cat` vuelca el fichero entero, pero en lugar de ir a la pantalla, ese texto entra en `grep`.
- **`grep "millionth"`** lee línea a línea y solo deja pasar las que contienen el patrón `millionth`. Todo lo demás se descarta.

Resultado: de todo el fichero, solo sobrevive la línea que buscamos, con la palabra y la contraseña.

## Quedarnos solo con la contraseña con awk

La línea anterior ya nos da la contraseña, pero todavía lleva delante la palabra `millionth`. Para sacar **solo** la contraseña, añadimos un eslabón más a la cadena:

```console
bandit7@bandit:~$ cat data.txt | grep "millionth" | awk '{print $2}'
[REDACTED]
```

`awk` es una herramienta (y en realidad un pequeño lenguaje) para procesar texto por columnas:

- Por defecto, divide cada línea en **campos** usando los espacios en blanco (espacios y tabuladores) como separador.
- `$1` es el primer campo (la palabra `millionth`), `$2` el segundo (la contraseña), y `$0` sería la línea completa.
- `{print $2}` significa "para cada línea que te llegue, imprime el segundo campo".

> Fíjate en que el programa de `awk` va entre **comillas simples**. Es importante: con comillas dobles, la shell intentaría sustituir `$2` por una variable propia (que estaría vacía) antes de que `awk` llegara a verlo, y el comando no haría lo que esperas.
{: .prompt-info }

## Otras formas

### 1. Sin `cat`: grep puede leer el fichero directamente

```console
bandit7@bandit:~$ grep "millionth" data.txt
millionth	[REDACTED]
```

`grep` acepta el nombre del fichero como argumento, así que `cat data.txt |` sobra. Funciona igual, pero te ahorras un proceso. Tiene hasta nombre propio en el mundillo: *useless use of cat* ("uso inútil de cat"). Con `cat` no está mal, solo es redundante.

### 2. Solo con awk

```console
bandit7@bandit:~$ awk '/millionth/ {print $2}' data.txt
[REDACTED]
```

`awk` también sabe filtrar: el patrón entre barras `/millionth/` hace de `grep`, y el bloque `{print $2}` solo se ejecuta en las líneas que coinciden. Un único comando hace el trabajo de los tres.

### 3. Con `cut` y búsqueda por palabra completa

```console
bandit7@bandit:~$ grep -w "millionth" data.txt | cut -f2
[REDACTED]
```

- `grep -w` busca la **palabra completa**: no coincidiría con algo como `millionths` si existiera en el fichero. Aquí no cambia el resultado, pero es buena costumbre cuando buscas una palabra concreta.
- `cut -f2` saca el segundo campo, pero **solo si el separador es un tabulador**, que es el que usa `cut` por defecto (a diferencia de `awk`, que acepta cualquier espacio en blanco). Puedes comprobar qué separador hay con `grep millionth data.txt | cat -A`: los tabuladores aparecen como `^I`.

## Pasar al nivel siguiente

```console
bandit7@bandit:~$ exit
$ sshpass -p '[REDACTED]' ssh bandit8@bandit.labs.overthewire.org -p 2220
```

## Resumen

| Comando / concepto | Para qué sirve |
|---|---|
| Tubería (barra vertical) | Pasar la salida de un comando como entrada del siguiente |
| `grep patrón fichero` | Mostrar solo las líneas que contienen el patrón |
| `grep -w` | Buscar la palabra completa |
| `awk '{print $2}'` | Imprimir el segundo campo de cada línea |
| `awk '/patrón/ {print $2}'` | Filtrar e imprimir un campo en un solo paso |
| `cut -f2` | Sacar el segundo campo separado por tabuladores |
| `wc -l` | Contar las líneas de un fichero |
| Comillas simples | Evitan que la shell interprete `$2` antes que `awk` |

{% include series-nav.html %}
