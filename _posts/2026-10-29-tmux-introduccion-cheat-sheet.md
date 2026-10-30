---
layout: post
title: "tmux: introducción y comandos esenciales"
date: 2026-10-29 19:47:00 +0100
categories: [Linux, Terminal]
tags: [tmux, terminal, ssh]
description: "Introducción breve a tmux y una chuleta con los comandos y atajos esenciales para trabajar con sesiones, ventanas y paneles."
---

Si trabajamos habitualmente desde la terminal, es bastante común terminar con varias ventanas abiertas: una para ejecutar comandos, otra para revisar logs y alguna conexión SSH a un servidor.

**tmux** permite organizar todo esto dentro de una misma sesión. Podemos dividir la pantalla en paneles, crear ventanas y desconectarnos sin detener los procesos que siguen ejecutándose.

Esto resulta especialmente útil por SSH: si perdemos la conexión, podemos volver a conectarnos y recuperar la sesión, siempre que tmux siga activo en el servidor.

## Tres conceptos básicos

- **Sesión:** agrupa nuestro espacio de trabajo.
- **Ventana:** funciona como una pestaña dentro de la sesión.
- **Panel:** cada una de las terminales en las que podemos dividir una ventana.

## Crear una sesión y volver a ella

Con tmux instalado, creamos una sesión con nombre:

```shell
tmux new -s trabajo
```

Para desconectarnos sin cerrarla, pulsamos `Ctrl + b`, soltamos ambas teclas y después pulsamos `d`.

Cuando queramos volver:

```shell
tmux attach -t trabajo
```

## Cheat sheet

Estos son los comandos y atajos esenciales con la configuración predeterminada.

### Sesiones

Los siguientes comandos se ejecutan desde la terminal:

| Comando | Acción |
| --- | --- |
| `tmux new -s nombre` | Crear una sesión con nombre |
| `tmux ls` | Listar las sesiones existentes |
| `tmux attach -t nombre` | Conectarse a una sesión |

### Ventanas y paneles

En esta tabla, **prefijo** significa pulsar `Ctrl + b` y soltar ambas teclas antes de pulsar la siguiente.

| Atajo | Acción |
| --- | --- |
| Prefijo → `d` | Desconectarse y dejar la sesión activa |
| Prefijo → `c` | Crear una ventana |
| Prefijo → `n` / `p` | Ir a la ventana siguiente / anterior |
| Prefijo → `,` | Renombrar la ventana actual |
| Prefijo → `%` | Dividir en paneles izquierda / derecha |
| Prefijo → `"` | Dividir en paneles arriba / abajo |
| Prefijo → flecha | Cambiar de panel |
| Prefijo → `z` | Ampliar el panel actual o restaurar su tamaño |
| Prefijo → `[` | Entrar en modo de desplazamiento; `q` para salir |
| Prefijo → `?` | Mostrar los atajos disponibles |

Para cerrar la shell del panel actual, escribimos `exit`. Al cerrar el último panel de la última ventana, termina la sesión.

Con esto ya podemos organizar varias terminales y dejar una tarea ejecutándose para retomarla más tarde.

## Recursos

- [Guía oficial de tmux](https://github.com/tmux/tmux/wiki/Getting-Started)
