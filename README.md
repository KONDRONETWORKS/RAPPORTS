# RAPPORTS
Rapport de Projets

# 🚀 Mode Opératoire Complet - Déploiement DuoAuthCore

## 📋 **PRÉREQUIS**

### 1. **Vérifications initiales**
```bash
# Vérifier IIS
Get-WindowsFeature -Name Web-*

# Vérifier .NET Core Runtime
dotnet --list-runtimes

# Vérifier le Hosting Bundle
Get-ItemProperty "HKLM:\SOFTWARE\Microsoft\ASP.NET Core\*"
```

### 2. **Installation des composants manquants**
```powershell
# Installer le Hosting Bundle ASP.NET Core
# Télécharger depuis : https://dotnet.microsoft.com/download/dotnet-core/8.0

# Activer les fonctionnalités IIS
Enable-WindowsOptionalFeature -Online -FeatureName IIS-WebServerRole
Enable-WindowsOptionalFeature -Online -FeatureName IIS-WebServer
Enable-WindowsOptionalFeature -Online -FeatureName IIS-ASPNET45
```

## 🏗️ **PHASE 1 : PRÉPARATION DE L'ENVIRONNEMENT**

### **1.1 Structure des dossiers**
```powershell
# Créer l'arborescence
$basePath = "C:\inetpub\wwwroot\sigec4"
New-Item -ItemType Directory -Path "$basePath" -Force
New-Item -ItemType Directory -Path "$basePath\duoauth" -Force
New-Item -ItemType Directory -Path "$basePath\duoauth\logs" -Force
New-Item -ItemType Directory -Path "$basePath\auth_tokens" -Force
New-Item -ItemType Directory -Path "$basePath\logs" -Force

# Vérification
tree $basePath /F
```

### **1.2 Configuration DNS et SSL**
```powershell
# Vérifier le binding SSL
Get-WebBinding | Where-Object {$_.protocol -eq "https"}

# Si besoin, ajouter le binding SSL
New-WebBinding -Name "SIGEC4" -Protocol https -Port 2012 -IPAddress "*" -SslFlags 1
```

## 🔧 **PHASE 2 : CONFIGURATION IIS**

### **2.1 Script de configuration IIS complet**
```powershell
# Configure-IIS-DuoAuth.ps1
param(
    [string]$SiteName = "SIGEC4",
    [string]$AppName = "duoauth",
    [string]$SitePath = "C:\inetpub\wwwroot\sigec4",
    [string]$DuoAuthPath = "C:\inetpub\wwwroot\sigec4\duoauth",
    [string]$Domain = "test.cafecacao.ci:2012"
)

Write-Host "=== CONFIGURATION IIS DUOAUTHCORE ===" -ForegroundColor Cyan

# Import module IIS
Import-Module WebAdministration

try {
    # 1. Créer le pool d'applications
    $appPoolName = "SIGEC4AppPool"
    
    if (Test-Path "IIS:\AppPools\$appPoolName") {
        Write-Host "✓ Pool $appPoolName existe déjà" -ForegroundColor Yellow
        Remove-WebAppPool -Name $appPoolName
        Write-Host "✓ Pool existant supprimé" -ForegroundColor Green
    }
    
    Write-Host "Création du pool $appPoolName..." -ForegroundColor Yellow
    New-WebAppPool -Name $appPoolName
    
    # Configuration .NET Core
    Set-ItemProperty "IIS:\AppPools\$appPoolName" managedRuntimeVersion ""
    Set-ItemProperty "IIS:\AppPools\$appPoolName" managedPipelineMode 0
    Set-ItemProperty "IIS:\AppPools\$appPoolName" processModel.identityType 4
    Set-ItemProperty "IIS:\AppPools\$appPoolName" recycling.periodicRestart.time "00:00:00"
    Set-ItemProperty "IIS:\AppPools\$appPoolName" processModel.idleTimeout "00:00:00"
    
    Write-Host "✓ Pool $appPoolName configuré pour .NET Core" -ForegroundColor Green

    # 2. Créer ou mettre à jour le site
    if (Test-Path "IIS:\Sites\$SiteName") {
        Write-Host "✓ Site $SiteName existe déjà" -ForegroundColor Yellow
        Stop-Website -Name $SiteName
        Remove-Website -Name $SiteName
        Write-Host "✓ Site existant supprimé" -ForegroundColor Green
    }
    
    Write-Host "Création du site $SiteName..." -ForegroundColor Yellow
    New-Website -Name $SiteName -PhysicalPath $SitePath -ApplicationPool $appPoolName -BindingInformation "*:2012:$Domain"
    
    Write-Host "✓ Site $SiteName créé" -ForegroundColor Green

    # 3. Créer l'application DuoAuthCore
    if (Get-WebApplication -Site $SiteName -Name $AppName -ErrorAction SilentlyContinue) {
        Write-Host "✓ Application $AppName existe déjà" -ForegroundColor Yellow
        Remove-WebApplication -Site $SiteName -Name $AppName
        Write-Host "✓ Application existante supprimée" -ForegroundColor Green
    }
    
    Write-Host "Création de l'application $AppName..." -ForegroundColor Yellow
    New-WebApplication -Site $SiteName -Name $AppName -PhysicalPath $DuoAuthPath -ApplicationPool $appPoolName
    
    Write-Host "✓ Application $AppName créée" -ForegroundColor Green

    # 4. Configurer les permissions
    Write-Host "Configuration des permissions..." -ForegroundColor Yellow
    
    # Reset des permissions
    icacls $SitePath /reset /T
    icacls $SitePath /grant "IIS_IUSRS:(OI)(CI)(RX)" /T
    icacls $DuoAuthPath /grant "IIS AppPool\$appPoolName:(OI)(CI)(RX)" /T
    
    # Permissions écriture logs
    $logsPath = Join-Path $DuoAuthPath "logs"
    icacls $logsPath /grant "IIS AppPool\$appPoolName:(OI)(CI)(W,R)" /T
    
    # Permissions tokens partagés
    $tokensPath = Join-Path $SitePath "auth_tokens"
    icacls $tokensPath /grant "IIS AppPool\$appPoolName:(OI)(CI)(W,R)" /T
    icacls $tokensPath /grant "IIS_IUSRS:(OI)(CI)(W,R)" /T
    
    Write-Host "✓ Permissions configurées" -ForegroundColor Green

    # 5. Démarrer les services
    Start-WebAppPool -Name $appPoolName
    Start-Website -Name $SiteName
    
    Write-Host "✓ Services démarrés" -ForegroundColor Green

    # 6. Vérification finale
    Start-Sleep -Seconds 3
    
    $poolState = (Get-WebAppPoolState -Name $appPoolName).Value
    $siteState = (Get-WebsiteState -Name $SiteName).Value
    
    Write-Host "`n=== VÉRIFICATION FINALE ===" -ForegroundColor Cyan
    Write-Host "Pool $appPoolName : $poolState" -ForegroundColor $(if($poolState -eq "Started"){"Green"}else{"Red"})
    Write-Host "Site $SiteName : $siteState" -ForegroundColor $(if($siteState -eq "Started"){"Green"}else{"Red"})
    
    if ($poolState -eq "Started" -and $siteState -eq "Started") {
        Write-Host "`n✅ CONFIGURATION IIS TERMINÉE AVEC SUCCÈS" -ForegroundColor Green
        Write-Host "URL Application : https://$Domain" -ForegroundColor Cyan
        Write-Host "URL DuoAuthCore : https://$Domain/duoauth" -ForegroundColor Cyan
        Write-Host "URL Health Check : https://$Domain/duoauth/status" -ForegroundColor Cyan
    } else {
        Write-Host "`n❌ ERREUR DANS LA CONFIGURATION" -ForegroundColor Red
    }

} catch {
    Write-Host "❌ ERREUR : $($_.Exception.Message)" -ForegroundColor Red
}
```

**Exécution :**
```powershell
.\Configure-IIS-DuoAuth.ps1 -Domain "test.cafecacao.ci:2012"
```

## 📄 **PHASE 3 : FICHIERS DE CONFIGURATION**

### **3.1 web.config racine**
**C:\inetpub\wwwroot\sigec4\web.config**
```xml
<?xml version="1.0" encoding="utf-8"?>
<configuration>
  <system.webServer>
    <!-- Handlers -->
    <handlers>
      <add name="ASPClassic" 
           path="*.asp" 
           verb="GET,HEAD,POST" 
           modules="IsapiModule" 
           scriptProcessor="%windir%\system32\inetsrv\asp.dll" 
           resourceType="File" />
      <remove name="ExtensionlessUrlHandler-Integrated-4.0" />
      <remove name="OPTIONSVerbHandler" />
      <remove name="TRACEVerbHandler" />
    </handlers>
    
    <!-- Modules -->
    <modules runAllManagedModulesForAllRequests="false">
      <remove name="WebDAVModule" />
      <remove name="AspNetCoreModuleV2" />
    </modules>
    
    <!-- Sécurité -->
    <security>
      <requestFiltering>
        <requestLimits maxAllowedContentLength="52428800" />
        <fileExtensions allowUnlisted="true">
          <add fileExtension=".asp" allowed="true" />
        </fileExtensions>
      </requestFiltering>
    </security>
    
    <!-- Réécriture d'URL -->
    <rewrite>
      <rules>
        <rule name="DuoAuth Health" stopProcessing="true">
          <match url="^duo-health$" />
          <action type="Rewrite" url="/duoauth/status" />
        </rule>
      </rules>
    </rewrite>
  </system.webServer>
  
  <!-- Configuration ASP -->
  <system.web>
    <compilation debug="false" targetFramework="4.0" />
    <sessionState mode="InProc" timeout="60" cookieless="false" />
    <httpRuntime maxRequestLength="51200" executionTimeout="3600" />
    <customErrors mode="Off" />
  </system.web>
  
  <appSettings>
    <add key="DuoAuthBaseUrl" value="https://test.cafecacao.ci:2012/duoauth" />
  </appSettings>
</configuration>
```

### **3.2 web.config DuoAuthCore**
**C:\inetpub\wwwroot\sigec4\duoauth\web.config**
```xml
<?xml version="1.0" encoding="utf-8"?>
<configuration>
  <location path="." inheritInChildApplications="false">
    <system.webServer>
      <handlers>
        <add name="aspNetCore" path="*" verb="*" modules="AspNetCoreModuleV2" resourceType="Unspecified" />
      </handlers>
      
      <aspNetCore 
        processPath="dotnet" 
        arguments=".\DuoAuthCore.dll" 
        stdoutLogEnabled="true"
        stdoutLogFile=".\logs\stdout"
        stdoutLogFileSizeLimit="10000000"
        stdoutFileRetentionCount="10"
        hostingModel="inprocess">
        
        <environmentVariables>
          <environmentVariable name="ASPNETCORE_ENVIRONMENT" value="Production" />
          <environmentVariable name="ASPNETCORE_URLS" value="" />
          <environmentVariable name="ContentRoot" value="C:\inetpub\wwwroot\sigec4\duoauth" />
          <environmentVariable name="ASPNETCORE_CONTENTROOT" value="C:\inetpub\wwwroot\sigec4\duoauth" />
        </environmentVariables>
      </aspNetCore>
      
      <!-- Sécurité -->
      <security>
        <requestFiltering>
          <hiddenSegments>
            <add segment="logs" />
            <add segment="bin" />
            <add segment="appsettings.json" />
          </hiddenSegments>
          <fileExtensions>
            <add fileExtension=".dll" allowed="false" />
            <add fileExtension=".pdb" allowed="false" />
            <add fileExtension=".config" allowed="false" />
          </fileExtensions>
        </requestFiltering>
      </security>
      
      <!-- Headers de sécurité -->
      <httpProtocol>
        <customHeaders>
          <add name="X-Content-Type-Options" value="nosniff" />
          <add name="X-Frame-Options" value="SAMEORIGIN" />
          <add name="X-XSS-Protection" value="1; mode=block" />
          <add name="Referrer-Policy" value="strict-origin-when-cross-origin" />
        </customHeaders>
      </httpProtocol>
    </system.webServer>
  </location>
</configuration>
```

## 🚀 **PHASE 4 : DÉPLOIEMENT DE L'APPLICATION**

### **4.1 Publication de DuoAuthCore**
```powershell
# Script de publication
# Publish-DuoAuthCore.ps1
param(
    [string]$ProjectPath = "C:\Dev\DuoAuthCore",
    [string]$PublishPath = "C:\inetpub\wwwroot\sigec4\duoauth"
)

Write-Host "=== PUBLICATION DUOAUTHCORE ===" -ForegroundColor Cyan

try {
    # Nettoyer le dossier de publication
    if (Test-Path $PublishPath) {
        Remove-Item "$PublishPath\*" -Recurse -Force
        Write-Host "✓ Dossier de publication nettoyé" -ForegroundColor Green
    }
    
    # Publication
    Write-Host "Publication de l'application..." -ForegroundColor Yellow
    dotnet publish $ProjectPath -c Release -o $PublishPath --self-contained false
    
    if ($LASTEXITCODE -eq 0) {
        Write-Host "✓ Application publiée avec succès" -ForegroundColor Green
        
        # Vérifier les fichiers
        $dllPath = Join-Path $PublishPath "DuoAuthCore.dll"
        if (Test-Path $dllPath) {
            Write-Host "✓ Fichier DLL trouvé : $dllPath" -ForegroundColor Green
        } else {
            Write-Host "❌ Fichier DLL manquant" -ForegroundColor Red
            exit 1
        }
    } else {
        Write-Host "❌ Erreur lors de la publication" -ForegroundColor Red
        exit 1
    }
    
} catch {
    Write-Host "❌ ERREUR : $($_.Exception.Message)" -ForegroundColor Red
    exit 1
}

Write-Host "✅ PUBLICATION TERMINÉE" -ForegroundColor Green
```

### **4.2 Configuration appsettings.json**
**C:\inetpub\wwwroot\sigec4\duoauth\appsettings.json**
```json
{
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft.AspNetCore": "Warning"
    },
    "File": {
      "Path": "C:\\inetpub\\wwwroot\\sigec4\\logs\\duoauth-{Date}.txt",
      "FileSizeLimitBytes": 10485760,
      "RetainedFileCountLimit": 5
    }
  },
  "Duo": {
    "ClientId": "DIXXXXXXXXXXXXXXXXXX",
    "ClientSecret": "XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX",
    "ApiHost": "api-XXXXX.duosecurity.com",
    "RedirectUri": "https://test.cafecacao.ci:2012/duoauth/callback"
  },
  "SharedAuth": {
    "TokenTimeoutMinutes": 5,
    "TokenCleanupIntervalMinutes": 10
  },
  "AllowedHosts": "test.cafecacao.ci",
  "Kestrel": {
    "Endpoints": {
      "Http": {
        "Url": "https://test.cafecacao.ci:2012"
      }
    }
  }
}
```

## 🔧 **PHASE 5 : CONFIGURATION ASP CLASSIC**

### **5.1 login.asp modifié**
```asp
<%
'=== CONFIGURATION ===
Dim DuoAuthBaseUrl
DuoAuthBaseUrl = "https://test.cafecacao.ci:2012/duoauth"

'=== TRAITEMENT DU LOGIN ===
If Request.ServerVariables("REQUEST_METHOD") = "POST" Then
    Dim username, password
    username = Trim(Request.Form("username"))
    password = Trim(Request.Form("password"))
    
    ' Validation des credentials
    If ValidateUserCredentials(username, password) Then
        ' Préparer la redirection MFA
        Session("PreAuthUser") = username
        Session("PreAuthTime") = Now()
        Session("PreAuthIP") = Request.ServerVariables("REMOTE_ADDR")
        
        ' URL de retour après MFA
        Dim returnUrl
        returnUrl = "/login_complete.asp"
        
        ' Redirection vers DuoAuthCore
        Response.Redirect DuoAuthBaseUrl & "/init?username=" & Server.URLEncode(username) & "&returnUrl=" & Server.URLEncode(returnUrl)
    Else
        ' Journaliser l'échec
        LogAuthenticationAttempt username, Request.ServerVariables("REMOTE_ADDR"), "FAILED"
        Response.Write("<div class='alert alert-error'>Identifiants invalides</div>")
    End If
End If
%>

<!DOCTYPE html>
<html lang="fr">
<head>
    <meta charset="UTF-8">
    <title>Connexion SIGEC4 - MFA</title>
    <style>
        .login-container { max-width: 400px; margin: 100px auto; padding: 20px; }
        .alert { padding: 10px; margin: 10px 0; border-radius: 4px; }
        .alert-error { background: #f8d7da; color: #721c24; }
    </style>
</head>
<body>
    <div class="login-container">
        <h2>Connexion SIGEC4</h2>
        <form method="post">
            <div>
                <label>Nom d'utilisateur:</label>
                <input type="text" name="username" required>
            </div>
            <div>
                <label>Mot de passe:</label>
                <input type="password" name="password" required>
            </div>
            <button type="submit">Se connecter</button>
        </form>
        
        <div style="margin-top: 20px; font-size: 12px; color: #666;">
            <p>⚠️ Après validation de vos identifiants, vous serez redirigé vers l'authentification MFA Duo.</p>
        </div>
    </div>
</body>
</html>

<%
'=== FONCTIONS ===
Function ValidateUserCredentials(username, password)
    On Error Resume Next
    
    ' Implémentez votre logique de validation
    ' Exemple basique - À ADAPTER avec votre base de données
    
    If username = "" Or password = "" Then
        ValidateUserCredentials = False
        Exit Function
    End If
    
    ' Simulation de validation
    If username = "demo" And password = "demo" Then
        ValidateUserCredentials = True
        LogAuthenticationAttempt username, Request.ServerVariables("REMOTE_ADDR"), "SUCCESS"
    Else
        ValidateUserCredentials = False
    End If
End Function

Sub LogAuthenticationAttempt(username, ipAddress, status)
    Dim logPath, logEntry
    logPath = Server.MapPath("../logs/auth_attempts.log")
    logEntry = Now() & " | USER: " & username & " | IP: " & ipAddress & " | STATUS: " & status & vbCrLf
    
    Dim fso, file
    Set fso = Server.CreateObject("Scripting.FileSystemObject")
    On Error Resume Next
    Set file = fso.OpenTextFile(logPath, 8, True) ' 8 = ForAppending
    If Err.Number = 0 Then
        file.Write(logEntry)
        file.Close()
    End If
    On Error GoTo 0
End Sub
%>
```

### **5.2 login_complete.asp**
```asp
<%
'=== VALIDATION DU CALLBACK MFA ===
Dim authToken, username, duoSuccess
authToken = Request.QueryString("auth_token")
username = Request.QueryString("username")
duoSuccess = Request.QueryString("duo_success")

If duoSuccess = "true" And authToken <> "" And username <> "" Then
    ' Validation du token MFA
    If ValidateDuoAuthToken(authToken, username) Then
        ' Authentification MFA réussie
        Session("Authenticated") = True
        Session("Username") = username
        Session("LoginTime") = Now()
        Session("MFAVerified") = True
        Session("UserIP") = Request.ServerVariables("REMOTE_ADDR")
        
        ' Journaliser la connexion
        LogSuccessfulLogin username
        
        ' Redirection vers le dashboard
        Response.Redirect "/dashboard.asp"
    Else
        ' Échec de validation MFA
        LogFailedLogin username, "MFA_TOKEN_INVALID"
        Session.Abandon()
        Response.Redirect "/login.asp?error=mfa_invalid"
    End If
Else
    ' Accès direct non autorisé
    LogSecurityEvent "DIRECT_ACCESS_ATTEMPT", Request.ServerVariables("REMOTE_ADDR")
    Response.Redirect "/login.asp?error=invalid_access"
End If

'=== FONCTIONS ===
Function ValidateDuoAuthToken(token, expectedUsername)
    On Error Resume Next
    
    Dim tokenPath, tokenData, parts, tokenUser, tokenTime
    tokenPath = Server.MapPath("../auth_tokens/" & token & ".token")
    
    Dim fso
    Set fso = Server.CreateObject("Scripting.FileSystemObject")
    
    If fso.FileExists(tokenPath) Then
        ' Lire le token
        Dim file
        Set file = fso.OpenTextFile(tokenPath, 1) ' 1 = ForReading
        tokenData = file.ReadAll()
        file.Close()
        
        ' Supprimer le token (usage unique)
        fso.DeleteFile(tokenPath)
        
        ' Parser les données
        parts = Split(tokenData, "|")
        If UBound(parts) = 1 Then
            tokenUser = parts(0)
            tokenTime = CDate(parts(1))
            
            ' Validation
            If tokenUser = expectedUsername And DateDiff("n", tokenTime, Now()) < 5 Then
                ValidateDuoAuthToken = True
                Exit Function
            End If
        End If
    End If
    
    ValidateDuoAuthToken = False
End Function

Sub LogSuccessfulLogin(username)
    Dim logPath, logEntry
    logPath = Server.MapPath("../logs/user_logins.log")
    logEntry = Now() & " | USER: " & username & " | IP: " & Request.ServerVariables("REMOTE_ADDR") & " | STATUS: SUCCESS" & vbCrLf
    
    Dim fso, file
    Set fso = Server.CreateObject("Scripting.FileSystemObject")
    On Error Resume Next
    Set file = fso.OpenTextFile(logPath, 8, True)
    If Err.Number = 0 Then
        file.Write(logEntry)
        file.Close()
    End If
    On Error GoTo 0
End Sub

Sub LogFailedLogin(username, reason)
    Dim logPath, logEntry
    logPath = Server.MapPath("../logs/failed_logins.log")
    logEntry = Now() & " | USER: " & username & " | IP: " & Request.ServerVariables("REMOTE_ADDR") & " | REASON: " & reason & vbCrLf
    
    Dim fso, file
    Set fso = Server.CreateObject("Scripting.FileSystemObject")
    On Error Resume Next
    Set file = fso.OpenTextFile(logPath, 8, True)
    If Err.Number = 0 Then
        file.Write(logEntry)
        file.Close()
    End If
    On Error GoTo 0
End Sub

Sub LogSecurityEvent(eventType, ipAddress)
    Dim logPath, logEntry
    logPath = Server.MapPath("../logs/security_events.log")
    logEntry = Now() & " | EVENT: " & eventType & " | IP: " & ipAddress & vbCrLf
    
    Dim fso, file
    Set fso = Server.CreateObject("Scripting.FileSystemObject")
    On Error Resume Next
    Set file = fso.OpenTextFile(logPath, 8, True)
    If Err.Number = 0 Then
        file.Write(logEntry)
        file.Close()
    End If
    On Error GoTo 0
End Sub
%>
```

## ✅ **PHASE 6 : VÉRIFICATIONS FINALES**

### **6.1 Script de vérification**
```powershell
# Test-DuoAuthDeployment.ps1
param(
    [string]$Domain = "test.cafecacao.ci:2012"
)

Write-Host "=== VÉRIFICATION DU DÉPLOIEMENT ===" -ForegroundColor Cyan

# 1. Vérification IIS
Write-Host "`n1. VÉRIFICATION IIS..." -ForegroundColor Yellow
$poolState = (Get-WebAppPoolState -Name "SIGEC4AppPool").Value
$siteState = (Get-WebsiteState -Name "SIGEC4").Value

Write-Host "Pool SIGEC4AppPool: $poolState" -ForegroundColor $(if($poolState -eq "Started"){"Green"}else{"Red"})
Write-Host "Site SIGEC4: $siteState" -ForegroundColor $(if($siteState -eq "Started"){"Green"}else{"Red"})

# 2. Vérification fichiers
Write-Host "`n2. VÉRIFICATION FICHIERS..." -ForegroundColor Yellow
$filesToCheck = @(
    "C:\inetpub\wwwroot\sigec4\duoauth\DuoAuthCore.dll",
    "C:\inetpub\wwwroot\sigec4\duoauth\web.config",
    "C:\inetpub\wwwroot\sigec4\web.config",
    "C:\inetpub\wwwroot\sigec4\login.asp"
)

foreach ($file in $filesToCheck) {
    $exists = Test-Path $file
    Write-Host "$file : $(if($exists){"✓"}else{"❌"})" -ForegroundColor $(if($exists){"Green"}else{"Red"})
}

# 3. Test de connexion
Write-Host "`n3. TEST DE CONNEXION..." -ForegroundColor Yellow
try {
    $healthUrl = "https://$Domain/duoauth/status"
    $response = Invoke-WebRequest -Uri $healthUrl -UseBasicParsing -TimeoutSec 10
    Write-Host "Health Check: ✓ (Status: $($response.StatusCode))" -ForegroundColor Green
} catch {
    Write-Host "Health Check: ❌ ($($_.Exception.Message))" -ForegroundColor Red
}

# 4. Vérification logs
Write-Host "`n4. VÉRIFICATION LOGS..." -ForegroundColor Yellow
$logFiles = Get-ChildItem "C:\inetpub\wwwroot\sigec4\duoauth\logs" -ErrorAction SilentlyContinue
if ($logFiles) {
    Write-Host "Logs trouvés: $($logFiles.Count) fichiers" -ForegroundColor Green
    $latestLog = $logFiles | Sort-Object LastWriteTime | Select-Object -Last 1
    Write-Host "Dernier log: $($latestLog.Name)" -ForegroundColor Cyan
} else {
    Write-Host "Aucun log trouvé" -ForegroundColor Yellow
}

Write-Host "`n=== RÉSUMÉ ===" -ForegroundColor Cyan
if ($poolState -eq "Started" -and $siteState -eq "Started") {
    Write-Host "✅ DÉPLOIEMENT RÉUSSI" -ForegroundColor Green
    Write-Host "URL: https://$Domain" -ForegroundColor White
    Write-Host "DuoAuth: https://$Domain/duoauth" -ForegroundColor White
} else {
    Write-Host "❌ PROBLÈME DÉTECTÉ" -ForegroundColor Red
}
```

### **6.2 Commandes de test manuelles**
```powershell
# Test santé application
curl -k "https://test.cafecacao.ci:2012/duoauth/status"

# Vérifier les processus IIS
Get-Process w3wp

# Vérifier les logs en temps réel
Get-Content "C:\inetpub\wwwroot\sigec4\duoauth\logs\stdout_*.log" -Wait -Tail 10
```

## 🚨 **PROCÉDURE DE DÉPANNAGE**

### **Erreurs courantes et solutions :**
```powershell
# 1. Erreur 500.19 - Reconfigurer web.config
.\Configure-IIS-DuoAuth.ps1

# 2. Erreur 502.5 - Redémarrer le pool
Restart-WebAppPool -Name "SIGEC4AppPool"

# 3. Permissions - Réappliquer
icacls "C:\inetpub\wwwroot\sigec4" /grant "IIS_IUSRS:(OI)(CI)(RX)" /T

# 4. Redémarrage complet IIS
iisreset
```

## 📊 **MONITORING EN PRODUCTION**

### **Script de monitoring**
```powershell
# Monitor-DuoAuth.ps1
while ($true) {
    Clear-Host
    Write-Host "=== MONITORING DUOAUTHCORE ===" -ForegroundColor Cyan
    Write-Host "Heure: $(Get-Date)" -ForegroundColor White
    
    # Statut IIS
    $poolState = (Get-WebAppPoolState -Name "SIGEC4AppPool").Value
    $siteState = (Get-WebsiteState -Name "SIGEC4").Value
    
    Write-Host "`nIIS Status:" -ForegroundColor Yellow
    Write-Host "Pool: $poolState" -ForegroundColor $(if($poolState -eq "Started"){"Green"}else{"Red"})
    Write-Host "Site: $siteState" -ForegroundColor $(if($siteState -eq "Started"){"Green"}else{"Red"})
    
    # Logs récents
    $logFile = Get-ChildItem "C:\inetpub\wwwroot\sigec4\duoauth\logs\stdout_*.log" | Sort-Object LastWriteTime | Select-Object -Last 1
    if ($logFile) {
        Write-Host "`nDerniers logs:" -ForegroundColor Yellow
        Get-Content $logFile.FullName -Tail 5 | ForEach-Object {
            Write-Host "  $_" -ForegroundColor Gray
        }
    }
    
    Start-Sleep -Seconds 10
}
```

Ce mode opératoire complet vous guide pas à pas du déploiement à la vérification en production. Tous les liens ont été corrigés pour utiliser `https://test.cafecacao.ci:2012/`.
