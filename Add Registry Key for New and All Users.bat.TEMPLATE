@echo off
setlocal ENABLEEXTENSIONS ENABLEDELAYEDEXPANSION

:: -------------------------------------------------------
:: Configurable Variables
:: -------------------------------------------------------

:: Define the full registry subkey path to modify (under HKCU/HKU or loaded hive)
:: Example: "REG_PATH=Software\Classes\CLSID\{Your-CLSID}" or "REG_PATH=SOFTWARE\Microsoft\Windows\CurrentVersion\Run"
set "REG_PATH=Software\Classes\CLSID\{12345678-1234-1234-1234-1234567890AB}"

:: Define the type of the registry entry (under HKCU/HKU or loaded hive) to add or update
:: Example: "VALUE_TYPE=REG_SZ" or "VALUE_TYPE=REG_DWORD" or "VALUE_TYPE=REG_BINARY" or "VALUE_TYPE=REG_MULTI_SZ" or "VALUE_TYPE=REG_EXPAND_SZ" or "VALUE_TYPE=REG_NONE"
set "VALUE_TYPE=REG_SZ"

:: Define the value name to add or update
:: Example: "VALUE_NAME=" (empty, signifying "(Default)" key") or "VALUE_NAME=AppID" or "VALUE_NAME=SomeValue"
::set "VALUE_NAME=AppID"
set "VALUE_NAME="

:: Define the value data to add or update
:: Example: "VALUE_DATA={ABCD1234-EF56-7890-ABCD-12345678EFGH}" or "VALUE_DATA=" (signifying "no value") or "VALUE_DATA=C:\Windows\cmd.exe" or "VALUE_DATA=\"some funky.exe\" \"that needs actual double quotes\" \"or needing percent sign %%\""
set "VALUE_DATA={ABCD1234-EF56-7890-ABCD-12345678EFGH}"

:: Calling the actual function to modify the registry entries
CALL :fnMODIFYREGISTRY
if !iRETURNERRORLEVEL! GTR 0 ( GOTO :EOF )

:: -------------------------------------------------------
:: Repeating Segments
:: -------------------------------------------------------

:: If there are any other registry keys, please edit the batch file and remove the double colon "::" commenting line prefixes in the segment below and repeat the segments as many times as necessary

:: ==== start of repeatable segment ====
::set "REG_PATH=Software\Classes\CLSID\{12345678-1234-1234-1234-1234567890AB}"
::set "VALUE_TYPE=REG_SZ"
::set "VALUE_NAME=AppID"
::set "VALUE_DATA={ABCD1234-EF56-7890-ABCD-12345678EFGH}"
::CALL :fnMODIFYREGISTRY
::if !iRETURNERRORLEVEL! GTR 0 ( GOTO :EOF )
:: ==== end of repeatable segment ====


:: completion of processing
GOTO :fnPROCESSINGCOMPLETE

:fnVALUECHECKDONE
	if !VALUE_TYPE_CHECK!==N (
		echo Please set a valid VALUE_TYPE variable!
		set /a "iRETURNERRORLEVEL=1"
	)
	set VALUE_TYPE_CHECK=
GOTO :EOF

:fnMODIFYREGISTRY
	:: Temporary key name for mounting the Default user hive
	set "TEMP_HIVE_KEY=TempHive"
	
	echo.
	echo === Registry Modification Script Started ===
	echo REG_PATH: !REG_PATH!
	echo VALUE_TYPE: !VALUE_TYPE!
	echo VALUE_NAME: !VALUE_NAME!
	echo VALUE_DATA: !VALUE_DATA!
	echo.
	
	:: -------------------------------------------------------
	:: Sanity checks on REG_PATH, VALUE_TYPE
	:: -------------------------------------------------------
	echo Doing sanity checks...
	
	:: Check if REG_PATH was unedited from template
	echo !REG_PATH! | findstr /i "{12345678-1234-1234-1234-1234567890AB}" >nul
	if !errorlevel! EQU 0 (
		echo Please edit the batch file and change the REG_PATH variable!
		set /a "iRETURNERRORLEVEL=1"
		GOTO :EOF
	)
	
	:: Check if VALUE_TYPE is valid
	set "VALUE_TYPE_CHECK=N"
	for %%x in (REG_SZ REG_DWORD REG_BINARY REG_MULTI_SZ REG_EXPAND_SZ REG_NONE) do (
		if "!VALUE_TYPE!"=="%%x" (
			set "VALUE_TYPE_CHECK=Y"
		)
	)
	CALL :fnVALUECHECKDONE
	if !iRETURNERRORLEVEL! GTR 0 ( GOTO :EOF )

	:: Check if VALUE_DATA was unedited from template
	echo !VALUE_DATA! | findstr /i "{ABCD1234-EF56-7890-ABCD-12345678EFGH}" >nul
	if !errorlevel! EQU 0 (
		echo Please edit the batch file and change the VALUE_DATA variable!
		set /a "iRETURNERRORLEVEL=1"
		GOTO :EOF
	)

	
	:: -------------------------------------------------------
	:: Detect if REG_PATH targets per-user CLASSES
	:: -------------------------------------------------------
	echo Checking if REG_PATH includes 'Software\Classes\CLSID'...
	
	echo !REG_PATH! | findstr /i "Software\\Classes\\CLSID" >nul
	if !errorlevel! EQU 0 (
		set "TARGET_CLASSES_HIVES=1"
		echo Detected: Targeting per-user _CLASSES hives under HKU.
	) else (
		set "TARGET_CLASSES_HIVES=0"
		echo Not detected: Targeting regular HKU user SIDs.
	)
	
	
	:: -------------------------------------------------------
	:: Step 1: Modify Default User Hive (NTUSER.DAT)
	:: -------------------------------------------------------
	echo.
	echo [1/3] Modifying Default User Hive (NTUSER.DAT)...
	
	:: Load the default user's NTUSER.DAT
	reg load HKU\!TEMP_HIVE_KEY! "C:\Users\Default\NTUSER.DAT" >nul 2>&1
	if !errorlevel! NEQ 0 (
		echo [ERROR] Failed to load Default user hive. Ensure this script is run as Administrator.
		set /a "iRETURNERRORLEVEL=1"
		GOTO :EOF
	)
	
	:: Add or update the registry key and value
	reg add "HKU\!TEMP_HIVE_KEY!\!REG_PATH!" /f >nul
	if "!VALUE_NAME!."=="." (
		reg add "HKU\!TEMP_HIVE_KEY!\!REG_PATH!" /ve /t !VALUE_TYPE! /d "!VALUE_DATA!" /f
	) else (
		reg add "HKU\!TEMP_HIVE_KEY!\!REG_PATH!" /v "!VALUE_NAME!" /t !VALUE_TYPE! /d "!VALUE_DATA!" /f
	)
	
	:: Unload the hive
	reg unload HKU\!TEMP_HIVE_KEY! >nul
	echo Done modifying Default user hive.
	
	
	:: -------------------------------------------------------
	:: Step 2: Modify Relevant HKU User Hives
	:: -------------------------------------------------------
	echo.
	echo [2/3] Modifying applicable HKU user hives...
	
	for /f "tokens=*" %%U in ('reg query HKU') do (
		set "USER_HIVE=%%U"
		set "MODIFY_HIVE=0"
	
		:: Remove trailing backslash (if any)
		if "!USER_HIVE:~-1!"=="\" set "USER_HIVE=!USER_HIVE:~0,-1!"
	
		:: Determine if this hive should be modified
		echo !USER_HIVE! | findstr /R "S-1-5-21-.*_CLASSES$" >nul
		if !errorlevel! EQU 0 (
			if !TARGET_CLASSES_HIVES!==1 (
				set MODIFY_HIVE=1
			)
		) else (
			echo !USER_HIVE! | findstr /R "S-1-5-21-" >nul
			if !errorlevel! EQU 0 (
				if !TARGET_CLASSES_HIVES!==0 (
					set MODIFY_HIVE=1
				)
			)
		)
	
		:: If eligible, modify the hive
		if !MODIFY_HIVE!==1 (
			echo Modifying: !USER_HIVE!\!REG_PATH!
			reg add "!USER_HIVE!\!REG_PATH!" /f >nul
			if "!VALUE_NAME!."=="." (
				reg add "!USER_HIVE!\!REG_PATH!" /ve /t !VALUE_TYPE! /d "!VALUE_DATA!" /f
			) else (
				reg add "!USER_HIVE!\!REG_PATH!" /v "!VALUE_NAME!" /t !VALUE_TYPE! /d "!VALUE_DATA!" /f
			)
		)
	)
	
	echo Done modifying HKU user hives.
	
	
	:: -------------------------------------------------------
	:: Step 3: Complete
	:: -------------------------------------------------------
	echo.
	echo [3/3] Registry modifications completed.
GOTO :EOF

:fnPROCESSINGCOMPLETE

endlocal
exit /b !iRETURNERRORLEVEL!
