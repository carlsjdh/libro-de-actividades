```
* Fecha de creación : curso 201415
* Fecha de UM       : curso 201516
* Sistema Operativo : OpenSUSE 13.2
```

#iSCSI en OpenSUSE

#1 Preparativos

Vamos a montar la práctica de iSCSI con OpenSUSE 13.2.

Necesitamos 2 MV's (Consultar [configuraciones](../../global/configuracion-aula109.md)).
* MV1: Esta MV actuará de `Initiator`.
    * Con dos interfaces de red. 
    * Una en modo puente (172.19.XX.31) 
    * y la otra en red interna (192.168.XX.31) con nombre `san`.
        * Este interfaz NO tiene gateway.
* MV2: Esta MV actuará de `Target`. 
    * Con un interfaz de red (192.168.XX.32) en modo red interna `san`. 
    * Este interfaz tiene como gateway 192.168.XX.31.
* Las IP's las pondremos todas estáticas.
* Las IP's de la red interna estarán en el rango 192.168.XX.NN/24.
Donde XX será el número correspondiente al puesto de cada alumno.

> Como vamos a necesitar acceso a los repositorios de Internet en el Target 
para instalar el software, podemos hacerlo de varias formas:
> * (a) Poner el interfaz de red temporalmente en puente, instalar y cambiar.
> * (b) Poner temporalmente un 2º interfaz puente para instalar y luego lo desactivamos.
> * (c) Activar/configurar enrutamiento en el Initiator.
>    
> **Enrutamiento en GNU/Linux**
>
> * [Enrutamiento en GNU/Linux](http://www.ite.educacion.es/formacion/materiales/85/cd/linux/m6/enrutamiento_en_linux.html)
> *  Ejemplo de script que activa el enrutamiento y el NAT:
> ```
>     // activar-enrutamiento.sh
>     echo "1" > /proc/sys/net/ipv4/ip_forward
>     iptables -A FORWARD -j ACCEPT
>     iptables -t nat -A POSTROUTING -s IP_RED_INTERNA/MASCARA_RED_INTERNA -o eth0 -j MASQUERADE
> ```
> *  Ejemplo de script que desactivara el enrutamiento:
> ```
>     // desactivar-enrutamiento.sh
>     echo "0" > /proc/sys/net/ipv4/ip_forward
> ```

#2 Target

Enlaces recomendados:
* [OpenSUSE - iSCSI Target](http://es.opensuse.org/iSCSI)
* [federicosayd - ISCSI Target en GNU/Linux Debian](https://federicosayd.wordpress.com/2007/09/11/instalando-un-target-iscsi/)

##2.1 Crear los dispositivos

* Crear los dispositivos
    * Creamos el dispositivo1 a partir de un fichero.
        * `dd if=/dev/zero of=/root/dispositivo1.img bs=1M count=500`
        * Hemos creado un fichero con tamaño 500M.
    * Creamos el dispositivo2 a partir de un disco extra.
        * Añadiremos un 2º disco de 700M a la MV Target.
        * `/dev/sdb` será nuestro dispositivo2.

##2.2 Instalar y configurar el Target

Vamos a la máquina target:
* `zypper -n in yast2-iscsi-lio-server`, Instala yast2-iscsi-lio-server.
* Yast -> Network services -> ISCSI LIO Target -> open firewall port -> 
* go to Global (terminal is SHIFT+G) -> chose authentication 
* if you need -> go to Targets (SHIFT+T) -> add target (select or not auth) -> add LUN -> select path to LUN (where you created the "vdisk") )
 
> **Desactualizado**
> * `zypper in iscsi-target`, para instalar el sofware iSCSI Target en la máquina.

##2.3 Teoría: configuración del Target

La configuración del Target se encuentra en el fichero `/etc/iet/ietd.conf`.
Contiene:
* El nombre de nuestro target
* El nombre de usuario y la contraseña para la conexión del iniciador
* El dispositivo que ofreceremos como target

###2.3.1 Nombre

El estándar iSCSI define que tanto los target como los iniciadores deben 
tener un nombre que sigue un patrón, 
el cual es el siguiente: `iqn.YYYY-MM.NOMBRE-DEL_DOMINIO_INVERTIDO:IDENTIFICADOR`. 
Donde:
* iqn es un término fijo y debe figurar al principio.
* YYYY-MM es la fecha de alta del dominio de la organización para la que estamos configurando el target.
* A continuación debe figurar el nombre del dominio invertido
* Luego de los “:”, un identificador que podemos ponerlo a nuestro gusto, y que 
puede en muchos casos brindar información del target.

Un ejemplo válido sería: `iqn.2005-02.au.com.empresa:san.200G.samba`.

Como vemos el identificador aunque es variable y personalizable, puede 
reflejar el nombre dado al target, la capacidad y el servicio donde lo usaremos.

###2.3.2 Autenticación

Si queremos que nuestro target requiera autenticación, podemos definir 
un usuario y una contraseña para que solo se conecten los iniciadores que nosotros queremos.

`IncomingUser usuario-iniciador clave-iniciador`

###2.3.3 Dispositivos

Luego debemos definir qué dispositivo ofreceremos como target. 
Debemos poner una línea como la siguiente: `Lun 0 Path=/dev/sda3,Type=fileio`

En este ejemplo el primer dispositivo que estamos ofreciendo es la 
partición /dev/sda3 del servidor. La documentación nos dice que además 
de particiones podemos usar discos enteros, volúmenes LVM y RAID, 
e incluso archivos. En cualquier caso solo hay que definir el path.

El archivo contiene muchos parámetros más de configuración, 
que en la mayoría de los casos tienen que ver con la performance del servidor. 
En nuestro ejemplo, configurando estos tres parámetros nos basta.

##2.4 Práctica: configuración del Target

En el fichero de configuración del target (`/etc/iet/ietd.conf`) 
definimos lo siguiente:

```
    iqn.2016-06.idp.SEGUNDOAPELLIDOALUMNOXXh:sanXX.1200M.test
    IncomingUser USUARIO-INICIADOR CLAVE-INICIADOR
    Lun 0 Path=/root/dispositivo1.img,Type=fileio
    Lun 1 Path=/dev/sdb,Type=fileio
``` 

> El texto anterior en mayúsculas hay que adaptarlo a la máquina de cada uno.

Una vez que hemos configurado el servidor, y que tenemos lista nuestros
dispositivos de almacenamiento, vamos a levantar el servidor.

* `systemctl start iscsitarget.service`, para iniciar el servicio.
    * Comprobar `systemctl status iscsitarget.service`

> Con el comando anterior se cargará el módulo iSCSI target en el kernel y se levantará 
el servidor ietd que es el que gestionará las peticiones de los iniciadores.

Como queremos que nuestro servicio iSCSI target inicie automáticamente al iniciar la máquina
* `systemctl enable iscsitarget.service`
    * Comprobar `systemctl is-enable iscsitarget.service`

> Ya tenemos nuestro servidor iSCSI instalado y listo para servir discos a los iniciadores de nuestra red interna. 
> Ahora necesitamos un iniciador iSCSI para que se conecte a nuestro target y empezar a usar los discos por la red.

#3 Initiator

Enlaces recomendados:
* [federicosayd - ISCSI Initiator en GNU/Linux Debian](http://federicosayd.wordpress.com/2007/09/13/montando-un-iniciador-iscsi-en-linux)

##3.1 Instalar Initiator

> * add client (cat /etc/iscsi/initiatorname.iscsi fromc client side and add inq number to yast)
> - -> Finish
> Test connection: iscsiadm -m discovery -t st -p IP.Address (default port is 3620, specify if you change it, if not leave just ip address)
> Connect from client machine: iscsiadm -m node -l ( This is a basic config without authentication )


Vamos a la máquina Iniciador
* `zypper in open-iscsi`

##3.2 Descubrir los targets

* `iscsiadm -m discovery -t sendtargets -p IP-DEL-TARGET`
* `iscsiadm -m discovery`
* `iscsiadm -m mode --targetname iqn.2016-06.idp.SEGUNDOAPELLIDOALUMNOXXh -p IP`

##3.3 Autenticación y Login

* `dmesg`, comprobamos que tenemos un nuevo disco SCSI de 1200M conectado a la MV.
* Formatear el disco, crear partición.
* Montarlo en la ruta `/home/remote_targetXX`.
* Configurar el montaje automático en cada reinicio (`/etc/fstab`).
* Guardar datos en el disco SAN.

#ANEXO

##A.1 Otros enlaces de interés

* TARGET - [Setting up iSCSI target on OpenSUSE](https://www.suse.com/documentation/sles10/book_sle_reference/data/sec_inst_system_iscsi_target.html)
* INITIATOR - [Setting up iSCSI initiator on OpenSUSE](https://www.suse.com/documentation/sles11/stor_admin/data/sec_inst_system_iscsi_initiator.html) 
* Vídeo: [EN - LINUX: ISCSI Target and Initiator Command Line configuration](https://youtu.be/5yMSxqUs4ys) 
* Vídeo: [EN - Configure iSCSI initiator (client)](https://youtu.be/8UojNONhQDo) 

##A.2 iSCSI en Debian

Enlaces de interés:
* iSCSI - [Using iSCSI (target and initiator) on Debian](https://www.howtoforge.com/using-iscsi-on-debian-lenny-initiator-and-target).
* TARGET - [Create targer iSCSI on Debian](https://wiki.debian.org/SAN/iSCSI/iscsitarget). 