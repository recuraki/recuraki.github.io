---
layout: post
title:  "Power ShellによるWindows updateの確認"
date:   2021-10-15T00:00:00+09:00
author: Akira Kanai
categories: diary
cover:  "/assets/instacode.png"
---

# 20211015: PSによるWindows updateの確認

```
#情報取得スクリプト
function Check-OsUpdate{
    $osUpdate = @{}
    $windowsUpdate = New-Object -ComObject "Microsoft.Update.AutoUpdate"
    $osUpdate["LastUpdateCheck"] = $windowsUpdate.Results.LastSearchSuccessDate.ToLocaltime()
    $osUpdate["LastUpdateInstall"] = $windowsUpdate.Results.LastInstallationSuccessDate.ToLocaltime()
    $osUpdate["LastOSReboot"] = [Management.ManagementDateTimeConverter]::ToDateTime((Get-WmiObject Win32_OperatingSystem).LastBootUpTime)
    return $osUpdate
}
Check-OsUpdate
```

結果

```
Check-OsUpdate

Name                           Value
----                           -----
LastUpdateInstall              2021/10/14 2:53:38
LastUpdateCheck                2021/10/15 10:14:50
LastOSReboot                   2021/10/13 11:11:18
```
