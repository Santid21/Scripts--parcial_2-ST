# Scripts--parcial_2-ST

# ✒️ Nicolás Cuellar Castrellón


# ✒️ Santiago Duque Valencia

## 📂 Infraestructura

El `Vagrantfile` define las siguientes máquinas virtuales:

| Máquina         | Hostname     | IP privada     | Rol / Servicios principales |
|-----------------|--------------|----------------|-----------------------------|
| **fw**          | fw-telema    | 192.168.50.1   | Firewall con UFW, NAT hacia Internet, DNAT para FTP y DNS |
| **ftp**         | ftp-telema   | 192.168.50.2   | Servidor FTP seguro (vsftpd + TLS), usuario local `ftpuser` |
| **dns-slave**   | dns-slave    | 192.168.50.3   | Servidor DNS esclavo (BIND9), replica desde master |
| **dns-master**  | dns-master   | 192.168.50.4   | Servidor DNS maestro (BIND9), zona `midominio.test` |
| **client**      | client-linux | 192.168.50.10  | Cliente Linux con herramientas de red (dig, tcpdump, tshark), DNS over TLS habilitado |

---

## ⚙️ Configuración global

- **Box base**: `ubuntu/jammy64` (Ubuntu 22.04 LTS).
- **Proveedor**: VirtualBox.
- **Recursos por VM**:  
  - RAM: `512 MB`  
  - CPU: `1`  

---

## 🔧 Detalles de servicios

### 🔥 Firewall (`fw`)
- Interfaz **NAT** (salida a Internet) + **LAN interna** (`192.168.50.1`).
- NAT habilitado con **UFW** (`MASQUERADE`).
- DNAT de puertos hacia servidores internos:
  - FTP (`21`, `30000-30100`) → `192.168.50.2`.
  - DNS (`53/tcp`, `53/udp`) → `192.168.50.4`.

### 📁 Servidor FTP (`ftp`)
- Servicio **vsftpd** configurado con:
  - TLS (certificado autofirmado).
  - Usuario: `ftpuser` / Contraseña: `ftpuser123`.
  - Modo pasivo (`30000-30100`).
  - La IP publicada para modo pasivo es la del **firewall** (`192.168.50.1`).
- Acceso directo bloqueado: solo se permite tráfico a través del firewall.

### 🌐 DNS Slave (`dns-slave`)
- BIND9 configurado como **esclavo** de `dns-master`.
- Replica la zona `midominio.test`.
- Solo acepta tráfico desde el firewall y el maestro.

### 🌐 DNS Master (`dns-master`)
- BIND9 configurado como **maestro** de la zona `midominio.test`.
- Zona definida con:
  - `ns1.midominio.test` → `192.168.50.4`
  - `ns2.midominio.test` → `192.168.50.3`
  - `www.midominio.test` → `192.168.50.100`
  - `ftp.midominio.test` → `192.168.50.1`
- Replica hacia `dns-slave`.

### 💻 Cliente (`client`)
- Herramientas instaladas: `dnsutils`, `tcpdump`, `tshark`.
- Configurado para:
  - Resolver con DNS sobre TLS (`DoT`) usando Cloudflare y Quad9.
  - Ruta por defecto vía el firewall (`192.168.50.1`).
  - Probar consultas DNS a través del firewall:  


# **FTP Seguro + Firewall**
## 🖥️ Máquinas involucradas

fw → Firewall con NAT y reglas UFW.

ftp → Servidor FTP seguro (vsftpd + TLS).

client → Cliente Linux para pruebas de conexión.

## 🔥 Firewall
Verificar reglas cargadas:
sudo ufw status verbose

Comprobar NAT hacia Internet:
curl ifconfig.me

Ver reglas de iptables:
sudo iptables -t nat -L -n -v

## 📁 FTP Seguro
Acceder desde cliente:
vagrant ssh client

ftp 192.168.50.1

Usuario: ftpuser
Contraseña: ftpuser123

Conexión TLS explícita:

openssl s_client -connect 192.168.50.1:21 -starttls ftp

Subir y descargar archivos:
echo "Hola FTP" > test.txt
put test.txt
get test.txt

# **DNS Maestro/Esclavo + Firewall** 
## 🖥️ Máquinas involucradas

fw → Firewall que redirige consultas DNS.

dns-master → BIND9 como servidor maestro.

dns-slave → BIND9 como servidor esclavo.

client → Cliente para consultas dig.

## 🌐 Consultas DNS

Desde cliente al firewall:

dig @192.168.50.1 www.midominio.test

dig @192.168.50.1 ftp.midominio.test

Validar zona en maestro:

sudo named-checkzone midominio.test /etc/bind/db.midominio.test

Consultas al esclavo:

dig @192.168.50.3 www.midominio.test

# **DNS sobre TLS**
## 🖥️ Máquinas involucradas

fw → Redirige consultas al DNS.

client → Configurado para usar DNS sobre TLS (DoT).

## 🔧 Verificación

Estado del resolvedor:

resolvectl status

Captura tráfico DNS sin cifrado:

sudo tcpdump -i eth1 port 53 -n

Captura DNS cifrado (DoT, puerto 853):

sudo tcpdump -i eth1 port 853 -n

    dig @192.168.50.1 www.midominio.test
    ```
