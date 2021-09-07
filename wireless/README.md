<img align="right" width="200px" src="https://github.com/facundoalarcon/computernetworks/blob/main/wireless/pictures/wireless.png">

# Laboratorio wireless
En este laboratorio vamos a cubrir un tipo de ataque llamado de deauth, que por si mismo puede provocar una denegación de servicio, pero también se puede usar para obtener la clave del access point(AP) objetivo analizando los paquetes capturados en el handshake que ocurre entre el AP y el dispositivo cliente.

## Metodología del ataque

En un ataque de [deauth](https://www.aircrack-ng.org/doku.php?id=es:deauthentication), básicamente lo que se hace es intervenir en el 4-way-handshake que hace un dispositivo, que ya está autenticado en la red con el wireless access point.
<p align="center">
	<img width="400px" src="https://github.com/facundoalarcon/computernetworks/blob/main/wireless/pictures/4-way-handshake.png">
</p>

El ataque inicia cuando nosotros enviemos el deauth frame. Lo que provocará que uno o más clientes se desconecten de la red. En cuento intenten conectarse nuevamente, vamos a interceptar el tráfico del handshake. Esto se puede lograr haciendo un spoofing de las direcciones MAC de los clientese impersonando las tramas que se envían hacia el AP.

El siguiente paso, consiste en descifrar la clave, ya que si bien podemos obtenerla desde las tramas, la misma se encuentrar encriptada. Para eso podemos usar distintas herramientas que comparan hashes. En este punto entra el ataque por fuerza bruta, el cual consiste en probar todas las combinaciones posibles para encontrar la clave. Si logramos que algún hash de nuestro disccionario coincida con lo que capturamos, podemos decir que tenemos la clave. 

![deauth_diag](https://github.com/facundoalarcon/computernetworks/blob/main/wireless/pictures/deauth_diag.png)

## Setup del laboratorio

En este laboratorio vamos a usar:
- Un dispositivo AP, en este caso un smartphone, pero se puede usar como AP hogareño.
- Una máquina cliente de ese AP, en este caso mi máquina host.
- Una máquina virtual que actúa de atacante, cuenta con SO [kali-linux](https://www.kali.org/) (ya que trae preinstalado muchas tools de seguridad) y con adaptador wireless.

Cabe aclarar que, la conexión entre la VM y el host es por medio de NAT. Para que utilice una configuracíon de red interna a la del dispositivo host y simulemos estar en una red externa a la de la víctima. Por otro lado, se quitó la asociación automática del cliente del AP a otros AP, ya que sino, una vez que lo desautentiquemos del AP se conectará a ese otro AP y el ataque no será efectivo.

Este laboratorio fué realizado con tp-link Archer T2 como wireless adapter.

Para instalarlo se pueden seguir estas guias:
- [tp-link guide](https://static.tp-link.com/2018/201812/20181207/Installation%20Guide%20for%20Linux.pdf)
- [aircrack-ng endpoint installer](https://github.com/aircrack-ng/rtl8812au)
- [step-by-step guide](https://github.com/nlkguy/archer-t2u-plus-linux)
- [posible error](https://askubuntu.com/questions/1082472/how-can-i-resolve-this-problem-unable-to-locate-package-linux-image-extra-4-15)



## airmon-ng
[airmon-ng](https://www.aircrack-ng.org/doku.php?id=es:airmon-ng) nos permite trabajar con la placa en modo monitor.

### Eliminar procesos
Podemos ver todos los procesos que pueden intervenir con el handshake con `airmon-ng check`. Tenemos que deshabilitar cualquiera que pueda interferir con su captura de paquetes, escribiendo el siguiente comando `airmon-ng check kill`.

## Modo monitor
La placa de red puede actuar en varios [modos](https://lazysolutions.blogspot.com/2008/10/difference-promiscuous-vs-monitor-mode.html) uno de ellos es el modo monitor, con el mismo podremos escuchar tráfico de red inalámbrico si estar asociado necesariamente a un access point, solo opera en redes wireless. A diferencia de modo promiscuo que si necesita establecer una conexión previamente con un AP.

En primer lugar deberíamos ver la configuración actual de nuestro adaptador wireless `iw dev`. El output debería ser similar al siguiente:

```
phy#0
	Interface [INTERFACE]
		ifindex 3
		wdev 0x1
		addr [MAC-AP]
		type managed
		txpower 20.00 dBm
```

Donde [INTERFACE] es el nombre de nuestra interfaz wireless y [MAC-AP] la dirección MAC del mismo.

En la mayoría de los casos lo encontraremos en modo managed, es normal al ser un cliente de la red (si fuera un AP estaría en modo master).

Para poner la placa en modo monitor debemos ejecutar estos comandos:

```
ip link set [INTERFACE] down
iw [INTERFACE] set monitor control
ip link set [INTERFACE] up
```

Para volver a setear la placa en managed mode

```
ip link set [INTERFACE] down
iw [INTERFACE] set type managed
ip link set [INTERFACE] up
```

Siempre podemos comprobar el estado con `iw dev`.

Otra manera de revisar esto es con `iwconfig` o `ifconfig`.

Se puede hacer este mismo proceso con airmon-ng usando el comando `airmon-ng start [MAC-AP]` y `airmon-ng stop [MAC-AP]`.

Cuando se revierte de modo monitor podemos iniciar los servicios de networking. Normalmente con este comando `systemctl restart networking.service` y `systemctl restart NetworkManager`.

## Definiendo el objetivo
Con [airodump-ng](https://www.aircrack-ng.org/doku.php?id=es:airodump-ng) podemos capturar paquetes wireless 802.11, es útil para ir acumulando vectores de inicialización con el fin de intentar usarlos con aircrack-ng y obtener claves.  

Empezaremos a analizar las redes wireless cercanas ejecutando `airodump-ng [INTERFACE]`

![airodump-ng_1](https://github.com/facundoalarcon/computernetworks/blob/main/wireless/pictures/airodump-ng_1.png)

### Descripción
![airodump-ng_2](https://github.com/facundoalarcon/computernetworks/blob/main/wireless/pictures/airodump-ng_2.png)

Una de las cosas más importantes que veremos son los beacons que básicamente indican la presencia de un AP, no importq que la red esté "oculta".

## Atacando al objetivo 
Capturamos paquetes de la red objetivo con

`airodump-ng -c [CHANNEL] --bssid [MAC-AP] [INTERFACE] -w [FILENAME]`

![airodump-ng_1](https://github.com/facundoalarcon/computernetworks/blob/main/wireless/pictures/airodump-ng_3.png)

Mientras corre este proceso, en otra terminal ejecutamos [aireplay-ng](https://www.aircrack-ng.org/doku.php?id=es:aireplay-ng). Puede usarse en varios ataques, en este lab vamos a relizar el ataque deauth. La idea es efectuar el mismo y luego capturar luego los paquetes del handshake.

`aireplay-ng -0 5 -a [MAC-AP] -c [MAC-CLIENT] [MAC-AP]`

con 0 0 indicas que se van a mandar para desautenticar hasta que se cierre el programa

![aireplay-ng_1](https://github.com/facundoalarcon/computernetworks/blob/main/wireless/pictures/aireplay-ng_1.png)

Si hacemos una prueba de ping desde el cliente del AP podremos ver perdida de paquetes.

![aireplay-ng_2](https://github.com/facundoalarcon/computernetworks/blob/main/wireless/pictures/aireplay-ng_2.png)

A su vez, si observamos el estado de la conexión veremos que el cliente no se está podrá conectar a la red.

Para finalizar el ataque debemos ver EAPOL en las notas de airodump-ng, cuando aireplay-ng termine de enviar las tramas de deauth. A veces no aparece instantaneamente y debemos dejar corriendo airodump unos momentos más.

![airodump-ng__4](https://github.com/facundoalarcon/computernetworks/blob/main/wireless/pictures/airodump-ng_4.png)

Si no se obtiene la nota que diga EAPOL, se debe volver a intentar la deautenticación, en todo caso si no se obtuvo un resultado satisfactorio en el primer ataque intentar aumentar el numero de deauth frames a un valor superior o bien indicar el valor 0 para que mande estos frames indefinidamente.

`aireplay-ng -0 0 -a [MAC-AP] -c [MAC-CLIENT] [MAC-AP]`

En este caso vigilar la terminal con airodump hasta que se obtenga dicho mensaje.

Al finalizar vamos a tener archivos .cap que se pueden interpretar con wireshark.

## Analizar con wireshark 

Como dijimos anteriormente, podemos analizar los archivos generados con por las herramientas anteriores con wiresark y ver todo el intercambio que ocurre entre el cliente y el AP. Si escribimos en el filtro de la captura `eapol`, veremos los frames que se usan para el [protocolo de autenticarción](https://support.huawei.com/enterprise/en/doc/EDOC1100086527) de dicho nombre en 802.1X, cuando manda una solicitud de autenticación. En wireshark incluso nos marca cuando empieza y cuando termina este handshake.

![wireshark_1](https://github.com/facundoalarcon/computernetworks/blob/main/wireless/pictures/wireshark_1.png)

### Tipos de trama
Podremos encontrar tres tipos de trama
- **Tramas de gestión**: estos paquetes se utilizan para descubrir AP y unirse a una red inalámbrica. Algunos subtipos importantes son Beacon, Probe Request & Response, Authentication & Deauthentication, Association y Disassociation. Podremos verlas en wireshark con el comando `wlan.fc.type == 0`.
- **Tramas de control**: estos paquetes se utilizan para reconocer la transmisión exitosa y reservar el medio inalámbrico. Los paquetes de control permiten la entrega de paquetes de gestión y de datos. Los subtipos comunes son ACK, request-to-send y clear-to-send. Podremos verlas con `wlan.fc.type == 1`
- **Tramas de datos**: estos paquetes contienen datos reales y son los únicos paquetes que se reenviarán desde la red inalámbrica a la red cableada. Los tipos de frames de datos incluyen: datos y función null. Se ven con el comando `wlan.fc.type_subtype == 0x20`.

## Brute force
A continuación basandonos en una wordlist, podremos encontrar la clave, a partir de la captura obtenída.

Podemos generar una wordlist o bien buscar alguna en la web, una de las más conocidas es rockyou, que se puede encontrar en kali linux con el comando `locate rockyou` y dirigiendonos a uno de los resultados `/usr/share/wordlists/rockyou.txt.gz`, se pueden encontrar versiones más actualizadas en internet.

Una vez que tengamos la wordlist podemos usar [airckack-ng](https://www.aircrack-ng.org/doku.php?id=es:aircrack-ng) con el que podremos recuperar claves una vez que se han capturado suficientes paquetes encriptados con airodump-ng.

Usaremos el comando `aircrack-ng -w /path/to/wordlist -b [MAC-AP] [FILENAME]*.cap`

Una vez utilizado el comando si la clave hasheada coincide con alguna de nuestra wordlist obtendremos la clave.

![airodump-ng__4](https://github.com/facundoalarcon/computernetworks/blob/main/wireless/pictures/aircrack-ng_1.png)

## Prevención

No existe una medida específica en contra de ataques de autenticación, *spoiler:  poner la red oculta no es suficiente*. Además, hay que tener en cuenta que cada vez se va actualizando más la tecnología y los métodos de autenticación, por ejemplo, WPA y WPA2 ya no son considerados completamente seguros.

Entre algunas recomendaciones, podemos destacar: utilizar métodos de autenticación seguros y además claves seguras, ya que mientras sean claves fuertes será más dificil encontrar la misma por fuerza bruta. Se pueden seguir recomendaciones de, por ejemplo, [Lastpass](https://www.lastpass.com/features/password-generator), [Google](https://support.google.com/accounts/answer/32040?hl=en#zippy=) o [Microsoft](https://support.microsoft.com/en-us/windows/create-and-use-strong-passwords-c5cebb49-8c53-4f5e-2bc4-fe357ca048eb). Si la clave es lo suficientemente fuerte y no estamos bajo un ataque dirigido donde seamos víctimas de ingeniería social, seguramente el atacante no podrá encontrar la clave tan fácilmente, ya que le exigiría mucho tiempo y mucho poder de cómputo, lo que haría que este ataque no valga la pena.

Otra cosa que se puede hacer es filtrar dispositivos por dirección MAC, esto para permitir únicamente a dispositivos conocidos se autentiquen a nuestro AP. Así también, se podría limitar los intentos de autenticación de los clientes. Normalmente son features que se pueden configurar en el AP.

## Más información
- [meraki](https://documentation.meraki.com/General_Administration/Tools_and_Troubleshooting/Analyzing_Wireless_Packet_Captures) - análisis de paquetes wireless.
- [cisco community](https://community.cisco.com/t5/wireless-mobility-documents/802-11-frames-a-starter-guide-to-learn-wireless-sniffer-traces/ta-p/3110019) - wireless sniffer traces.
- [juniper-mistIa](https://www.mist.com/documentation/what-is-the-difference-between-ap-deauthentication-and-client-deauthentication/) diferencias entre tramas de deauth y disassociation.
- [hashcat](https://hashcat.net/hashcat/) - tool que puede hacer fuerza bruta con GPU, hay que convertir los archivos .cap a un formato particular que usa este programa.
- [crunch](https://tools.kali.org/password-attacks/crunch) - generador de wordlist en base a determinados parámetros.
- [cupp](https://github.com/Mebus/cupp) - generador de wordlist en base a preguntas relacionadas al objetivo.
- [airolib-ng](https://www.aircrack-ng.org/doku.php?id=airolib-ng) - tablas rainbow con hashes precalculados.
- [aircrack-lab](https://www.aircrack-ng.org/doku.php?id=cracking_wpa)
- [fluxion](https://github.com/FluxionNetwork/fluxion) - crear un "evil-twin", ataques WPA MITM.
- [macchanger](https://www.kali.org/tools/macchanger/) - tool que permite cambiar la dirección MAC a tu adaptador.
- [eapol](https://juncotic.com/wpa2-como-funciona-algoritmo-wifi/) - lectura adicional.

