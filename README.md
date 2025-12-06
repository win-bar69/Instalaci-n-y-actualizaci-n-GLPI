# 🚀 Guía Profesional: Migración GLPI 10.x → 11.x Solo por CLI

**Versión:** 1.0  
**Autor:** [Tu Nombre]  
**Última actualización:** 2025-12-06  
**Licencia:** MIT

Proceso completo de migración de GLPI 10.x a 11.x únicamente mediante línea de comandos, incluyendo solución definitiva para problemas de `.htaccess` y reescritura de URLs.

---

## 📋 Tabla de Contenidos

1. [El Problema de `.htaccess` en GLPI 11](#el-problema-de-htaccess-en-glpi-11)
2. [Requisitos Previos](#requisitos-previos)
3. [Paso 0: Verificación y Credenciales](#paso-0-verificación-y-credenciales)
4. [Paso 1: Backup Total](#paso-1-backup-total)
5. [Paso 2: Actualización del Sistema Operativo](#paso-2-actualización-del-sistema-operativo)
6. [Paso 3: Actualización de PHP 8.3](#paso-3-actualización-de-php-83)
7. [Paso 4: Instalación de GLPI 11](#paso-4-instalación-de-glpi-11)
8. [Paso 5: Configuración Apache + `.htaccess`](#paso-5-configuración-apache--htaccess)
9. [Paso 6: Migración de Base de Datos](#paso-6-migración-de-base-de-datos)
10. [Paso 7: Gestión de Plugins](#paso-7-gestión-de-plugins)
11. [Paso 8: Verificación y Puesta en Producción](#paso-8-verificación-y-puesta-en-producción)
12. [🚨 Solución de Emergencia](#solución-de-emergencia)
13. [🔧 Troubleshooting](#troubleshooting)
14. [⚡ Quick Reference](#quick-reference)

---

## 🎯 El Problema de `.htaccess` en GLPI 11

### **Causa Raíz**

GLPI 11 cambió su estructura de seguridad:

| Versión | DocumentRoot | Ubicación `.htaccess` | Consecuencia |
|---------|--------------|----------------------|--------------|
| **GLPI 10** | `/var/www/glpi/` | Raíz del proyecto | Funciona por defecto |
| **GLPI 11** | `/var/www/glpi/public` | **OBLIGATORIO** en `/public` | Si falta o tiene permisos incorrectos → **Error "Not Found"** |

### **Síntomas**
- Página de login aparece
- Al ingresar credenciales → `404 Not Found`
- `/install/install.php` no se encuentra
- URLs no se reescriben correctamente

### **Solución**
1. `DocumentRoot` **DEBE** apuntar a `/public`
2. Archivo `.htaccess` **DEBE** existir en `/public` con reglas de reescritura
3. `AllowOverride All` **DEBE** estar en el VirtualHost
4. Módulo `rewrite` **DEBE** estar activo

---

## 📋 Requisitos Previos

- **IP Estática** configurada en el servidor
- Acceso root o sudo
- GLPI 10.x funcionando
- Conocer credenciales MySQL (`config_db.php`)
- 30-60 minutos de ventana de mantenimiento
- Al menos 2GB de RAM libres
