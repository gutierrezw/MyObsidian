C:\Users\54911\AppData\Roaming\Microsoft\Windows\Start Menu\Programs\Python 3.11
==================================================================================
instalacion de modulos

github:
    https://github.com/gutierrezw/MyPython

Softare y Datos:
	./MyPython
	./dumps


Software base:
	java: https://www.java.com/es/download/ie_manual.jsp
	node: https://nodejs.org/es (latest LTS for windows)
	mysql : https://dev.mysql.com/downloads/
	python install Nabager: https://www.python.org/downloads/windows/
	tsw: https://www.interactivebrokers.co.uk/es/trading/ib-api.php
	git: https://git-scm.com/downloads/win
	clientportal.gw:  https://www.interactivebrokers.com/es/trading/ib-api.php

Node.js — uso en el proyecto:
	Node se usa exclusivamente para server-api: proxy HTTP autenticado sobre MySQL,
	bridge TradingView (Fase 2), y MCP server para el agente autónomo (Fase 3).
	No reemplaza Python — conviven: Python corre la app Tkinter, Node corre el servidor.
	Versión activa: v24.13.1 / npm 11.8.0 (verificado 2026-06-28)
	Proceso gestionado por PM2 (auto-start, auto-restart, logs integrados).
	Diseño completo: ver [[design-api-server]]

	Repo: Documents\MyNode\  (separado de MyPython — código Node puro)
	Credenciales: Documents\Claude-Cowork-Scripts\mysql_config.json  (fuera del repo, nunca commiteado)

==================================================================================
INSTALACION server-api (Fase 1) — pasos completos
==================================================================================

	Qué es:
		Servidor Node.js standalone que actúa como proxy HTTP autenticado sobre MySQL.
		Permite que co-working y el agente autónomo consulten la BD sin exponer el puerto 3306.
		Corre en puerto 8050, independiente de AppOO Tkinter.

	Paso 1 — Clonar / ubicar el código
		Carpeta: C:\Users\InversionesWildaga\Documents\MyNode\server-api\
		Branch:  server-api-fase1

	Paso 2 — Crear el archivo de configuración (NUNCA commitear)
		Ubicación: C:\Users\InversionesWildaga\Documents\Claude-Cowork-Scripts\mysql_config.json
		Copiar desde config.example.json y completar:
		{
		  "api_key": "<generar con comando abajo>",
		  "port": 8050,
		  "db": { "host": "localhost", "port": 3306, "user": "root", "password": "...", "database": "bdinv" }
		}

		Generar API key (PowerShell):
		$bytes = New-Object byte[] 32
		[System.Security.Cryptography.RandomNumberGenerator]::Create().GetBytes($bytes)
		[System.Convert]::ToBase64String($bytes)

	Paso 3 — Instalar dependencias
		cd C:\Users\InversionesWildaga\Documents\MyNode\server-api
		npm install

	Paso 4 — Instalar PM2 globalmente
		npm install -g pm2

	Paso 5 — Levantar el servidor con PM2
		pm2 start ecosystem.config.js
		pm2 list   <- verificar status "online"

	Paso 6 — Configurar auto-start con Windows (ejecutar como Administrador)
		npx pm2-windows-startup install
		pm2 save

	Verificación final:
		curl http://localhost:8050/health
		-> { "status": "ok", "version": "1.0.0", "uptime": ... }

	Comandos del día a día:
		pm2 list                  <- estado actual
		pm2 logs server-api       <- logs en tiempo real
		pm2 restart server-api    <- reiniciar
		pm2 stop server-api       <- detener

	PENDIENTE — mantenimiento de versiones Node (BACKLOG #62):
		Al igual que Python, definir proceso de actualización controlada de Node:
		- evaluar nvm-windows como gestor de versiones (permite tener v18/v20/v24 en paralelo)
		- o winget upgrade OpenJS.NodeJS para actualizaciones simples
		- documentar versión mínima requerida por server-api
		- validar PM2 + dependencias npm tras cada upgrade
		

Excepciones:
	clientportal.gw:  mover las carptas a resource ubicada en Mypython
	Ejecutar eventualmente TSW para que se instalen nuevos certificados o prametros globales de las API
	Copiar confg.yaml deszde backup a root -- configuracion personalziada para el puerto 5501	
	


Comandos upgrate python
===================================================================================
pip list
pip install <paquete>
pip uninstall <paquete>
cuando cambie de version
Set-ExecutionPolicy -ExecutionPolicy Unrestricted -Scope CurrentUser
Get-ExecutionPolicy -List

== identifica las dependencias de APP
pip install pipreqs

== Ejecuta
pipreqs --force
pip install -r requirements.txt


Ejecucion de API
resources\bin\run.bat resources\root\conf.yaml
==================================================================================
Instalacion de API IBKRs  desde directorio

cd  C:\Users\54911\Documents\MyPython\TWS-API\source\pythonclient
pip install .
==================================================================================

==================================================================================
API Key y Secret Key: ver gestor de contrasenas (NO guardar aqui)
==================================================================================
Binance-conector

pip install binance-connector


==================================================================================
BACKUP DIARIO AUTOMATIZADO
==================================================================================

Ubicacion del script:
	C:\Users\InversionesWildaga\Documents\backup_script\backup_diario.bat

Destino:
	Google Drive: G:\Mi unidad\Backup\
		dumps\YYYYMMDD\       <- dump MySQL diario
		ebook\YYYYMMDD\       <- libros electronicos
		Inversiones\YYYYMMDD\ <- carpeta inversiones

Retencion:
	15 dias - rotacion automatica de carpetas antiguas

Base de datos respaldada:
	MySQL local - base: bdinv
	Usuario: root
	Dump completo: schema + data
	Archivo generado: dumps\bdinv_YYYYMMDD.sql

Programacion automatica (Task Scheduler):
	Nombre tarea: Wildaga Backup
	Frecuencia: Diaria
	Hora: 07:00
	Usuario: InversionesWildaga

Comandos utiles:
	-- Ver estado de la tarea
	schtasks /query /tn "Wildaga Backup" /fo LIST /v

	-- Forzar ejecucion manual
	schtasks /run /tn "Wildaga Backup"

	-- Ver log del dia
	type C:\Users\InversionesWildaga\Documents\backup_YYYYMMDD.log

Pendiente activar (cuando se configuren MCPs en VS Code):
	Descomentar seccion MCP en backup_diario.bat
	Ruta config MCP: %APPDATA%\Code\User\globalStorage\saoudrizwan.claude-dev\settings
	Destino: G:\Mi unidad\Backup\VSCode-MCP\YYYYMMDD\


==================================================================================
REPORTE SEMANAL MYSQL AUTOMATIZADO
==================================================================================

Por qué es local (no Cowork cloud):
	El sandbox de Cowork es un contenedor Linux aislado en la nube, sin ruta de red
	hacia este PC. Confirmado con curl -> "Connection refused" a localhost:8050.
	Por eso la tarea programada de Cowork "mysql-weekly-report" fue desactivada
	(quedó en estado enabled:false) y reemplazada por Task Scheduler local,
	igual que "Wildaga Backup".

Ubicacion del script:
	C:\Users\InversionesWildaga\Documents\MyNode\mysql-weekly-report\report.js

Que hace:
	1. Llama a GET http://localhost:8050/db/diagnostics (server-api, requiere x-api-key)
	2. Construye HTML con: tablas por tamaño, índices sin uso, queries con full scan, buffer pool
	3. Guarda la salida en .\output\reporte-YYYYMMDD.html y .\output\reporte_ultimo.html
	   (NO envía email — solo deja el documento para revisión manual)
	4. Si algo falla, escribe el error en .\output\ y registra log en .\logs\

Endpoint nuevo en server-api:
	GET /db/diagnostics  (routes/db.js) — solo lectura, queries fijas sobre
	information_schema / performance_schema. No usa ALLOWED_TABLES (es un endpoint
	separado del /db/query genérico, no lo modifica).
	IMPORTANTE: tras este cambio hace falta "pm2 restart server-api" para recargarlo.

Programacion automatica (Task Scheduler):
	Nombre tarea: "Reporte Semanal MySQL"
	Frecuencia: Semanal, Lunes
	Hora: 08:00
	Comando: node C:\Users\InversionesWildaga\Documents\MyNode\mysql-weekly-report\report.js

Comandos utiles:
	-- Probar manualmente
	cd C:\Users\InversionesWildaga\Documents\MyNode\mysql-weekly-report
	node report.js

	-- Ver logs
	type C:\Users\InversionesWildaga\Documents\MyNode\mysql-weekly-report\logs\report-YYYYMMDD.log

