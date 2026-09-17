---
layout: post
title: "Utilidades web desde la terminal: ifconfig.co y wttr.in"
date: 2017-07-06
categories: [Linux, Networking]
tags: [linux, terminal, curl, network, weather]
description: "Consulta la IP pública y la previsión meteorológica desde la terminal utilizando ifconfig.co y wttr.in."
---

No siempre necesitamos abrir un navegador para realizar una consulta sencilla. Algunos servicios web están diseñados para responder directamente en texto, lo que permite utilizarlos desde la terminal y también integrarlos en nuestros scripts.

Dos ejemplos especialmente útiles son [ifconfig.co](https://ifconfig.co), para consultar nuestra dirección IP pública, y [wttr.in](https://wttr.in), para obtener la previsión meteorológica.

## Consultar la IP pública con ifconfig.co

La consulta más sencilla devuelve únicamente la dirección IP pública desde la que estamos accediendo:

```shell
$ curl https://ifconfig.co/ip
203.0.113.10
```

También podemos realizar la misma consulta con otras herramientas:

```shell
$ http -b https://ifconfig.co/ip
203.0.113.10

$ wget -qO- https://ifconfig.co/ip
203.0.113.10

$ fetch -qo- https://ifconfig.co/ip
203.0.113.10
```

### Forzar IPv4 o IPv6

Cuando disponemos de conectividad mediante ambos protocolos, podemos indicar a `curl` cuál queremos utilizar:

```shell
$ curl -4 https://ifconfig.co/ip
203.0.113.10

$ curl -6 https://ifconfig.co/ip
2001:db8::10
```

Si la conexión solicitada no está disponible, el comando devolverá un error. Esto también nos sirve como una comprobación rápida de la conectividad IPv4 o IPv6 del sistema.

### Obtener la respuesta en JSON

El endpoint `/json` devuelve, además de la IP, información sobre la red y la localización asociada:

```shell
$ curl https://ifconfig.co/json
```

Salida abreviada:

```json
{
  "ip": "203.0.113.10",
  "country": "Spain",
  "country_iso": "ES",
  "city": "Valencia",
  "time_zone": "Europe/Madrid",
  "asn": "AS64500"
}
```

Con `jq` podemos seleccionar solamente el dato que necesitamos:

```shell
$ curl -s https://ifconfig.co/json | jq -r '.ip'
203.0.113.10
```

## Consultar el tiempo con wttr.in

`wttr.in` muestra la previsión directamente en la terminal. Si no especificamos ninguna ubicación, intenta determinarla a partir de nuestra dirección IP:

```shell
curl https://wttr.in
```

También podemos indicar una ciudad concreta:

```shell
curl https://wttr.in/Valencia
```

Para ver las opciones disponibles:

```shell
curl https://wttr.in/:help
```

Si solo queremos las condiciones actuales, sin la previsión de los próximos días, utilizamos la opción `0`:

```shell
curl "https://wttr.in/Valencia?0"
```

### Salida compacta

Para scripts, barras de estado o mensajes podemos solicitar una respuesta de una sola línea:

```shell
$ curl -s "https://wttr.in/Valencia?format=3"
Valencia: ☀️ +24°C
```

También podemos definir el formato exacto. En este ejemplo mostramos la ubicación, el estado, la temperatura y la humedad:

```shell
$ curl -s "https://wttr.in/Valencia?format=%l:+%c+%t,+humedad+%h"
Valencia: ☀️ +24°C, humedad 48%
```

### Obtener la previsión en JSON

El formato `j1` devuelve información detallada en JSON:

```shell
curl -s "https://wttr.in/Valencia?format=j1"
```

Podemos combinarlo con `jq` para extraer, por ejemplo, la temperatura actual:

```shell
$ curl -s "https://wttr.in/Valencia?format=j1" |
  jq -r '.current_condition[0].temp_C'
24
```

La respuesta incluye las condiciones actuales, la ubicación detectada y la previsión de los siguientes días.

## Utilizarlos en scripts

Al devolver texto plano, ambos servicios pueden integrarse fácilmente en un script. Por ejemplo, podemos mostrar un pequeño resumen al iniciar una sesión:

```bash
#!/usr/bin/env bash

public_ip="$(curl -fsS https://ifconfig.co/ip)"
weather="$(curl -fsS "https://wttr.in/Valencia?format=3")"

printf 'IP pública: %s\n' "$public_ip"
printf 'Tiempo: %s\n' "$weather"
```

La salida sería similar a esta:

```text
IP pública: 203.0.113.10
Tiempo: Valencia: ☀️ +24°C
```

También podemos utilizar la IP pública en una comprobación automática y actuar solamente cuando cambie:

```bash
#!/usr/bin/env bash

current_ip="$(curl -fsS https://ifconfig.co/ip)"
saved_ip="$(cat /tmp/last-public-ip 2>/dev/null)"

if [ "$current_ip" != "$saved_ip" ]; then
  printf 'La IP pública ha cambiado: %s\n' "$current_ip"
  printf '%s\n' "$current_ip" > /tmp/last-public-ip
fi
```

Conviene controlar los errores y no realizar consultas innecesarias cuando integremos estos servicios en automatizaciones periódicas. En el caso de `ifconfig.co`, su uso automatizado debe respetar el límite indicado por el propio servicio.

## Recursos

- [ifconfig.co](https://ifconfig.co)
- [Documentación de wttr.in](https://github.com/chubin/wttr.in)
