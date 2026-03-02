# CyberDefenders – Reveal Lab (Volatility 3)

**Autor**: Santiago Daniel Sandili  
**Fecha**: Marzo 2026  

---

## 1. Resumen del caso

El SIEM detectó actividad rara en una workstation con acceso a datos financieros.  
Nos dieron un memory dump y el objetivo del lab es encontrar el proceso malicioso, ver qué hace, con qué se conecta y a qué familia de malware pertenece.

---

## 2. Qué usé y cómo lo enfoqué

- **Sistema**: Volatility 3 sobre el dump `Reveal.dmp`
- **Plugins principales**:
  - `windows.pslist`, `windows.cmdline` y `windows.info` para ver procesos, comandos e información del sistema.
  - `windows.netscan` para conexiones de red
- **OSINT**:
  - VirusTotal para revisar la IP que aparecía en el comando de PowerShell
  - Búsqueda de familia de malware a partir de los archivos relacionados a esa IP

**La idea fue**: primero ver qué procesos se están ejecutando, después buscar algo raro en las líneas de comando, y finalmente pivotear con OSINT.

---

## 3. Hallazgos paso a paso

### 3.1. Proceso malicioso y padre

Con `windows.pslist` y `windows.cmdline` encontré un proceso con un comando de PowerShell sospechoso que hacía un `net use` a un recurso remoto y luego ejecutaba un DLL con una utilidad de Windows.

**Datos importantes que extraje**:

Nombre del proceso malicioso: powershell.exe (PID: 4120)

PPID (parent PID): 3692

Usuario bajo el que corre: Elon



### 3.2. Segundo stage y recurso compartido

En la línea de comando del proceso se veía claramente:

Un net use apuntando a la IP remota: 45.9.74.32:8888

Recurso compartido: \45.9.74.32@8888\davwwwroot\

Archivo DLL: 3435.dll ejecutado como second-stage


También se observa que se usa una utilidad legítima de Windows para ejecutar el DLL, lo que encaja con:

**MITRE ATT&CK sub-technique**: `T1218.011`

---

## 4. OSINT y familia de malware

En vez de dumpear memoria o binarios, aproveché la información directa del comando:

1. **Saqué la IP remota** del comando de PowerShell: `45.9.74.32:8888`
2. **Busqué esa IP en VirusTotal**
3. **Miré los archivos relacionados** con esa IP
4. **Varios de esos archivos** estaban etiquetados por distintos engines como pertenecientes a la familia **StrelaStealer**

**Nombre de la familia de malware**: `StrelaStealer`

Este enfoque es bastante realista para un analista SOC: muchas veces no tenés que hacer reversing profundo si con logs + OSINT ya podés clasificar la amenaza.

---

## 5. Resumen de respuestas del lab

Proceso malicioso: powershell.exe (PID: 4120)

Parent PID: 3692

Archivo del second-stage: 3435.dll

Shared directory: \45.9.74.32@8888\davwwwroot\

MITRE ATT&CK sub-technique ID: T1218.011

Usuario del proceso: Elon

Familia de malware: StrelaStealer


---

## 6. Qué aprendí

- Usar `pslist` + `cmdline` en Volatility para encontrar procesos sospechosos de forma rápida.
- Leer bien las líneas de comando puede darte casi toda la historia del ataque sin necesidad de dumpear todo.
- Pivotear una IP a VirusTotal y usar los archivos relacionados para identificar la familia de malware.
- Con pocos pasos se puede reconstruir una cadena de ataque bastante completa.
