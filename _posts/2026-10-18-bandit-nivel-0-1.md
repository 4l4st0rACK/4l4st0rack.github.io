---
title: "Bandit: nivel 0 → 1"
date: 2026-10-18 12:00:00 +0200
categories: [Linux, Bandit]
tags: [bandit, overthewire, ssh, sshpass, awk, linux]
description: Empezamos el wargame Bandit de OverTheWire. Entramos por SSH, arreglamos la terminal y sacamos la contraseña del nivel 1 filtrando un fichero con tail y awk.
ref: bandit-0-1
serie: bandit
nivel: 1
---

**Bandit** es el wargame de [OverTheWire](https://overthewire.org/wargames/bandit/) pensado para empezar desde cero. Cada nivel es un pequeño reto en un servidor Linux. Para superarlo tienes que encontrar la contraseña que da acceso al siguiente nivel, y para encontrarla vas aprendiendo a manejar la terminal.

En esta serie resuelvo los niveles uno a uno, y no me quedo en el comando que hay que escribir: explico **por qué** funciona y busco la forma más limpia de hacerlo.

> Siguiendo las [normas de OverTheWire](https://overthewire.org/rules/), las contraseñas de los niveles no se publican. En los ejemplos aparecen como `[REDACTED]`. La gracia es que las consigas tú.
{: .prompt-warning }

## Qué necesitas

- **Una terminal con SSH.** En Linux y macOS ya viene. En Windows 10/11, PowerShell trae `ssh` de serie, aunque para seguir esta serie te recomiendo **WSL**.
- **`sshpass`** (opcional). Permite pasarle la contraseña a SSH en el mismo comando, sin que te la pida. Como vas a conectarte muchas veces, ahorra tiempo:

```console
$ sudo apt install sshpass
```

## Conexión al nivel 0

Los datos de conexión están en la [página del nivel 0](https://overthewire.org/wargames/bandit/bandit0.html): servidor `bandit.labs.overthewire.org`, puerto `2220`, usuario `bandit0` y contraseña `bandit0`. Es la única contraseña que da la propia web; a partir de aquí, cada una la consigues resolviendo el nivel anterior.

```console
$ sshpass -p 'bandit0' ssh bandit0@bandit.labs.overthewire.org -p 2220
```

Por partes:

- `sshpass -p 'bandit0'`: sshpass le entrega la contraseña `bandit0` a SSH cuando la pida.
- `ssh bandit0@bandit.labs.overthewire.org`: conectarse como el **usuario** `bandit0` al **servidor** `bandit.labs.overthewire.org`, con el formato `usuario@servidor`.
- `-p 2220`: el **puerto**. SSH usa el 22 por defecto, pero Bandit escucha en el 2220.

Fíjate en que **`-p` aparece dos veces y significa cosas distintas**. El primero es de `sshpass` y es la contraseña (*password*). El segundo es de `ssh` y es el puerto (*port*). Cada opción pertenece al comando que tiene delante.

> **Si sshpass no hace nada la primera vez:** la primera vez que te conectas a un servidor, SSH pregunta si confías en su huella (*fingerprint*) y espera un `yes`. sshpass no sabe responder a esa pregunta, así que se queda parado o sale sin conectar. Conéctate una vez con `ssh` a secas y acepta con `yes`, o añade `-o StrictHostKeyChecking=accept-new` al comando para aceptar servidores nuevos automáticamente.
{: .prompt-tip }

> Pasar contraseñas en el propio comando es cómodo para un laboratorio, pero **en un entorno real es mala práctica**: la contraseña queda en el historial de la shell y otros usuarios de la máquina pueden verla en la lista de procesos. En sistemas de verdad se usan claves SSH.
{: .prompt-danger }

## Arreglar la terminal: `export TERM=xterm`

Nada más entrar, lo primero que hago en cada nivel es esto:

```console
bandit0@bandit:~$ export TERM=xterm
```

La variable de entorno `TERM` indica qué tipo de terminal estás usando, y los programas la consultan para saber cómo pintar en pantalla. Al conectarte por SSH, el servidor hereda el valor de tu terminal local. Si el servidor no conoce ese tipo, comandos como `clear` o editores como `nano` fallan o se ven mal. Con `export TERM=xterm` le decimos que use `xterm`, un tipo que cualquier sistema reconoce.

`export` hace que la variable esté disponible también para los programas que lances desde esa shell. Solo dura la sesión, por eso hay que repetirlo cada vez que entras en un nivel.

## Encontrar la contraseña del nivel 1

La [página del nivel 1](https://overthewire.org/wargames/bandit/bandit1.html) dice que la contraseña está en un fichero llamado **`readme`** en el directorio *home*. Miramos qué hay:

```console
bandit0@bandit:~$ ls
readme
```

Con `cat` vemos su contenido:

```console
bandit0@bandit:~$ cat readme
```

Sale un mensaje de bienvenida de OverTheWire y, en la última línea con texto, la contraseña. Podríamos copiarla a mano y seguir, pero la idea de esta serie es aprender a usar la terminal. Así que vamos a sacar **solo la contraseña**.

## Filtrar la salida: `cat | tail | awk`

```console
bandit0@bandit:~$ cat readme | tail -n 2 | awk '{print $NF}'
[REDACTED]
```

La barra vertical `|` es una **tubería** (*pipe*): coge la salida de un comando y se la pasa como entrada al siguiente. Así encadenamos herramientas pequeñas, y cada una hace una sola cosa:

1. **`cat readme`** saca el contenido completo del fichero.
2. **`tail -n 2`** se queda solo con las **2 últimas líneas**. Cogemos dos y no una porque el fichero termina con una línea en blanco, y la contraseña está en la penúltima.
3. **`awk '{print $NF}'`** imprime el **último campo** de cada línea. `awk` trocea cada línea por los espacios. `NF` (*Number of Fields*) es el número de trozos de la línea, así que `$NF` es el último. En la línea de la contraseña, el último trozo es justo la contraseña. En la línea en blanco no hay nada que imprimir.

> **Versión más corta:** `awk` puede leer el fichero directamente y filtrar por contenido, sin `cat` ni `tail`: `awk '/password/ {print $NF}' readme`. Imprime el último campo solo de las líneas que contienen la palabra *password*. Encadenar comandos es la mejor forma de aprender, pero conviene saber que muchas veces hay un atajo.
{: .prompt-info }

## Pasar al nivel 1

Con la contraseña en la mano, salimos y nos conectamos como `bandit1`:

```console
bandit0@bandit:~$ exit
$ sshpass -p '[REDACTED]' ssh bandit1@bandit.labs.overthewire.org -p 2220
```

> Ve guardando las contraseñas en un fichero de notas en tu equipo. Bandit no las recuerda por ti.
{: .prompt-tip }

## Resumen

| Comando | Para qué sirve |
|---|---|
| `sshpass -p 'pass' ssh usuario@host -p puerto` | Conectarse por SSH pasando la contraseña en el comando |
| `export TERM=xterm` | Decirle al servidor qué tipo de terminal usamos, para que `clear` y compañía funcionen |
| `ls` | Listar ficheros |
| `cat fichero` | Mostrar el contenido de un fichero |
| `comando1 \| comando2` | Pasar la salida de un comando como entrada de otro |
| `tail -n N` | Quedarse con las últimas N líneas |
| `awk '{print $NF}'` | Imprimir el último campo de cada línea |

{% include series-nav.html %}
