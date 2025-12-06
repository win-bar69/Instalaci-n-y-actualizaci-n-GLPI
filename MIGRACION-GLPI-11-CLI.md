## 🔍 Paso 0: Verificación y Credenciales

```bash
# VERIFICAR CREDENCIALES MYSQL (guarda estos valores)
sudo cat /var/www/glpi/config/config_db.php | grep -E "DB_USER|DB_NAME|DB_PASSWORD"

# VERIFICAR IP ESTÁTICA
ip a | grep -A2 "enp\|eno\|ens" | grep "inet "

# VERIFICAR VERSIONES ACTUALES
echo "=== VERSIONES INSTALADAS ==="
lsb_release -a | grep "Description"
php -v | head -1
mysql --version
apache2 -v | head -1


💾 Paso 1: Backup Total

#!/bin/bash
# Script: backup-glpi.sh
BACKUP_DIR="/opt/glpi-backup-$(date +%Y%m%d-%H%M%S)"
mkdir -p $BACKUP_DIR

# Base de datos
sudo mysqldump -u glpiuser -p'CONTRASEÑA_REAL' glpi > $BACKUP_DIR/glpi-db.sql

# Directorios críticos
sudo tar -czvf $BACKUP_DIR/glpi-config.tar.gz -C /var/www/glpi config --exclude=config_db.php
sudo tar -czvf $BACKUP_DIR/glpi-files.tar.gz -C /var/www/glpi files
sudo tar -czvf $BACKUP_DIR/glpi-plugins.tar.gz -C /var/www/glpi plugins
sudo tar -czvf $BACKUP_DIR/glpi-marketplace.tar.gz -C /var/www/glpi marketplace
sudo tar -czvf $BACKUP_DIR/glpi-full.tar.gz -C /var/www glpi

# Configuración de Apache
sudo cp /etc/apache2/sites-available/glpi.conf $BACKUP_DIR/

echo "✅ Backup completado en: $BACKUP_DIR"
ls -lh $BACKUP_DIR/

🔄 Paso 2: Actualización del Sistema Operativo
Opción A: Ubuntu 22.04 → 24.04

sudo apt update && sudo apt upgrade -y
sudo apt install update-manager-core -y
sudo do-release-upgrade -d
# Sigue instrucciones, el sistema se reiniciará

Opción B: Solo actualizar paquetes (si ya estás en 24.04)

sudo apt update && sudo apt upgrade -y


🐘 Paso 3: Actualización de PHP 8.3

# INSTALAR PHP 8.3 COMPLETO
sudo apt install -y php8.3-fpm php8.3-curl php8.3-gd php8.3-intl php8.3-mysql php8.3-bz2 php8.3-zip php8.3-apcu php8.3-cli php8.3-imap php8.3-mbstring php8.3-dom php8.3-simplexml php8.3-xmlreader php8.3-xmlwriter php8.3-bcmath php8.3-redis php8.3-soap php8.3-cas php8.3-ldap

# CONFIGURAR PHP.INI
sudo nano /etc/php/8.3/fpm/php.ini
# Modificar:
# memory_limit = 256M
# max_execution_time = 300
# upload_max_filesize = 16M
# post_max_size = 16M
# date.timezone = America/Santiago

# ACTIVAR en Apache
sudo a2enmod proxy_fcgi setenvif
sudo a2enconf php8.3-fpm
sudo a2dismod php7.4 2>/dev/null || sudo a2dismod php8.1 2>/dev/null

# REINICIAR
sudo systemctl restart apache2 php8.3-fpm
php8.3 -v  # Confirmar versión

🚀 Paso 4: Instalación de GLPI 11

# DETENER SERVICIOS
sudo systemctl stop apache2

# PRESERVAR DATOS
sudo mkdir -p /opt/glpi10-data
sudo cp -a /var/www/glpi/config/* /opt/glpi10-data/
sudo cp -a /var/www/glpi/files/* /opt/glpi10-data/
sudo cp -a /var/www/glpi/plugins /opt/glpi10-data/
sudo cp -a /var/www/glpi/marketplace /opt/glpi10-data/

# ELIMINAR VIEJO
sudo rm -rf /var/www/glpi

# DESCARGAR Y EXTRAER GLPI 11
cd /tmp
# IMPORTANTE: Verifica última versión en https://github.com/glpi-project/glpi/releases
wget https://github.com/glpi-project/glpi/releases/download/11.0.0-beta5/glpi-11.0.0-beta5.tgz
tar -xvzf glpi-11.0.0-beta5.tgz
sudo mv glpi /var/www/
sudo chown -R www-data:www-data /var/www/glpi
sudo chmod -R 755 /var/www/glpi

# RESTAURAR CONFIGURACIÓN CRÍTICA
sudo cp /opt/glpi10-data/config_db.php /var/www/glpi/config/
sudo cp /opt/glpi10-data/glpicrypt.key /var/www/glpi/config/ 2>/dev/null || echo "No existe glpicrypt.key, se generará automáticamente"

🔧 Paso 5: Configuración Apache + .htaccess
5.1 VirtualHost (solución definitiva)

sudo tee /etc/apache2/sites-available/glpi.conf > /dev/null << 'EOF'
<VirtualHost *:80>
    ServerName 192.168.10.110
    ServerAlias localhost
    DocumentRoot /var/www/glpi/public
    
    ErrorLog ${APACHE_LOG_DIR}/glpi_error.log
    CustomLog ${APACHE_LOG_DIR}/glpi_access.log combined
    
    <Directory /var/www/glpi/public>
        Options -Indexes +FollowSymLinks
        AllowOverride All
        Require all granted
        
        <IfModule mod_rewrite.c>
            RewriteEngine On
            RewriteCond %{REQUEST_FILENAME} !-f
            RewriteCond %{REQUEST_FILENAME} !-d
            RewriteRule ^(.*)$ index.php [QSA,L]
        </IfModule>
    </Directory>
    
    <Directory /var/www/glpi>
        Require all denied
    </Directory>
    
    <Directory /var/www/glpi/public>
        Require all granted
    </Directory>
    
    <IfModule mod_headers.c>
        Header set X-Content-Type-Options "nosniff"
        Header set X-Frame-Options "SAMEORIGIN"
    </IfModule>
</VirtualHost>
EOF

sudo a2dissite 000-default.conf
sudo a2ensite glpi.conf
sudo a2enmod rewrite headers proxy_fcgi setenvif
sudo apache2ctl configtest
sudo systemctl restart apache2

5.2 .htaccess en /public (SOLUCIÓN AL ERROR POST-LOGIN)

sudo tee /var/www/glpi/public/.htaccess > /dev/null << 'EOF'
<IfModule mod_rewrite.c>
    RewriteEngine On
    RewriteCond %{REQUEST_FILENAME} -f
    RewriteRule ^ - [L]
    RewriteCond %{REQUEST_FILENAME} !-f
    RewriteCond %{REQUEST_FILENAME} !-d
    RewriteRule ^(.*)$ index.php [QSA,L]
</IfModule>

<FilesMatch "\.(php|inc|sql|log|twig)$">
    Require all denied
</FilesMatch>

<FilesMatch "^(index|api|apirest|apixmlrpc|cron|front)\.php$">
    Require all granted
</FilesMatch>

Options -Indexes
AddDefaultCharset UTF-8
EOF

sudo chown www-data:www-data /var/www/glpi/public/.htaccess
sudo chmod 644 /var/www/glpi/public/.htaccess

# PERMISOS FINALES
sudo find /var/www/glpi -type d -exec chmod 755 {} \;
sudo find /var/www/glpi -type f -exec chmod 644 {} \;
sudo chmod -R 775 /var/www/glpi/files /var/www/glpi/config

🗄️ Paso 6: Migración de Base de Datos

cd /var/www/glpi

# MANTENIMIENTO Y ACTUALIZACIÓN
sudo -u www-data php bin/console maintenance:enable
sudo -u www-data php bin/console db:update --no-interaction
sudo -u www-data php bin/console db:check --no-interaction
sudo -u www-data php bin/console cache:clear
sudo rm -rf /var/www/glpi/files/_cache/*
sudo -u www-data php bin/console maintenance:disable

# ELIMINAR INSTALL
sudo rm -rf /var/www/glpi/install

🔌 Paso 7: Gestión de Plugins

# LISTAR PLUGINS
sudo -u www-data php bin/console plugin:list

# ACTIVAR PLUGIN COMPATIBLE
sudo -u www-data php bin/console plugin:activate receivers

# SI FALLA, DESACTIVAR Y BUSCAR EN MARKETPLACE
sudo -u www-data php bin/console plugin:deactivate receivers
sudo -u www-data php bin/console plugin:uninstall receivers

# DESCARGAR DESDE MARKETPLACE (ejemplo)
cd /var/www/glpi
sudo -u www-data php bin/console marketplace:install receivers

✅ Paso 8: Verificación y Puesta en Producción

# VERIFICAR SERVICIOS
sudo systemctl status apache2 php8.3-fpm mariadb

# VER LOGS
sudo tail -20 /var/log/apache2/glpi_error.log

# VERIFICAR REQUISITOS
sudo -u www-data php bin/console system:check_requirements

# ACCESO
echo "===================================="
echo "URL: http://192.168.10.110"
echo "Usuario: glpi"
echo "Contraseña: [misma que GLPI 10]"
echo "===================================="

🚨 Solución de Emergencia: Volver a GLPI 10
Si GLPI 11 falla, ejecuta esto INMEDIATAMENTE:

#!/bin/bash
# Script: rollback-glpi.sh

echo "⚠️  INICIANDO ROLLBACK A GLPI 10"
sudo systemctl stop apache2
sudo rm -rf /var/www/glpi
sudo tar -xzvf /opt/glpi-migracion/glpi10-full.tar.gz -C /var/www/
sudo mysql -u root -p'CONTRASEÑA' glpi < /opt/glpi-migracion/glpi10-db-*.sql
sudo systemctl start apache2
echo "✅ ROLLBACK COMPLETADO"




