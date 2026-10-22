# Evidencias visuales

Esta carpeta contiene una selección de evidencias técnicas del laboratorio.

No se incluyen todas las capturas generadas durante la práctica, sino únicamente aquellas que permiten demostrar los puntos más relevantes del despliegue, hardening y validación del sistema.

## Evidencias seleccionadas

### Virtualización

- Extensiones de virtualización anidada habilitadas.
- VMware Workstation ejecutando correctamente Ubuntu Server.

### Identidad y autenticación

- Política `sudoers`.
- Integración y validación de `pam_faillock`.
- Bloqueo real de una cuenta tras varios intentos fallidos.

### SSH

- Configuración efectiva de OpenSSH mediante `sshd -T`.
- Rechazo de autenticación mediante contraseña.
- Rechazo de acceso remoto como `root`.
- Acceso correcto mediante clave Ed25519 y puerto personalizado.

### Mantenimiento y servicios

- Sincronización mediante Chrony.
- Validación de `unattended-upgrades`.
- Inspección final de servicios mediante `ss -tulpn`.

---

Las capturas han sido seleccionadas únicamente con fines de documentación técnica y se han evitado credenciales, claves privadas y otra información sensible.
