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

	Comandos Node útiles:
		node --version              <- verificar versión instalada
		npm install                 <- instalar dependencias (equivalente a pip install -r)
		npm install -g pm2          <- instalar PM2 globalmente
		pm2 start ecosystem.config.js
		pm2 list                    <- ver procesos activos
		pm2 logs server-api         <- ver logs en tiempo real
		pm2 restart server-api
		pm2 stop server-api

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

