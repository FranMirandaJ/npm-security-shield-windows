# 🛡️ Escudo de Seguridad npm para Windows 10

Una configuración de seguridad completa para proteger tu estación de trabajo Windows 10 contra ataques a la cadena de suministro de npm, incluyendo malware en scripts de instalación, typosquatting y secuestro de paquetes.

---

## 📋 Tabla de Contenidos

- [Descripción General](#descripción-general)
- [Características](#características)
- [Guía de Instalación](#guía-de-instalación)
  - [Paso 1: Configuración Global de npm](#paso-1-configuración-global-de-npm-la-coraza)
  - [Paso 2: Habilitar Ejecución de Scripts](#paso-2-habilitar-ejecución-de-scripts-en-windows)
  - [Paso 3: Configurar el Script Guardián](#paso-3-configurar-el-script-guardián)
  - [Paso 4: Aplicar Cambios](#paso-4-aplicar-cambios)
- [Uso Diario](#uso-diario)
  - [Instalando Paquetes Nuevos](#instalando-paquetes-nuevos)
  - [Trabajando con Proyectos Clonados](#trabajando-con-proyectos-clonados)
  - [Paquetes que Requieren Compilación](#paquetes-que-requieren-compilación)
- [Cómo Funciona](#cómo-funciona)
- [Solución de Problemas](#solución-de-problemas)
- [Licencia](#licencia)

---

## 🎯 Descripción General

El ecosistema de npm enfrenta constantemente ataques donde se publican versiones maliciosas de librerías populares. Este sistema implementa una **defensa de dos capas**:

1. **Coraza Global (`.npmrc`)**: Bloquea la ejecución automática de scripts de instalación (el principal vector de infección) y fija versiones exactas para prevenir actualizaciones sorpresa.

2. **Guardián de Cuarentena (PowerShell / VS Code)**: Intercepta inteligentemente el comando `npm install` al agregar nuevos paquetes. Si un paquete fue publicado hace menos de 24 horas, detiene la instalación y busca automáticamente la última versión segura (estable en el tiempo) para proteger tu sistema sin interrumpir tu flujo de trabajo.

---

## ✨ Características

- 🔒 **Bloqueo automático de scripts** - Previene la ejecución de código malicioso durante la instalación
- ⏰ **Cuarentena de 24 horas** - Bloquea paquetes recién publicados automáticamente
- 🔍 **Alternativa inteligente** - Encuentra e instala la última versión segura
- 🎯 **Bloqueo de versión exacta** - Previene actualizaciones inesperadas
- 🚫 **Protección contra memoria muscular** - Intercepta comandos `npm install` inseguros
- ✅ **Flujo de trabajo sin interrupciones** - Funciona con PowerShell y terminal de VS Code

---

## 🚀 Guía de Instalación

Sigue estos pasos en tu máquina Windows 10 (aplica tanto para la consola nativa de PowerShell como para la terminal integrada de VS Code).

### Paso 1: Configuración Global de npm (La Coraza)

Ejecuta los siguientes comandos en tu terminal para establecer las reglas de seguridad globales en tu archivo `.npmrc`:

```bash
# Evita que los paquetes ejecuten código en tu máquina al instalarse
npm config set ignore-scripts true

# Fuerza a npm a guardar versiones exactas en package.json (evita el uso de ^)
npm config set save-exact true
```

**Verificación**: Ejecuta `npm config ls` para confirmar que la configuración se guardó correctamente.

---

### Paso 2: Habilitar Ejecución de Scripts en Windows

Por defecto, Windows 10 bloquea la carga de perfiles personalizados de PowerShell. Ejecuta este comando para permitir que tu usuario cargue el escudo de seguridad:

```powershell
Set-ExecutionPolicy -ExecutionPolicy RemoteSigned -Scope CurrentUser -Force
```

---

### Paso 3: Configurar el Script Guardián

En tu terminal (preferiblemente dentro de VS Code), ejecuta el siguiente comando para abrir el archivo de perfil que la consola lee al arrancar:

```powershell
notepad $PROFILE
```

> **Nota**: Si el Bloc de notas te pregunta si deseas crear un archivo nuevo, selecciona **Sí**.

Borra cualquier contenido previo y pega el siguiente código completo:

```powershell
function npm-safe {
    param(
        # El nombre del paquete va en la primera posición
        [Parameter(Mandatory=$true, Position=0)]
        [string]$Paquete,
        
        # Captura cualquier flag adicional (ej: -g, --save-dev)
        [Parameter(ValueFromRemainingArguments=$true)]
        [string[]]$Flags
    )

    Write-Host "Revisando el historial de '$Paquete' en npm..." -ForegroundColor Cyan
    
    try {
        $respuesta = Invoke-RestMethod -Uri "https://registry.npmjs.org/$Paquete"
        $versionLatest = $respuesta.'dist-tags'.latest
        $fechaPublicacion = [datetime]$respuesta.time.$versionLatest
        $horasTranscurridas = (Get-Date) - $fechaPublicacion
        
        # Límite de cuarentena (24 horas)
        if ($horasTranscurridas.TotalHours -lt 24) {
            Write-Host "---------------------------------------------------" -ForegroundColor Red
            Write-Host "[ALERTA ROJA] Paquete demasiado reciente detectado!" -ForegroundColor Red
            Write-Host " -> Paquete: $Paquete" -ForegroundColor Red
            Write-Host " -> Version bloqueada: $versionLatest ($([math]::Round($horasTranscurridas.TotalHours, 1)) horas)" -ForegroundColor Red
            Write-Host "---------------------------------------------------" -ForegroundColor Red
            Write-Host "Buscando la ultima version segura disponible..." -ForegroundColor Yellow
            
            $tiempos = $respuesta.time
            $versiones = $tiempos.psobject.properties | Where-Object { $_.Name -match '^\d+\.\d+\.\d+$' }
            
            # Filtrar versiones con más de 24 horas
            $versionesSeguras = $versiones | Where-Object { ((Get-Date) - [datetime]$_.Value).TotalHours -ge 24 } | Sort-Object { [datetime]$_.Value } -Descending
            
            if ($versionesSeguras.Count -gt 0) {
                $versionSegura = $versionesSeguras[0].Name
                $horasSegura = ((Get-Date) - [datetime]$versionesSeguras[0].Value).TotalHours
                
                Write-Host "[EXITO] Alternativa segura encontrada:" -ForegroundColor Green
                Write-Host " -> Version a instalar: $versionSegura ($([math]::Round($horasSegura, 1)) horas)" -ForegroundColor Green
                
                # Instalar versión segura pasando las flags
                $argumentos = @("install", "$Paquete@$versionSegura") + $Flags
                & "npm.cmd" @argumentos
            } else {
                Write-Host "[ERROR] No se encontro ninguna version con mas de 24 horas. Cancelando." -ForegroundColor Red
            }
        } else {
            Write-Host "La version mas nueva ($versionLatest) es segura ($([math]::Round($horasTranscurridas.TotalHours, 1)) horas). Instalando..." -ForegroundColor Green
            
            # Instalar versión más reciente pasando las flags
            $argumentos = @("install", $Paquete) + $Flags
            & "npm.cmd" @argumentos
        }
    } catch {
        Write-Host "Error al consultar el paquete. Verifica que el nombre este bien escrito." -ForegroundColor Red
    }
}

function Invoke-NpmShield {
    $comando = $args[0]
    
    # Interceptar si el comando es 'install' o 'i' y hay más de un argumento
    if (($comando -eq 'install' -or $comando -eq 'i') -and $args.Length -gt 1) {
        
        $paquete = $null
        $flags = @()
        
        # Separar inteligentemente cuál es el paquete y cuáles son las flags
        for ($i = 1; $i -lt $args.Length; $i++) {
            if ($args[$i] -match '^-') {
                $flags += $args[$i]
            } elseif ($null -eq $paquete) {
                $paquete = $args[$i]
            }
        }
        
        if ($paquete) {
            Write-Host "[ALTO AHI] La memoria muscular te traiciono." -ForegroundColor Red
            Write-Host "Recuerda que tienes activa la proteccion contra paquetes nuevos." -ForegroundColor Yellow
            
            # Construir la sugerencia exacta incluyendo las flags
            $flagsString = if ($flags.Count -gt 0) { " " + ($flags -join " ") } else { "" }
            Write-Host "-> Por favor, ejecuta: npm-safe $paquete$flagsString" -ForegroundColor Cyan
            return
        }
    }

    # Para cualquier otro comando, ejecutar npm real
    & "npm.cmd" @args
}

Set-Alias -Name npm -Value Invoke-NpmShield -Scope Global -Force
```

**Guarda** el archivo (`Ctrl + S`) y cierra el Bloc de notas.

---

### Paso 4: Aplicar Cambios

Para activar el escudo de inmediato sin reiniciar tu computadora, ejecuta en la terminal:

```powershell
. $PROFILE
```

---

## 💼 Uso Diario

Una vez configurado, tu flujo de trabajo cambia ligeramente para garantizar tu seguridad:

### Instalando Paquetes Nuevos

Si escribes `npm install nombre-paquete` por hábito, la terminal bloqueará la acción (previniendo que la memoria muscular te exponga a riesgos) y te recordará el comando correcto:

```
[ALTO AHI] La memoria muscular te traiciono.
Recuerda que tienes activa la proteccion contra paquetes nuevos.
-> Por favor, ejecuta: npm-safe nombre-paquete
```

**Usa el comando seguro en su lugar:**

```powershell
npm-safe nombre-paquete
```

El guardián:
- ✅ Verificará si el paquete fue publicado en las últimas 24 horas
- ✅ Si es inseguro, encontrará e instalará la última versión segura
- ✅ Si es seguro, procederá con la instalación normalmente

---

### Trabajando con Proyectos Clonados

Cuando descargues un proyecto existente (de la empresa o de terceros) que ya tiene un archivo `package-lock.json`, **no uses** `npm install`. La práctica de seguridad recomendada es:

```powershell
npm ci
```

Este comando borra la carpeta `node_modules` e instala las versiones exactas firmadas del archivo de bloqueo, asegurando que no se descargue código publicado recientemente.

---

### Paquetes que Requieren Compilación

Debido a que bloqueamos los scripts globales con `ignore-scripts=true`, algunas librerías legítimas que compilan binarios nativos (ej., `esbuild`, `sass`, `cypress`) pueden requerir un paso extra después de la instalación segura. Si notas que una herramienta no funciona, compílala manualmente después de validar que es segura:

```powershell
npm rebuild nombre-del-paquete
```

---

## 🔧 Cómo Funciona

### La Defensa de Dos Capas

**Capa 1: Configuración Global**
- `ignore-scripts true` - Bloquea todos los scripts de instalación por defecto
- `save-exact true` - Fija versiones exactas (sin rangos `^` o `~`)

**Capa 2: Guardián PowerShell**
- Intercepta comandos `npm install`
- Consulta el registro de npm para conocer el tiempo de publicación del paquete
- Aplica período de cuarentena de 24 horas
- Auto-selecciona la última versión segura si la más reciente es muy nueva
- Permite comandos npm normales (`npm start`, `npm test`, etc.)

---

## 🐛 Solución de Problemas

**Problema**: PowerShell dice "la ejecución de scripts está deshabilitada"
- **Solución**: Ejecuta el Paso 2 nuevamente con privilegios de administrador

**Problema**: Comando `npm-safe` no encontrado
- **Solución**: Ejecuta `. $PROFILE` para recargar tu perfil de PowerShell

**Problema**: El paquete requiere compilación pero falla al ejecutarse
- **Solución**: Usa `npm rebuild nombre-del-paquete` después de la instalación

**Problema**: Quiero instalar un paquete publicado hace menos de 24 horas
- **Solución**: Verifica manualmente la seguridad del paquete, luego usa `npm.cmd install nombre-paquete@version` para saltarte el guardián

---

## 📝 Licencia

Esta configuración se proporciona tal cual para fines de refuerzo de seguridad. Úsala bajo tu propia discreción.

---

## ⚠️ Notas Importantes de Seguridad

- Esta configuración reduce significativamente el riesgo pero no elimina todas las amenazas
- Siempre verifica los nombres de los paquetes cuidadosamente para evitar typosquatting
- Revisa las dependencias de los paquetes regularmente
- Mantén tus versiones de npm y Node.js actualizadas
- Considera usar herramientas adicionales como `npm audit` regularmente

---

**¡Entorno de desarrollo blindado con éxito!** 🛡️
