# mytest
everynone
# mytest
This is a submission for my course assignment.
All credits belong to the original author.
You can search "pavise" on GitHub; the original repository will appear at the top.
[CmdletBinding()]
param([string]$InstallPath = '', [switch]$Uninstall)

# 将路径转换为RTSS可执行文件完整路径
function ConvertTo-RtssExecutablePath {
    param([string]$Path)
    if ([string]::IsNullOrWhiteSpace($Path)) { return '' }
    try {
        $candidate = [Environment]::ExpandEnvironmentVariables($Path.Trim().Trim('"'))
        if ($candidate -notmatch '^[A-Za-z]:\\' -or $candidate.IndexOfAny([char[]]"`r`n`"") -ge 0) { return '' }
        if ([IO.Path]::GetExtension($candidate) -ine '.exe') { $candidate = [IO.Path]::Combine($candidate, 'RTSS.exe') }
        if ([IO.Path]::GetFileName($candidate) -ine 'RTSS.exe') { return '' }
        $candidate = [IO.Path]::GetFullPath($candidate)
        if ([IO.Path]::GetDirectoryName($candidate).TrimEnd('\') -ieq [IO.Path]::GetPathRoot($candidate).TrimEnd('\')) { return '' }
        return $candidate
    } catch { return '' }
}

# 读取注册表RTSS卸载注册信息
function Get-RtssUninstallRegistrations {
    $views = @([Microsoft.Win32.RegistryView]::Registry32)
    if ([Environment]::Is64BitOperatingSystem) { $views = @([Microsoft.Win32.RegistryView]::Registry64, [Microsoft.Win32.RegistryView]::Registry32) }
    foreach ($hive in @([Microsoft.Win32.RegistryHive]::LocalMachine, [Microsoft.Win32.RegistryHive]::CurrentUser)) {
        foreach ($view in $views) {
            $root = $null; $key = $null
            try {
                $root = [Microsoft.Win32.RegistryKey]::OpenBaseKey($hive, $view)
                $key = $root.OpenSubKey('SOFTWARE\Microsoft\Windows\CurrentVersion\Uninstall\RTSS', $false)
                if ($null -ne $key) {
                    [pscustomobject]@{ 
                        KeyName = 'RTSS'; 
                        DisplayName = $key.GetValue('DisplayName'); 
                        Publisher = $key.GetValue('Publisher');
                        InstallLocation = $key.GetValue('InstallLocation'); 
                        UninstallString = $key.GetValue('UninstallString'); 
                        DisplayVersion = $key.GetValue('DisplayVersion') 
                    }
                }
            } finally { if ($null -ne $key) { $key.Dispose() }; if ($null -ne $root) { $root.Dispose() } }
        }
    }
}

# 获取正在运行的RTSS进程
function Get-RtssActiveProcesses {
    param([string]$Executable)
    foreach ($process in @(Get-Process -Name RTSS -ErrorAction SilentlyContinue)) {
        try { 
            if ($process.MainModule.FileName -ieq $Executable) { 
                [pscustomobject]@{ ProcessId = $process.Id; Path = $Executable } 
            } 
        }
        finally { $process.Dispose() }
    }
}

# 判断是否管理员权限
function Test-RtssAdministrator {
    $identity = [Security.Principal.WindowsIdentity]::GetCurrent()
    try { 
        return [Security.Principal.WindowsPrincipal]::new($identity).IsInRole([Security.Principal.WindowsBuiltInRole]::Administrator) 
    }
    finally { $identity.Dispose() }
}

# 主入口
function Invoke-RtssUninstallCommand {
    param([string]$InstallPath = '', [switch]$Uninstall)
    Write-Host "RTSS Uninstall Script Start"
    $registrations = @(Get-RtssUninstallRegistrations)
    if($registrations.Count -eq 0){
        Write-Host "No RTSS registry uninstall entry found."
        return
    }
    foreach($reg in $registrations){
        Write-Host "Found RTSS: $($reg.DisplayName) | Path: $($reg.InstallLocation)"
    }

    if(-not $Uninstall){
        Write-Host "Preview mode. Add -Uninstall parameter to perform actual uninstall."
        return
    }

    if(-not (Test-RtssAdministrator)){
        Write-Error "Must run as Administrator!"
        return
    }

    $runningProcs = @(Get-RtssActiveProcesses -Executable (ConvertTo-RtssExecutablePath $registrations[0].InstallLocation))
    if($runningProcs.Count -gt 0){
        Write-Warning "RTSS is running, please close RTSS before uninstall."
    }
}

# 脚本执行入口
if ($MyInvocation.InvocationName -ne '.') {
    Set-StrictMode -Version Latest
    $ErrorActionPreference = 'Stop'
    try { 
        Invoke-RtssUninstallCommand -InstallPath $InstallPath -Uninstall:$Uninstall 
    }
    catch { 
        Write-Error -Message $_.Exception.Message -ErrorAction Continue; 
        exit 1 
    }
}
