**IMPLEMENTACIÓN DE UN FIREWALL CON DMZ EN GNS3**

Un firewall es un sistema o conjunto de sistemas colocado en el punto de entrada entre una red privada y una red pública, de tal manera que todos los paquetes tienen que pasar a través de él. Pueden ser tanto programas de software como dispositivos de hardware.

Una zona desmilitarizada (DMZ) es una red aislada que se encuentra dentro de la red interna de una organización. En ella se encuentran ubicados exclusivamente todos los recursos de la empresa que deben ser accesibles desde Internet, como el servidor web o de correo. Aunque físicamente los servidores dentro de una DMZ se encuentran en la misma empresa, no están conectados directamente con los equipos de la red local.

Por lo general, una DMZ permite las conexiones procedentes tanto de Internet, como de la red local de la empresa donde están los equipos de los trabajadores, pero las conexiones que van desde la DMZ a la red local, no están permitidas. Esto se debe a que los servidores que son accesibles desde Internet son más susceptibles a sufrir un ataque que pueda comprometer su seguridad. Si un intruso comprometiera un servidor de la zona desmilitarizada, tendría mucho más complicado acceder a la red local de la organización, ya que las conexiones procedentes de la DMZ se encuentran bloqueadas.

Se implementaron las reglas IPTables en la siguiente topología:

![topologia-firewall](https://github.com/user-attachments/assets/2d8354b6-18f4-49d5-998f-162865cb547c)

Lo que se hace primero es un flush de las reglas, esto es el borrado de las reglas aplicadas actualmente.

    iptables -F
    
    iptables -X
    
    iptables -Z

Si bien lo óptimo es establecer una política por defecto DROP y a partir de allí permitir lo que sea necesario, para este ejemplo, se establece la política por defecto ACCEPT. Como no indicamos en qué tabla, por defecto es FILTER.

    iptables -P INPUT ACCEPT
    
    iptables -P OUTPUT ACCEPT
    
    iptables -P FORWARD ACCEPT 

Al firewall tenemos acceso desde la red local.

    iptables -A INPUT -s 192.168.1.1/24 -i eth1 -j ACCEPT

Se activa el bit de forwarding. Con esto se permite hacer forward de paquetes en el firewall, es decir, que otras máquinas puedan salir a través del firewall.

    echo 1 > /proc/sys/net/ipv4/ip_forward

Se permite el acceso a internet desde la DMZ.

    iptables -A FORWARD -s 192.168.2.2/24 -d 192.168.0.2/24 -j ACCEPT

Se cierra el acceso de internet al firewall.

    iptables -A INPUT -s 192.168.0.2/24 -d 192.168.2.1/24 -j REJECT

Se cierra el acceso de la DMZ al firewall.

    iptables -A INPUT -s 192.168.2.2/24 -i eth2 -j REJECT

Se permite el tráfico entre Firewall y LAN.

    iptables -A INPUT -i eth1 -d 192.168.1.1/24 -p icmp -j ACCEPT
    iptables -A OUTPUT -o eth1 -s 192.168.1.1/24 -p icmp -j ACCEPT

Se permite el tráfico entre Firewall y la DMZ.

    iptables -A INPUT -i eth2 -d 192.168.2.1/24 -p icmp -j ACCEPT
    iptables -A OUTPUT -o eth2 -s 192.168.2.1/24 -p icmp -j ACCEPT

Se permite la solicitud de ping desde la LAN a la DMZ y el reply correspondiente.

    iptables -A FORWARD -i eth1 -d 192.168.2.0/24 -p icmp --icmp-type echo-request -j ACCEPT
    iptables -A FORWARD -o eth1 -s 192.168.2.0/24 -p icmp --icmp-type echo-reply -j ACCEPT

Se permite el tráfico desde el firewall y el exterior.

    iptables -A INPUT -i eth0 -p icmp --icmp-type echo-reply -j ACCEPT
    iptables -A OUTPUT -o eth0 -p icmp --icmp-type echo-request -j ACCEPT

Se permite el tráfico desde la DMZ y el exterior.

    iptables -I FORWARD -i eth2 -p icmp --icmp-type echo-request -j ACCEPT
    iptables -I FORWARD -o eth2 -p icmp --icmp-type echo-reply -j ACCEPT

Se cierra el acceso de internet a la DMZ.

    iptables -A FORWARD -i eth0 -o eth2 -j REJECT

Se permite el acceso de la LAN a internet.

    iptables -I FORWARD -s 192.168.1.1/24 -d 192.168.0.2/24 -j ACCEPT

Se cierra el acceso de internet a la LAN.

    iptables -A FORWARD -i eth0 -o eth1 -j REJECT

Se cierran los accesos indeseados del exterior  (0.0.0.0/0 significa cualquier red).

    iptables -A INPUT -s 0.0.0.0/0 -p tcp --dport 1:1024 -j DROP
    iptables -A INPUT -s 0.0.0.0/0 -p udp --dport 1:1024 -j DROP

Se cierra el acceso de la DMZ a la LAN.

    iptables -I FORWARD -s 192.168.2.2/24 -d 192.168.1.1/24 -j REJECT

Se permite el tráfico desde la LAN y el exterior.

    iptables -I FORWARD -i eth1 -p icmp --icmp-type echo-request -j ACCEPT
    iptables -I FORWARD -o eth1 -p icmp --icmp-type echo-reply -j ACCEPT

Se permite acceder al puerto 23 (Telnet) desde la LAN.

    iptables -I FORWARD -p tcp -i eth1 --dport 23 -m conntrack --ctstate NEW,ESTABLISHED -j ACCEPT
    iptables -I FORWARD -p tcp -o eth1 --sport 23 -m conntrack --ctstate ESTABLISHED -j ACCEPT

Se permite acceder al puerto 23 (Telnet) desde el exterior.

    iptables -I FORWARD -p tcp -i eth0 --dport 23 -m conntrack --ctstate NEW,ESTABLISHED -j ACCEPT
    iptables -I FORWARD -p tcp -o eth0 --sport 23 -m conntrack --ctstate ESTABLISHED -j ACCEPT

Se permite acceder al puerto 23 (Telnet) desde la DMZ.

    iptables -I FORWARD -p tcp -i eth2 --dport 23 -m conntrack --ctstate NEW,ESTABLISHED -j ACCEPT
    iptables -I FORWARD -p tcp -o eth2 --sport 23 -m conntrack --ctstate ESTABLISHED -j ACCEPT

Se permite el acceso desde el exterior a los puertos 80, 53 y 443 de la DMZ.

    iptables -A FORWARD -d 192.168.2.2/24 -p tcp --dport 80 -j ACCEPT
    iptables -A FORWARD -d 192.168.2.2/24 -p udp --dport 53 -j ACCEPT
    iptables -A FORWARD -d 192.168.2.2/24 -p tcp --dport 443 -j ACCEPT

Se permite el acceso de la red local a internet (puerto 80, 53 y 443).

    iptables -I FORWARD -d 192.168.1.0/24 -p tcp -o eth1 --sport 80 -m conntrack --ctstate NEW,ESTABLISHED -j ACCEPT
    iptables -A FORWARD -d 192.168.1.0/24 -p udp --dport 53 -j ACCEPT
    iptables -A FORWARD -d 192.168.1.0/24 -p tcp --dport 443 -j ACCEPT

En estas reglas se permitió el acceso a algunos puertos pues son necesarios para navegar por internet o para la resolución de nombres, a continuación se explica con mayor detalle:
- Puerto TCP/23: Telnet sirve para establecer conexión remotamente con otro equipo por la línea de comandos y controlarlo. Si bien es un protocolo no seguro ya que la autenticación y todo el tráfico de datos se envía sin cifrar, es utilizado para realizar la demostración del funcionamiento de las reglas.
- Puerto UDP/53: es utilizado para servicios DNS (sistema de nombres de dominio), este protocolo permite utilizar tanto TCP como UDP para la comunicación con los servidores DNS. 
- Puerto TCP/80: este puerto es el que se usa para la navegación web de forma no segura HTTP.
- Puerto TCP/443: este puerto es también para la navegación web, pero en este caso usa el protocolo HTTPS que es seguro.
