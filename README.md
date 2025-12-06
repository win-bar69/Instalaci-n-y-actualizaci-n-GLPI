🎯 INTRODUCCIÓN
Esta guía documenta el proceso 100% por línea de comandos para migrar GLPI 10.x a 11.x, incluyendo actualización del sistema operativo Ubuntu. El foco especial está en la configuración de .htaccess y Apache, que es la fuente del 90% de los errores en migraciones.
Requisitos previos:
Acceso root/sudo al servidor
GLPI 10.x funcionando en Ubuntu 22.04 o 24.04
IP estática configurada
Conocimiento de tus credenciales MySQL
🔍 EL PROBLEMA DE .HTACCESS EN GLPI 11
¿Por qué fallan las URLs después de la migración?
GLPI 11 cambió su arquitectura de seguridad:
GLPI 10: DocumentRoot /var/www/glpi/ → El .htaccess estaba en la raíz
GLPI 11: DocumentRoot /var/www/glpi/public/ → El .htaccess DEBE estar en /public
El error "Not Found" después del login ocurre porque:
Apache no encuentra /public/.htaccess (no existe o permisos incorrectos)
AllowOverride All no está configurado en el VirtualHost
El módulo rewrite no está activo
Las reglas de reescritura no se aplican → Apache busca /install/install.php literalmente
Solución: Configurar VirtualHost APUNTE a /public Y cree .htaccess funcional.
