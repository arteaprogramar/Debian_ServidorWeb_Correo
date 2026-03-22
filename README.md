# Servidor Web y Correo con Debian

Este proyecto documenta la creación de una **intranet sencilla** utilizando una máquina virtual con **Debian 13** como servidor central. El objetivo es montar un entorno de red local funcional que incluya:

- **Servidor web** accesible por nombre (arteaprogramar.local)
- **Servidor DNS** para resolución de nombres interna
- **Servidor de correo** con protocolos SMTP (envío) y POP3 (recepción)

Todo el entorno funciona dentro de una red local, sin necesidad de dominio público, permitiendo que equipos conectados (como Windows) accedan a los servicios mediante nombres locales y no solo por IP. Es una solución ideal para prácticas de administración de sistemas, entornos de prueba o pequeñas redes domésticas con servicios integrados.

---

## Descripción del Proyecto
Configuración de un servidor completo en Debian que incluye:
- Servidor Web (Nginx)
- Servidor DNS local (Dnsmasq)
- Servidor de Correo (Sendmail + Popa3d)
- Acceso al servidor desde dominio local (http://arteaprogramar.local)
- Gestión de correo usando Thunderbird

## Configuración del Entorno
- **Sistema Operativo**: Debian 13
- **IP del Servidor**: 192.168.1.248
- **IP del Host Windows**: 192.168.1.253
- **Gateway**: 192.168.1.1

---

## Instalación Inicial

### 1. Actualizar el sistema
```bash
ping -c 4 google.com
sudo apt update && sudo apt upgrade -y
```

### 2. Instalar herramientas básicas
```bash
sudo apt install -y nano wget curl net-tools dnsutils debconf-utils
```

### 3. Configurar PATH para root
```bash
nano /root/.bashrc
# Agregar: export PATH=$PATH:/usr/sbin
source /root/.bashrc
dpkg-reconfigure --version
```

---

## Configuración de Red

### 1. Identificar interfaz de red

De este comando es importante identificar el nombre de la interfaz donde tenemos acceso a internet

```bash
ip a

enp0s3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 08:00:27:f8:16:e4 brd ff:ff:ff:ff:ff:ff
    altname enx080027f816e4
    inet 192.168.1.248/24 brd 192.168.1.255 scope global dynamic noprefixroute enp0s3
       valid_lft 86372sec preferred_lft 75572sec
    inet6 fe80::d9c:7c27:1f79:eb58/64 scope link valid_lft forever preferred_lft forever

```



Es importante recordar el nombre de nuestra interfaz `enp0s3`


### 2. Configurar IP estática
```bash
sudo nano /etc/network/interfaces
```

```ini
auto lo
iface lo inet loopback

auto enp0s3
iface enp0s3 inet static
    address 192.168.1.248
    netmask 255.255.255.0
    gateway 192.168.1.1
    dns-nameservers 8.8.8.8 8.8.4.4
```

### 3. Reiniciar red
```bash
sudo systemctl restart networking
```

### 4. Configurar hostname
```bash
sudo nano /etc/hostname
# Contenido: arteaprogramar

sudo nano /etc/hosts
```

```ini
127.0.0.1       localhost
127.0.1.1       arteaprogramar

192.168.1.248   arteaprogramar arteaprogramar.local
192.168.1.248   mail.arteaprogramar.local
```

### 5. Verificar
```bash
ping -c 2 arteaprogramar
```

---

## Servidor Web (Nginx)

### 1. Instalar Nginx
```bash
sudo apt install -y nginx
```

### 2. Crear directorio del sitio
```bash
sudo mkdir -p /var/www/arteaprogramar
sudo chmod 777 /var/www/arteaprogramar
```

### 3. Crear página web
```bash
sudo nano /var/www/arteaprogramar/index.html
```

```html
<!DOCTYPE html>
<html>
<head>
    <title>Bienvenido a Arte a Programar</title>
</head>
<body>
    <h1>Servidor Web arteaprogramar</h1>
    <p>Este es el servidor web funcionando correctamente.</p>
</body>
</html>
```

### 4. Configurar sitio en Nginx
```bash
sudo nano /etc/nginx/sites-available/arteaprogramar
```

```nginx
server {
    listen 80;
    listen [::]:80;
    
    server_name arteaprogramar arteaprogramar.local;
    
    root /var/www/arteaprogramar;
    index index.html;
    
    location / {
        try_files $uri $uri/ =404;
    }
}
```

### 5. Habilitar sitio
```bash
sudo ln -s /etc/nginx/sites-available/arteaprogramar /etc/nginx/sites-enabled/
sudo rm /etc/nginx/sites-enabled/default
sudo systemctl reload nginx
```

### 6. Probar
```bash
curl http://arteaprogramar
```

### 7. Configurar firewall
```bash
sudo apt install -y ufw
sudo ufw allow 80/tcp
sudo ufw enable
```

---

## Configuración de DNS Local (Dnsmasq)

### 1. Instalar dnsmasq
```bash
sudo apt install -y dnsmasq
sudo systemctl status dnsmasq
sudo systemctl stop dnsmasq
```

### 2. Configurar dnsmasq
```bash
sudo nano /etc/dnsmasq.conf
```

```ini
interface=*
bind-interfaces
port=53
no-hosts

address=/arteaprogramar/192.168.1.248
address=/mail.arteaprogramar.local/192.168.1.248

server=8.8.8.8
server=8.8.4.4

domain=arteaprogramar.local
cache-size=1000
log-queries
log-facility=/var/log/dnsmasq.log
```

### 3. Crear archivo de log
```bash
sudo touch /var/log/dnsmasq.log
sudo chown dnsmasq:dnsmasq /var/log/dnsmasq.log
```

### 4. Iniciar dnsmasq
```bash
sudo systemctl start dnsmasq
sudo systemctl status dnsmasq
sudo netstat -tlnp | grep 53
```

### 5. Probar DNS local
```bash
nslookup arteaprogramar 127.0.0.1
nslookup google.com 127.0.0.1
```

### 6. Configurar Debian para usar su propio DNS
```bash
sudo cp /etc/resolv.conf /etc/resolv.conf.bak
sudo chattr -i /etc/resolv.conf 2>/dev/null
echo "nameserver 127.0.0.1" | sudo tee /etc/resolv.conf
sudo chattr +i /etc/resolv.conf
sudo systemctl restart dnsmasq
```

---

## Servidor de Correo (Sendmail + Popa3d)

### 1. Instalar servicios de correo
```bash
apt install -y sendmail popa3d mailutils bsd-mailx
```

### 2. Crear usuarios de correo
```bash
adduser arteaprogramar
# Contraseña: 12345678

adduser test
# Contraseña: test123
```

### 3. Iniciar servicios
```bash
systemctl start sendmail
systemctl enable sendmail
systemctl start popa3d
systemctl enable popa3d
```

### 4. Verificar servicios
```bash
systemctl status sendmail
systemctl status popa3d
netstat -tlnp | grep 110
```

### 5. Crear buzones de correo
```bash
touch /var/mail/arteaprogramar
chown arteaprogramar:mail /var/mail/arteaprogramar

touch /var/mail/test
chown test:mail /var/mail/test
```

### 6. Configurar firewall para correo
```bash
ufw allow 110/tcp
ufw allow 25/tcp
```

---

## Configuración Avanzada de Sendmail

### 1. Instalar editor gráfico
```bash
apt install -y mousepad
```

### 2. Editar configuración de sendmail
```bash
mousepad /etc/mail/sendmail.mc
```

**Cambiar líneas:**

Buscar:
```
DAEMON_OPTIONS(`Family=inet,  Name=MTA-v4, Port=smtp, Addr=127.0.0.1')dnl
```

Cambiar a:
```
DAEMON_OPTIONS(`Family=inet,  Name=MTA-v4, Port=smtp, Addr=0.0.0.0')dnl
```

Buscar:
```
DAEMON_OPTIONS(`Family=inet,  Name=MSP-v4, Port=submission, M=Ea, Addr=127.0.0.1')dnl
```

Cambiar a:
```
DAEMON_OPTIONS(`Family=inet,  Name=MSP-v4, Port=submission, M=Ea, Addr=0.0.0.0')dnl
```

### 3. Recompilar y reiniciar
```bash
cd /etc/mail
make
systemctl restart sendmail
```

### 4. Verificar que SMTP escucha en todas las interfaces
```bash
netstat -tlnp | grep :25
# Debe mostrar: 0.0.0.0:25
```

---

## Configuración en Windows

### 1. Configurar DNS manual en Windows
- **Configuración** → **Red e Internet** → **WiFi**
- **Configuración de IP** → **Editar** → **Manual**
- Activar **IPv4**
- **DNS preferido**: `192.168.1.248`
- **DNS alternativo**: `8.8.8.8`

### 2. Configurar firewall de Windows para permitir ping
```powershell
# Ejecutar como Administrador
New-NetFirewallRule -DisplayName "Allow ICMPv4" -Protocol ICMPv4 -IcmpType 8 -Action Allow -Direction Inbound
```

### 3. Probar conectividad
```cmd
ping 192.168.1.248
telnet 192.168.1.248 110
telnet 192.168.1.248 25
```

---

## Configuración de Thunderbird

### 1. Instalar Thunderbird (opcional en Debian)
```bash
sudo apt install thunderbird
```

### 2. Configuración de cuenta

| Campo | Valor |
|-------|-------|
| **Nombre** | Arteaprogramar |
| **Correo electrónico** | arteaprogramar@arteaprogramar.local |
| **Contraseña** | 12345678 |

**Servidor de entrada (POP3)**:
- **Servidor**: `192.168.1.248`
- **Puerto**: `110`
- **Conexión**: Ninguna
- **Autenticación**: Contraseña normal

**Servidor de salida (SMTP)**:
- **Servidor**: `192.168.1.248`
- **Puerto**: `25`
- **Conexión**: Ninguna
- **Autenticación**: Sin autenticación

---

## 🧪 Pruebas de Funcionamiento

### 1. Probar envío de correo local
```bash
echo "Prueba local de correo" | mail -s "Test Local" arteaprogramar@arteaprogramar.local
```

### 2. Verificar recepción
```bash
cat /var/mail/arteaprogramar
```

### 3. Probar POP3
```bash
telnet localhost 110
```

Comandos:
```
USER arteaprogramar
PASS 12345678
LIST
QUIT
```

---

## Resumen de Servicios

| Servicio | Puerto | Estado |
|----------|--------|--------|
| Web (Nginx) | 80 | ✅ Activo |
| DNS (Dnsmasq) | 53 | ✅ Activo |
| SMTP (Sendmail) | 25 | ✅ Activo |
| POP3 (Popa3d) | 110 | ✅ Activo |

---

## Usuarios de Correo

| Usuario | Contraseña | Correo |
|---------|-----------|--------|
| arteaprogramar | 12345678 | arteaprogramar@arteaprogramar.local |
| test | 12345678 | test@arteaprogramar.local |

---

## Comandos Útiles

### Verificar servicios
```bash
systemctl status nginx sendmail popa3d dnsmasq
```

### Ver puertos activos
```bash
netstat -tlnp | grep -E ":(80|25|110|53)"
```

### Ver logs de correo
```bash
tail -f /var/log/mail.log
```

### Ver cola de correo
```bash
mailq
```

### Forzar envío de correos pendientes
```bash
sendmail -q -v
```

### Ver configuración de red
```bash
ip addr show
ip route show
```

---

## Solución de Problemas Comunes

### SMTP no escucha en 0.0.0.0
```bash
# Verificar configuración
grep "DaemonPortOptions" /etc/mail/sendmail.cf
# Debe mostrar: Addr=0.0.0.0

# Editar directamente si es necesario
mousepad /etc/mail/sendmail.cf
# Cambiar: O DaemonPortOptions=Port=smtp,Addr=0.0.0.0, Name=MTA

systemctl restart sendmail
```

### POP3 no acepta conexiones externas
```bash
# Verificar que escucha en 0.0.0.0
netstat -tlnp | grep 110

# Configurar popa3d
nano /etc/default/popa3d
# Agregar: OPTIONS="-a -l"

systemctl restart popa3d
```

### DNS no resuelve nombres locales
```bash
# Verificar dnsmasq
systemctl status dnsmasq
tail -f /var/log/dnsmasq.log

# Probar resolución
nslookup arteaprogramar 127.0.0.1
```

---

## Notas Finales
- Este servidor está configurado para funcionar en **intranet** (red local)
- No se utiliza SSL/TLS por simplicidad (solo para pruebas)
- Los correos se almacenan en `/var/mail/[usuario]`
- La resolución DNS local se maneja con dnsmasq

---

## Referencias
- [Documentación de Nginx](https://nginx.org/en/docs/)
- [Documentación de Sendmail](https://www.sendmail.org/)
- [Documentación de Dnsmasq](http://www.thekelleys.org.uk/dnsmasq/doc.html)

---

*Configuración realizada en Debian 13 - Marzo 2026*
