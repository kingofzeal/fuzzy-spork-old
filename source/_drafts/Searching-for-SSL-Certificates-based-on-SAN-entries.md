---
title: Searching for SSL Certificates based on SAN entries
tags:
---

- Lets Encrypt
- SAN Certificates
- Octopus Deploy
  - https://octopusdeploy.uservoice.com/forums/170787-general/suggestions/15045072-support-letsncrypt-for-octopus-certificates


```powershell
$sanAddress = "[DNS address]"

$thumbprint = Get-ChildItem Cert:\LocalMachine\My -Recurse | Where-Object {
    $_.NotAfter -gt (get-date) -and $_.NotBefore -le (get-date)
} | Select-Object -Property Thumbprint,@{
    Name="San";
    Expression={$_.Extensions | Where-Object {$_.Oid.FriendlyName -eq "subject alternative name"}}
} | Where-Object {
    $_.San.Count -gt 0 -and 
    $_.San.Format(1).Contains("DNS Name=$sanAddress")
} | Select-Object -ExpandProperty Thumbprint

Set-OctopusVariable -name "Thumbprint" -value "$thumbprint"
```