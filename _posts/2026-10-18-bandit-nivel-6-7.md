---
title: "Bandit: nivel 6 → 7"
date: 2026-10-18 20:00:00 +0200
categories: [Linux, Bandit]
tags: [bandit, overthewire, linux, find, redirection, stderr]
description: La contraseña está en cualquier parte del servidor. Buscamos desde la raíz por dueño, grupo y tamaño con find y silenciamos los errores redirigiendo stderr a /dev/null.
ref: bandit-6-7
serie: bandit
nivel: 7
---

En el [nivel anterior](/posts/bandit-nivel-5-6/) buscábamos con `find` dentro de una carpeta concreta. Ahora el enunciado sube la apuesta: el fichero está **en algún lugar del servidor**, sin pistas de carpeta. Lo único que tenemos es su "retrato robot": dueño, grupo y tamaño. Toca buscar desde la raíz del sistema y, de paso, aprender a quitarnos de encima los cientos de errores de permisos que eso genera.

> Las contraseñas aparecen como `[REDACTED]`, siguiendo las [normas de OverTheWire](https://overthewire.org/rules/).
{: .prompt-warning }

## Conexión al nivel 6

```console
$ sshpass -p '[REDACTED]' ssh bandit6@bandit.labs.overthewire.org -p 2220
bandit6@bandit:~$ export TERM=xterm
```

Como siempre, `sshpass` nos pasa la contraseña al SSH sin tener que escribirla a mano, y `export TERM=xterm` evita los avisos de terminal desconocido al usar comandos como `clear`.

## El enunciado

Según el [enunciado del nivel 7](https://overthewire.org/wargames/bandit/bandit7.html), la contraseña está guardada en algún lugar del servidor en un fichero con estas propiedades:

- su dueño es el usuario **bandit7**
- pertenece al grupo **bandit6**
- ocupa **33 bytes**

Aquí no tiene sentido mirar en nuestro home con `ls`: el fichero puede estar en cualquier sitio. Necesitamos recorrer todo el sistema de ficheros.

## Buscar en todo el servidor con find

```console
bandit6@bandit:~$ find / -type f -user bandit7 -group bandit6 -size 33c 2>/dev/null
/var/lib/dpkg/info/bandit7.password
```

Un solo resultado. Desmontemos el comando pieza a pieza:

| Parte | Qué hace |
|---|---|
| `/` | Empieza a buscar en la raíz, es decir, en **todo** el sistema de ficheros. |
| `-type f` | Solo ficheros regulares (descarta directorios, enlaces, etc.). |
| `-user bandit7` | El **dueño** del fichero tiene que ser `bandit7`. |
| `-group bandit6` | El **grupo** del fichero tiene que ser `bandit6`. |
| `-size 33c` | Tamaño exacto de 33 **bytes**. |
| `2>/dev/null` | Manda los mensajes de error a la "papelera" del sistema. |

`find` combina todos los filtros con un **Y** lógico implícito: el fichero tiene que cumplir los cuatro a la vez.

### Ojo con la `c` de `-size`

Ya la usamos en el nivel anterior con `-size 1033c`, pero merece la pena insistir: **sin sufijo, `find` cuenta en bloques de 512 bytes**, no en bytes. Así que `-size 33` buscaría ficheros de unos 33 × 512 = 16 896 bytes, y no encontraría nada. La `c` (de *char*, un byte por carácter) es lo que hace que la búsqueda sea en bytes exactos.

Y los 33 bytes tienen sentido: las contraseñas de Bandit tienen 32 caracteres, más el salto de línea final.

### ¿Por qué `2>/dev/null`?

En Linux cada proceso tiene tres canales estándar, identificados por un número (*file descriptor*):

| Descriptor | Nombre | Para qué |
|---|---|---|
| `0` | stdin | Entrada |
| `1` | stdout | Salida normal (los resultados) |
| `2` | stderr | Mensajes de error |

Al buscar desde `/` como `bandit6`, `find` intenta entrar en un montón de directorios donde no tenemos permiso: `/root`, los homes de otros usuarios, partes de `/proc`... Cada uno de ellos genera una línea del tipo `Permission denied` por **stderr**. Sin redirigir, el único resultado bueno queda enterrado entre cientos de errores.

Con `2>` le decimos a la shell "lo que salga por el canal 2, mándalo aquí", y `/dev/null` es un fichero especial que se traga todo lo que le escribes. Resultado: los errores desaparecen y en pantalla solo queda **stdout**, que es justo lo que nos interesa.

> `2>/dev/null` no hace que `find` tenga más permisos ni que busque mejor: los errores siguen ocurriendo, simplemente no los vemos. Por eso es seguro usarlo aquí, donde sabemos que esos errores no nos aportan nada.
{: .prompt-info }

## Leer la contraseña

```console
bandit6@bandit:~$ cat /var/lib/dpkg/info/bandit7.password
[REDACTED]
```

Dos detalles curiosos:

- **El escondite.** `/var/lib/dpkg/info/` es donde `dpkg`, el gestor de paquetes de Debian/Ubuntu, guarda la información de los paquetes instalados (listas de ficheros, checksums...). Hay cientos de ficheros ahí, así que es un buen sitio para camuflar uno más. Buscar por nombre a ojo habría sido una pesadilla; buscar por propiedades lo encuentra al instante.
- **¿Por qué podemos leer un fichero que no es nuestro?** El dueño es `bandit7`, pero el grupo es `bandit6`, que es nuestro usuario. Los permisos de Linux se evalúan en tres niveles (dueño, grupo y otros), y lo que nos deja leerlo es el permiso de lectura del **grupo**. Por eso el enunciado nos da el grupo como pista: es la "puerta" que nos han dejado abierta.

## Otras formas

### 1. Buscar y leer en un solo paso con `-exec`

```console
bandit6@bandit:~$ find / -type f -user bandit7 -group bandit6 -size 33c -exec cat {} \; 2>/dev/null
[REDACTED]
```

`-exec` ejecuta un comando sobre cada fichero encontrado. `{}` se sustituye por la ruta del fichero y `\;` marca el final del comando (la barra escapa el `;` para que no se lo coma la shell). Si hubiera muchos resultados, `-exec cat {} +` pasaría todas las rutas de golpe a un solo `cat`, que es más eficiente.

### 2. Ver los permisos con `-ls`

```console
bandit6@bandit:~$ find / -type f -user bandit7 -group bandit6 -size 33c -ls 2>/dev/null
```

`-ls` muestra cada resultado con un formato parecido a `ls -l`: permisos, dueño, grupo, tamaño y fecha. Es una buena forma de **comprobar** que el fichero cumple las condiciones y de ver con tus propios ojos el permiso de lectura del grupo del que hablábamos antes. Lo mismo conseguirías con `ls -l /var/lib/dpkg/info/bandit7.password`.

### 3. Filtrar los errores con `grep` en lugar de tirarlos

```console
bandit6@bandit:~$ find / -type f -user bandit7 -group bandit6 -size 33c 2>&1 | grep -v "Permission denied"
```

Aquí hacemos lo contrario que antes: `2>&1` **une** stderr con stdout (manda el canal 2 al mismo sitio que el 1), así que todo pasa por la tubería, y `grep -v` descarta las líneas que contienen "Permission denied". Es más largo que `2>/dev/null`, pero tiene una ventaja: si apareciera algún error **distinto** (y quizá interesante), lo seguirías viendo.

## Pasar al nivel siguiente

```console
bandit6@bandit:~$ exit
$ sshpass -p '[REDACTED]' ssh bandit7@bandit.labs.overthewire.org -p 2220
```

## Resumen

| Comando / concepto | Para qué sirve |
|---|---|
| `find /` | Buscar en todo el sistema de ficheros |
| `-user` / `-group` | Filtrar por dueño y por grupo |
| `-size 33c` | Tamaño exacto en bytes (sin `c`, cuenta bloques de 512 bytes) |
| `2>/dev/null` | Descartar los mensajes de error (stderr) |
| `2>&1` | Unir stderr con stdout para procesarlos juntos |
| `-exec cmd {} \;` | Ejecutar un comando sobre cada resultado |
| `-ls` | Ver permisos, dueño, grupo y tamaño de cada resultado |
| Permisos de grupo | Permiten leer ficheros de otro dueño si compartimos grupo |

{% include series-nav.html %}
