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
  - [Instalando Paquetes Globales](#instalando-paquetes-globales)
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
- 🏴 **Soporte completo de flags** - Compatible con `-g`, `--save-dev`, `--save-optional` y todas las banderas de npm

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

### Instalando Paquetes Globales

El comando `npm-safe` ahora soporta todas las banderas de npm, incluyendo instalaciones globales:

```powershell
# Instalación global
npm-safe @open-wc/building-rollup -g

# Instalación como dependencia de desarrollo
npm-safe eslint --save-dev

# Instalación como dependencia opcional
npm-safe sharp --save-optional

# Combinación de flags
npm-safe typescript -g --force
```

**Importante**: El sistema reconoce automáticamente las flags y las aplica correctamente, preservando la funcionalidad completa de npm mientras mantiene la protección de seguridad.

---

### Trabajando con Proyectos Clonados

Cuando descargues un proyecto existente (de la empresa o de terceros) que ya tiene un archivo `package-lock.json`, **no uses** `npm install`. La práctica de seguridad recomendada es:

```powershell
npm ci
```

Este comando borra la carpeta `node_modules` e instala las versiones exactas firmadas del archivo de bloqueo, asegurando que no se descargue código publicado recientemente.

---

### Paquetes que Requieren Compilación

Debido a que bloqueamos los scripts globales con `ignore-scripts=true`, algunas librerías legítimas que compilan binarios nativos (como `esbuild`, `sass`, `cypress`, `@open-wc/building-rollup`) pueden requerir un paso extra después de la instalación segura.

#### Síntoma del Problema

Si después de instalar un paquete global con `npm-safe` obtienes un error al ejecutarlo como:

```
Error: Cannot find module 'xyz'
command not found: xyz
```

O el paquete simplemente no funciona correctamente, es probable que necesite compilación.

#### Solución: Permitir Scripts para Paquetes Específicos

**Opción 1: Reconstruir el paquete específico** (Recomendado)

Después de instalar el paquete con `npm-safe`, ejecuta:

```powershell
npm rebuild nombre-del-paquete
```

**Ejemplo real con @open-wc/building-rollup:**

```powershell
# 1. Instalar de forma segura
npm-safe @open-wc/building-rollup -g

# 2. Reconstruir para permitir la compilación
npm rebuild @open-wc/building-rollup

# 3. Ahora el comando funcionará correctamente
```

**Opción 2: Excluir paquetes específicos de la restricción global**

Si un paquete en particular es de confianza y necesitas permitir sus scripts permanentemente:

```bash
# Configurar excepciones para paquetes específicos
npm config set ignore-scripts false --location=project

# O permitir scripts solo para ese paquete usando .npmrc en tu proyecto
echo "ignore-scripts=true" >> .npmrc
echo "@open-wc:ignore-scripts=false" >> .npmrc
```

**Opción 3: Instalación manual bypass (último recurso)**

Si necesitas instalar un paquete urgentemente y estás seguro de su legitimidad:

```powershell
# Temporalmente deshabilitar la protección
npm config set ignore-scripts false

# Instalar el paquete
npm.cmd install nombre-del-paquete -g

# Restaurar la protección inmediatamente
npm config set ignore-scripts true
```

> ⚠️ **Advertencia**: Solo usa la Opción 3 cuando estés completamente seguro de la confiabilidad del paquete.

#### Lista de Paquetes Comunes que Requieren Rebuild

Estos paquetes legítimos comúnmente necesitan `npm rebuild` después de instalarse:

- `@open-wc/building-rollup` - Herramientas de build para Web Components
- `esbuild` - Bundler y minificador ultrarrápido
- `sass`, `node-sass` - Preprocesador CSS
- `cypress` - Framework de testing E2E
- `puppeteer` - Automatización de Chrome
- `sharp` - Procesamiento de imágenes
- `sqlite3` - Base de datos SQLite
- `bcrypt` - Hashing de contraseñas
- `canvas` - Generación de imágenes en Node.js

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
- Soporta todas las flags de npm (`-g`, `--save-dev`, `--force`, etc.)

---

## 🐛 Solución de Problemas

### Problema: PowerShell dice "la ejecución de scripts está deshabilitada"
**Solución**: Ejecuta el Paso 2 nuevamente con privilegios de administrador:
```powershell
Set-ExecutionPolicy -ExecutionPolicy RemoteSigned -Scope CurrentUser -Force
```

---

### Problema: Comando `npm-safe` no encontrado
**Solución**: Ejecuta `. $PROFILE` para recargar tu perfil de PowerShell:
```powershell
. $PROFILE
```

---

### Problema: El paquete global se instaló pero no funciona al ejecutarlo
**Causa**: El paquete requiere compilación de binarios nativos, pero `ignore-scripts=true` bloqueó el proceso.

**Solución**: Usa `npm rebuild` después de la instalación:
```powershell
npm rebuild nombre-del-paquete
```

**Ejemplo completo**:
```powershell
# Instalación
npm-safe esbuild -g

# Si falla al ejecutar, reconstruir
npm rebuild esbuild

# Ahora debería funcionar
esbuild --version
```

---

### Problema: Necesito instalar un paquete publicado hace menos de 24 horas
**Solución**: Verifica manualmente la seguridad del paquete en:
- https://npmjs.com/package/nombre-paquete (verificar autor, descargas, actividad)
- https://socket.dev/ (análisis de seguridad)
- https://snyk.io/vuln/ (vulnerabilidades conocidas)

Luego usa bypass directo:
```powershell
npm.cmd install nombre-paquete@version
```

---

### Problema: Instalación global con `-g` no funciona
**Causa**: Probablemente usaste el alias bloqueado `npm install -g`.

**Solución**: Usa `npm-safe` que ahora soporta todas las flags:
```powershell
npm-safe nombre-paquete -g
```

---

### Problema: El guardián no detecta mi intento de `npm install`
**Causa**: Puede que el perfil no esté cargado o el alias no esté activo.

**Solución**: Recarga el perfil:
```powershell
. $PROFILE
```

Verifica el alias:
```powershell
Get-Alias npm
```

Deberías ver: `Invoke-NpmShield`

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
- Los paquetes que requieren `npm rebuild` son legítimos, pero **siempre verifica primero** en npm, Socket.dev o Snyk antes de reconstruir

---

## 🔍 Recursos de Verificación de Seguridad

Antes de permitir scripts o reconstruir un paquete, verifica su seguridad en:

- **npm oficial**: https://www.npmjs.com/package/NOMBRE_PAQUETE
- **Socket.dev**: https://socket.dev/npm/package/NOMBRE_PAQUETE
- **Snyk Advisor**: https://snyk.io/advisor/npm-package/NOMBRE_PAQUETE
- **GitHub Repository**: Revisa el código fuente del paquete
- **npm audit**: Ejecuta `npm audit` regularmente en tus proyectos

---

**¡Entorno de desarrollo blindado con éxito!** 🛡️
