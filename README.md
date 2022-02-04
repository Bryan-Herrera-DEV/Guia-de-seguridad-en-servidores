# Guía de seguridad en servidores
## Tabla de contenidos
- [El puerto SSH](#El-puerto-SSH)
- [Fundamentos del FireWall](#Fundamentos-del-FireWall)
  - [Bloquear un puerto específico](#Bloquear-un-puerto-específico)
  - [Permitir un puerto específico](#Permitir-un-puerto-específico)
  - [Bloquear una dirección IP específica](#Bloquear-una-dirección-IP-específica)
  - [Permitir una dirección IP específica](#Permitir-una-dirección-IP-específica)
  - [Bloquear todos los puertos para cualquier IP](#Bloquear-todos-los-puertos-para-cualquier-IP)
 - [Consejos sobre el cortafuegos](#Consejos-sobre-el-cortafuegos)
  - [Bloqueo de escaneo de puertos](#Bloqueo-de-escaneo-de-puertos)
  - [Bloquea paquetes ICMP](#Bloquea-paquetes-ICMP)
  - [Bruteforce Limitar las conexiones ssh](#Bruteforce-Limitar-las-conexiones-ssh)


# El puerto SSH
El puerto SSH (por defecto es el 22) es el que nos permite desde nuestro ordenador o cualquier dispositivo conectarnos a este servicio con un cliente SSH (ejemplo [Bitvise](https://www.bitvise.com/ssh-client-download) o [Putty](https://www.putty.org/)).

Es muy importante protegerlo, lo más básico sería cambiar el puerto.
Para ello seguiremos las siguientes instrucciones:
```bash
# En la terminal:
apt install nano
nano -w /etc/ssh/sshd_config
```
Este comando abrirá el archivo `/etc/ssh/sshd_config` que contiene los parámetros del servicio SSH. Iremos a la línea que contiene `# Port 22` y eliminaremos el punto (#), luego cambiaremos el número 22 por el que queremos que sea nuestro nuevo puerto SSH.  
luego reiniciaremos el servicio SSH usando:  
```bash
service sshd restart
``` 
# Fundamentos del FireWall
### Bloquear un puerto específico
Para bloquear un puerto específico para el acceso externo utilizaremos la siguiente regla iptables:  
```bash
iptables -A INPUT -p tcp --dport PORT_NUMBER_HERE -j DROP
```
### Permitir un puerto específico
Para permitir de nuevo un puerto específico bloqueado para el acceso externo utilizaremos la siguiente regla iptables:  
```bash
iptables -A INPUT -p tcp --dport PORT_NUMBER_HERE -j ACCEPT
```
### Bloquear una dirección IP específica
La regla de iptables para bloquear una dirección IP específica para el acceso externo es la siguiente 
```bash
iptables -A INPUT -s DIRECCIÓN_IP -j DROP
```
### Permitir una dirección IP específica
La regla de iptables para permitir de nuevo una dirección IP específica bloqueada para el acceso externo es la siguiente 
```bash
iptables -A INPUT -s DIRECCIÓN_IP -j ACCEPT
```
### Bloquear todos los puertos para cualquier IP
Si queremos bloquear todos los puertos para el uso externo, utilice el siguiente comando:  
```bash
iptables -P INPUT DROP
iptables -A INPUT -j REJECT
```
no olvides permitir el tráfico saliente:
```bash
iptables -A OUTPUT -j ACCEPT
iptables -A OUTPUT -o lo -j ACCEPT
```
y permitir los puertos que quieras abrir (puerto ssh/22 y puerto web/80 en la mayoría de los casos)
```bash
iptables -A INPUT -p tcp --dport 22 -j ACCEPT
iptables -A INPUT -p tcp --dport 80 -j ACCEPT
```
# Consejos sobre el cortafuegos
### Bloqueo de escaneo de puertos
Es importante si tenemos servicios con puertos públicos en nuestro servidor bloquear los escaneos de la red para evitar que los intrusos sepan por dónde atacar.  

Para ello instalaremos una herramienta que nos ayudará a falsear los puertos y que instalaremos de la siguiente manera.
```bash
# Si no tenemos git instalado, lo instalaremos:
apt install git

# luego clonaremos el repositorio usando:
git clone https://github.com/drk1wi/portspoof.git
cd portspoof

# Instalar make
apt install make

# Instalar g++
apt install g++

# e instalaremos el programa de la siguiente manera
./configure && make && sudo make install
portspoof -c portspoof.conf -s portspoof_signatures -D
```
Por último, debemos especificar qué puertos no queremos que sean redirigidos y cuáles sí.  
Para ello utilizaremos el siguiente comando:
```bash
iptables -t nat -A PREROUTING -i eth0 -p tcp -m tcp -m multiport --dports INSERT:PORT,RANGE:HERE -j REDIRECT --to-ports 4444
```
Cambie eth0 por el nombre de su adaptador de red.  

He aquí una breve explicación:  
Supongamos que quiero bloquear todos los puertos excepto el 22.  
Entonces el rango de puertos sería así  
1: 21,23: 65535  

Ahora supongamos que quiero bloquear todos los puertos excepto el 22 y el 80, entonces el rango de puertos sería así  
1: 21,23: 79,81: 65535  

Ahora como último ejemplo, supongamos que quiero bloquear todos los puertos excepto el 22, 80 y 3306, entonces el rango de puertos se vería así  
1: 21,23: 79,81: 3305,3307: 65535  

¿se entiende?  
Básicamente en la siguiente operación:  
1: 21,23: 79,81: 3305,3307: 65535  

Especificamos que  
del 1 al 21 se bloqueará (dejando libre el 22)  
del 23 al 79 se bloqueará (dejando libre el 80)  
del 81 al 3305 se bloqueará (dejando libre el 3306)  
y finalmente del 3007 al 65535 serán bloqueados.  

(el puerto 65535 es el último)  
Cuidado: elimine el espacio en los ejemplos. 
### Bloquea paquetes ICMP
Los paquetes ICMP pueden causar inestabilidad al servidor a gran escala, y es un uso innecesario de recursos, podemos deshabilitar que nuestro servidor responda a estas peticiones usando el siguiente comando:  
```bash
sudo iptables -A INPUT -p icmp --icmp-type echo-request -j REJECT
```
### Bruteforce Limitar las conexiones ssh
Recuerda que si tienes un servicio de firewall, no podrás volver a acceder a él si no abres el nuevo puerto que colocarás en lugar del 22.  

Existe un ataque a las contraseñas llamado "Fuerza Bruta", esto coincide en hacer múltiples intentos de contraseñas hasta encontrar la correcta.  
Para bloquear esto podemos configurar nuestro servidor para que sólo permita una conexión cada minuto desde una dirección IP. 

Teniendo iptables instalado y listo para usar, escribiremos en la terminal:
```bash
iptables -I INPUT -p tcp --dport 22 -i eth0 -m state --state NEW -m recent --set
iptables -I INPUT -p tcp --dport 22 -i eth0 -m state --state NEW -m recent --update --seconds 60 --hitcount 2 -j DROP
```
Si has cambiado el puerto SSH que viene por defecto, debes cambiar el parámetro "22" en el comando.  
También puede cambiar el tiempo que debe pasar entre la conexión de la misma ip, el parámetro por defecto es de 60 segundos. 
