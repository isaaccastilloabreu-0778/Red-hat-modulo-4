# Red-hat-modulo-4
comandos del modulo 4
Practica 1
# ==========================================
# 1. Instalar dependencias y Apache
# ==========================================
sudo dnf install httpd policycoreutils-python-utils -y
sudo systemctl enable --now httpd

# ==========================================
# 2. Crear las carpetas para los dos sitios
# ==========================================
sudo mkdir -p /var/www/html/sitio1
sudo mkdir -p /var/www/html/sitio2

# ==========================================
# 3. Crear los archivos HTML de cada sitio
# ==========================================
sudo bash -c 'cat > /var/www/html/sitio1/index.html << '\''EOF'\''
<!DOCTYPE html>
<html>
<head>
    <title>Sitio 1</title>
</head>
<body>
    <h1>Hola Mundo</h1>
</body>
</html>
EOF
'

sudo bash -c 'cat > /var/www/html/sitio2/index.html << '\''EOF'\''
<!DOCTYPE html>
<html lang="es">
<head>
    <meta charset="UTF-8">
    <title>Sitio 2 - Datos Personales</title>
</head>
<body>
    <h1>Información Personal</h1>
    <p><strong>Nombre:</strong> Isaac Castillo</p>
    <p><strong>Carrera:</strong> Ingeniería</p>
    <p>Este sitio secundario está corriendo en el puerto 8080.</p>
</body>
</html>
EOF
'

# ==========================================
# 4. Configurar los Virtual Hosts (Puerto 80 y 8080)
# ==========================================
sudo bash -c 'cat > /etc/httpd/conf.d/vhosts.conf << '\''EOF'\''
Listen 8080

<VirtualHost *:80>
    DocumentRoot "/var/www/html/sitio1"
    ServerName localhost
    <Directory "/var/www/html/sitio1">
        AllowOverride All
        Require all granted
    </Directory>
</VirtualHost>

<VirtualHost *:8080>
    DocumentRoot "/var/www/html/sitio2"
    ServerName localhost
    <Directory "/var/www/html/sitio2">
        AllowOverride All
        Require all granted
    </Directory>
</VirtualHost>
EOF
'

# ==========================================
# 5. Configurar Firewall y SELinux para el puerto 8080
# ==========================================
sudo semanage port -a -t http_port_t -p tcp 8080 2>/dev/null || true
sudo firewall-cmd --permanent --add-service=http
sudo firewall-cmd --permanent --add-port=8080/tcp
sudo firewall-cmd --reload

# ==========================================
# 6. Ajustar permisos y contextos de seguridad
# ==========================================
sudo chown -R apache:apache /var/www/html
sudo chmod -R 755 /var/www/html
sudo restorecon -R /var/www/html
sudo restorecon -R /var/log/httpd

# ==========================================
# 7. Verificar sintaxis y reiniciar Apache
# ==========================================
sudo httpd -t
sudo systemctl restart httpd

###########################################################################################################################################################################################################################################################################################################################################################################
# ===== Práctica 2: Servidor de correo Postfix (RHEL) =====

# 1. Instalar paquetes
sudo dnf install -y postfix mailx cyrus-sasl-plain

# 2. Iniciar y habilitar Postfix
sudo systemctl enable --now postfix
sudo systemctl status postfix

# 3. Configurar el relay de Gmail en /etc/postfix/main.cf
sudo postconf -e "default_database_type = lmdb"
sudo postconf -e "inet_protocols = ipv4"
sudo postconf -e "relayhost = [smtp.gmail.com]:587"
sudo postconf -e "smtp_use_tls = yes"
sudo postconf -e "smtp_tls_security_level = encrypt"
sudo postconf -e "smtp_sasl_auth_enable = yes"
sudo postconf -e "smtp_sasl_security_options = noanonymous"
sudo postconf -e "smtp_sasl_password_maps = lmdb:/etc/postfix/sasl_passwd"
sudo postconf -e "smtp_tls_CAfile = /etc/pki/tls/certs/ca-bundle.crt"

# 4. Crear el archivo de credenciales (editar con nano)
sudo nano /etc/postfix/sasl_passwd
# Contenido (una sola línea):
# [smtp.gmail.com]:587 tucorreo@gmail.com:CONTRASEÑA_DE_APLICACION

# 5. Convertir a base de datos y proteger el archivo
sudo postmap /etc/postfix/sasl_passwd
sudo chmod 600 /etc/postfix/sasl_passwd /etc/postfix/sasl_passwd.lmdb

# 6. Verificar configuración y reiniciar
sudo postfix check
sudo systemctl restart postfix
postconf -n | grep -E "relayhost|smtp_|inet_protocols|default_database_type"

# 7. Abrir el firewall para SMTP
sudo firewall-cmd --permanent --add-service=smtp
sudo firewall-cmd --reload

# 8. Probar la conexión con Gmail
curl -4 -v --connect-timeout 15 telnet://smtp.gmail.com:587

# 9. Enviar el correo al maestro
echo "Tu Nombre Completo - Tu Matricula" | mail -s "MambruSeFueALaGuerra" os3conadrian@gmail.com

# 10. Verificar el envío
mailq
sudo tail -n 10 /var/log/maillog
sudo grep os3conadrian /var/log/maillog

###########################################################################################################################################################################################################################################################################################################################################################################

# ==========================================================
# PRACTICA 3: SERVIDOR DE IMPRESION CUPS EN RHEL 10
# ==========================================================

# ---------- 1. RED Y SISTEMA ----------
ip a                                              # Ver IP del servidor
ping -c 3 dl.fedoraproject.org                    # Verificar internet
sudo dnf update -y                                # Actualizar sistema

# ---------- 2. INSTALAR CUPS ----------
sudo dnf install cups cups-client -y              # Servidor de impresion y utilidades
sudo systemctl enable --now cups                  # Iniciar y habilitar al arranque
sudo systemctl status cups                        # Verificar active (running)

# ---------- 3. INSTALAR EPEL (necesario para cups-pdf) ----------
sudo subscription-manager repos --enable codeready-builder-for-rhel-10-$(arch)-rpms   # Repo de dependencias
sudo dnf install https://dl.fedoraproject.org/pub/epel/epel-release-latest-10.noarch.rpm -y   # Repo EPEL
rpm -q epel-release                               # Confirmar EPEL instalado
dnf repolist                                      # Debe aparecer epel
sudo dnf clean all                                # Limpiar cache de metadatos
sudo dnf makecache                                # Regenerar cache

# ---------- 4. INSTALAR IMPRESORA VIRTUAL PDF ----------
sudo dnf install cups-pdf -y                      # Impresora virtual que genera PDF
sudo systemctl restart cups                       # Cargar la impresora nueva
lpstat -p -d                                      # Verificar que existe Cups-PDF
# Si no aparece, crearla manualmente:
# sudo lpadmin -p Cups-PDF -E -v cups-pdf:/ -m everywhere

# ---------- 5. CONFIGURAR ACCESO REMOTO ----------
sudo cp /etc/cups/cupsd.conf /etc/cups/cupsd.conf.bak   # Respaldo
sudo vi /etc/cups/cupsd.conf
#   Cambiar:  Listen localhost:631   ->   Port 631
#   Asegurar: Browsing On
#   Agregar "Allow @LOCAL" debajo de "Order allow,deny" en:
#       <Location />
#       <Location /admin>
#       <Location /admin/conf>
sudo grep -n -B2 "Allow @LOCAL" /etc/cups/cupsd.conf   # Verificar los 3 bloques
sudo cupsd -t                                     # Validar sintaxis
sudo systemctl restart cups                       # Aplicar cambios
sudo ss -tlnp | grep 631                          # Debe escuchar en *:631 o 0.0.0.0:631

# ---------- 6. COMPARTIR LA IMPRESORA ----------
sudo cupsctl --share-printers --remote-admin      # Habilitar compartir y admin remota
sudo lpadmin -p Cups-PDF -o printer-is-shared=true   # Marcar impresora como compartida
sudo systemctl restart cups

# ---------- 7. FIREWALL ----------
sudo firewall-cmd --permanent --add-service=ipp   # Permitir servicio IPP
sudo firewall-cmd --permanent --add-port=631/tcp  # Permitir puerto 631
sudo firewall-cmd --reload                        # Aplicar reglas
sudo firewall-cmd --list-all                      # Verificar reglas

# ---------- 8. PRUEBA LOCAL EN EL SERVIDOR ----------
echo "Prueba CUPS" | lp -d Cups-PDF               # Enviar trabajo de prueba
lpstat -t                                         # Estado completo de CUPS
ls -l /var/spool/cups-pdf/                        # PDF generados
ls -l /var/spool/cups-pdf/ANONYMOUS/              # PDF de clientes remotos

# ---------- 9. CLIENTE WINDOWS ----------
# Navegador:  http://IP_DEL_SERVIDOR:631
# Configuracion > Impresoras y escaneres > Agregar > Agregar manualmente
# > "Seleccionar una impresora compartida por nombre":
#     http://IP_DEL_SERVIDOR:631/printers/Cups-PDF
# Driver: Microsoft PS Class Driver (o Generic / Text Only)
# Imprimir un documento de Word y verificar el PDF en el servidor

# ---------- 10. DIAGNOSTICO ----------
sudo tail -f /var/log/cups/error_log              # Ver errores en tiempo real
getenforce                                        # Estado de SELinux
sudo ausearch -m avc -ts recent                   # Errores de SELinux
