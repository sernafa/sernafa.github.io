---
layout: post
title: "OpenSSL: operaciones útiles"
date: 2017-06-14
categories: [Linux, Security]
tags: [openssl, tls, ssl, certificates]
description: "Comandos útiles de OpenSSL para inspeccionar, convertir y verificar certificados, claves privadas y ficheros PKCS#12."
---

OpenSSL es una de esas herramientas que terminamos utilizando cada vez que necesitamos revisar un certificado, convertirlo a otro formato o comprobar por qué falla una conexión TLS.

Esta es una recopilación de las operaciones que utilizo con más frecuencia desde la terminal.

## Consultar la versión instalada

Antes de comenzar podemos comprobar la versión de OpenSSL disponible en el sistema:

```shell
openssl version
```

Para mostrar también las opciones de compilación y los directorios utilizados:

```shell
openssl version -a
```

## Inspeccionar un certificado

Para mostrar toda la información de un certificado en formato PEM:

```shell
openssl x509 -in certificado.pem -noout -text
```

La salida incluye el emisor, el sujeto, el número de serie, el periodo de validez, la clave pública y las extensiones X.509.

Si solo necesitamos un resumen podemos seleccionar los campos más importantes:

```shell
openssl x509 -in certificado.pem -noout \
  -subject -issuer -serial -dates -fingerprint
```

### Comprobar la fecha de caducidad

Podemos consultar directamente las fechas de inicio y finalización:

```shell
openssl x509 -in certificado.pem -noout -dates
```

La opción `-checkend` permite comprobar si el certificado caducará dentro de un número determinado de segundos. Para revisar los próximos 30 días:

```shell
openssl x509 -in certificado.pem -noout -checkend 2592000
```

El comando devuelve un código distinto de cero cuando el certificado caducará dentro del periodo indicado, por lo que podemos utilizarlo en un script:

```bash
#!/usr/bin/env bash

if ! openssl x509 -in certificado.pem -noout -checkend 2592000; then
  echo "El certificado caduca en menos de 30 días"
fi
```

## Trabajar con ficheros PFX o P12

Los ficheros `.pfx` y `.p12` utilizan el formato PKCS#12 y pueden contener el certificado, su clave privada y los certificados de la cadena de confianza.

### Convertir un PFX o P12 a PEM

Para extraer todo su contenido a un único fichero PEM:

```shell
openssl pkcs12 -in certificado.pfx -out certificado.pem -nodes
```

OpenSSL solicitará la contraseña del fichero de entrada. La opción `-nodes` deja la clave privada sin cifrar dentro del PEM resultante.

> El fichero generado contiene material privado. Debemos proteger sus permisos y evitar copiarlo o almacenarlo en una ubicación accesible por otros usuarios.

### Consultar el contenido sin extraerlo

Podemos mostrar la estructura del fichero PKCS#12 sin escribir certificados ni claves en disco:

```shell
openssl pkcs12 -in certificado.pfx -info -noout
```

### Extraer solamente el certificado

```shell
openssl pkcs12 -in certificado.pfx -clcerts -nokeys \
  -out certificado.pem
```

`-clcerts` selecciona el certificado asociado a la clave y `-nokeys` evita extraer la clave privada.

### Extraer solamente la clave privada

```shell
openssl pkcs12 -in certificado.pfx -nocerts -nodes \
  -out clave-privada.pem
```

En este caso la clave también se escribe sin cifrar debido a la opción `-nodes`.

### Extraer los certificados de la CA

```shell
openssl pkcs12 -in certificado.pfx -cacerts -nokeys \
  -out cadena-ca.pem
```

### Crear un fichero PFX

También podemos realizar la operación inversa y reunir el certificado, la clave privada y la cadena de confianza en un fichero PKCS#12:

```shell
openssl pkcs12 -export \
  -out certificado.pfx \
  -inkey clave-privada.pem \
  -in certificado.pem \
  -certfile cadena-ca.pem
```

Durante el proceso tendremos que definir la contraseña que protegerá el fichero resultante.

## Convertir certificados entre PEM y DER

PEM representa el certificado en Base64 y añade las líneas `BEGIN CERTIFICATE` y `END CERTIFICATE`. DER utiliza una codificación binaria.

Para convertir un certificado PEM a DER:

```shell
openssl x509 -in certificado.pem -inform PEM \
  -out certificado.der -outform DER
```

Para realizar la conversión inversa:

```shell
openssl x509 -in certificado.der -inform DER \
  -out certificado.pem -outform PEM
```

## Comprobar un certificado y su clave privada

Podemos comparar el módulo RSA del certificado con el de la clave privada:

```shell
openssl x509 -in certificado.pem -noout -modulus | openssl md5
openssl rsa -in clave-privada.pem -noout -modulus | openssl md5
```

Los dos comandos deben producir el mismo resultado. En este caso MD5 se utiliza únicamente para reducir ambos módulos a un valor fácil de comparar, no para firmar ni proteger el certificado.

También podemos verificar la integridad de la clave privada:

```shell
openssl rsa -in clave-privada.pem -check -noout
```

## Crear una clave privada y una solicitud CSR

Generamos una clave privada RSA de 2048 bits:

```shell
openssl genrsa -out dominio.key 2048
```

Con esa clave creamos una solicitud de firma de certificado o CSR:

```shell
openssl req -new -key dominio.key -out dominio.csr
```

Durante el proceso OpenSSL solicitará los datos que formarán el sujeto del certificado.

Antes de enviar la CSR a la autoridad certificadora podemos revisar su contenido y verificar su firma:

```shell
openssl req -in dominio.csr -noout -text -verify
```

## Verificar una cadena de confianza

Si disponemos del certificado de la autoridad certificadora podemos comprobar la cadena con:

```shell
openssl verify -CAfile cadena-ca.pem certificado.pem
```

Cuando la validación es correcta, OpenSSL devuelve:

```text
certificado.pem: OK
```

Si el certificado intermedio se encuentra en un fichero separado, lo indicamos mediante `-untrusted`:

```shell
openssl verify \
  -CAfile ca-raiz.pem \
  -untrusted ca-intermedia.pem \
  certificado.pem
```

## Comprobar un servicio TLS remoto

`s_client` permite abrir una conexión TLS y revisar el certificado entregado por un servidor:

```shell
openssl s_client -connect www.example.com:443 \
  -servername www.example.com
```

La opción `-servername` envía el nombre mediante SNI. Esto es necesario cuando varios sitios HTTPS comparten una misma dirección IP.

Para mostrar todos los certificados enviados por el servidor:

```shell
openssl s_client -connect www.example.com:443 \
  -servername www.example.com -showcerts
```

También podemos conectar con servicios que comienzan en texto plano y negocian TLS mediante `STARTTLS`. Por ejemplo, para un servidor SMTP:

```shell
openssl s_client -connect mail.example.com:25 \
  -servername mail.example.com -starttls smtp
```

## Resumen

Con estos comandos podemos resolver buena parte de las tareas habituales relacionadas con certificados:

- Examinar su contenido y periodo de validez.
- Convertir entre PKCS#12, PEM y DER.
- Separar certificados, claves privadas y cadenas de confianza.
- Comprobar que una clave corresponde con su certificado.
- Crear y revisar solicitudes CSR.
- Verificar una cadena de certificación.
- Diagnosticar conexiones TLS remotas.

La mayoría de las operaciones solo leen información, pero debemos prestar especial atención a las que extraen una clave privada sin cifrar. Un fichero PEM generado con `-nodes` debe tratarse con los mismos controles que cualquier otra credencial.

## Recursos

- [Documentación de OpenSSL](https://www.openssl.org/docs/)
