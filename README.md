# Script-PowerShell-Monitoring-Sistem
Script ini bertujuan untuk memonitor kondisi sistem Windows setiap jam, mendeteksi kondisi kritis pada CPU, memori, disk, dan layanan penting, serta melakukan logging dan notifikasi email jika ada masalah.

# Config
$LogFile = "C:\Monitoring\SystemLog.txt"
$CriticalServices = @("W32Time","WinDefend","EventLog")
$SMTPServer = "smtp.example.com"
$SMTPFrom = "monitor@example.com"
$SMTPTo = "admin@example.com"
$SMTPSubject = "SYSTEM ALERT"
$AlertTriggered = $false
$CPUThreshold = 80
$CPUDuration = 5 # minutes

# Function to log message
function Write-Log {
    param([string]$Message)
    $TimeStamp = Get-Date -Format "yyyy-MM-dd HH:mm:ss"
    "$TimeStamp - $Message" | Out-File -Append -FilePath $LogFile
}

# Function to send email alert
function Send-AlertEmail {
    param([string]$Body)
    Send-MailMessage -SmtpServer $SMTPServer -From $SMTPFrom -To $SMTPTo -Subject $SMTPSubject -Body $Body
}

# CPU Monitoring
function Check-CPUUsage {
    $HighCount = 0
    for ($i=0; $i -lt $CPUDuration; $i++) {
        $CPU = Get-Counter '\Processor(_Total)\% Processor Time'
        $Usage = [math]::Round($CPU.CounterSamples.CookedValue,2)
        Write-Log "CPU Usage: $Usage%"
        if ($Usage -gt $CPUThreshold) { $HighCount++ }
        Start-Sleep -Seconds 60
    }
    if ($HighCount -eq $CPUDuration) {
        Write-Log "ALERT: CPU usage > $CPUThreshold% for $CPUDuration minutes"
        $GLOBALS:AlertTriggered = $true
    }
}

# Memory Monitoring
function Check-Memory {
    $FreeMem = (Get-CimInstance Win32_OperatingSystem).FreePhysicalMemory / 1024
    Write-Log "Available Memory: $([math]::Round($FreeMem,2)) MB"
    if ($FreeMem -lt 1024) {
        Write-Log "ALERT: Memory available < 1GB"
        $GLOBALS:AlertTriggered = $true
    }
}

# Disk Monitoring
function Check-DiskSpace {
    Get-PSDrive -PSProvider FileSystem | ForEach-Object {
        $FreePercent = [math]::Round(($_.Free / $_.Used*100),2)
        Write-Log "Drive $($_.Name): $FreePercent% free"
        if ($FreePercent -lt 10) {
            Write-Log "ALERT: Drive $($_.Name) free space < 10%"
            $GLOBALS:AlertTriggered = $true
        }
    }
}

# Services Monitoring
function Check-Services {
    foreach ($svc in $CriticalServices) {
        $status = (Get-Service $svc).Status
        Write-Log "Service $svc status: $status"
        if ($status -ne "Running") {
            Write-Log "ALERT: Service $svc stopped"
            $GLOBALS:AlertTriggered = $true
        }
    }
}

# Main Execution
Check-CPUUsage
Check-Memory
Check-DiskSpace
Check-Services

if ($AlertTriggered) {
    Send-AlertEmail -Body "Sistem mendeteksi satu atau lebih kondisi kritis. Periksa log di $LogFile"
}
