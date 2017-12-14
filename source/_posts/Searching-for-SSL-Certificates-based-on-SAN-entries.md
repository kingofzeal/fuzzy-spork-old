---
title: Searching for SSL Certificates based on SAN entries
tags:
  - LetsEncrypt
  - SSL Certificates
  - Octopus Deploy
  - Devops
date: 2017-12-21 08:00:00
---


Lately, I've been playing with [Lets Encrypt](https://letsencrypt.org/) as a means to encrypt public(ish) web-based services I run on my local server. It's actually remarkable what this non-profit group has accomplished when it comes to SSL - they offer free, automated SSL certificates to anybody who wants them. They even support SAN certificates (those that cover multiple, explicitly-named DNS entries), and starting in January they will offer wildcard certificates. I will tell anybody who is willing to listen, and who is looking to avoid paying paying $50+ for an SSL certificate, about them and encourage everyone to look into their service. <!-- more -->

The only real downside (which some argue isn't really a downside) is that these certificates generally have a pretty short lifetime - they last 90 days, with no options for longer lengths (unlike other CAs). There are reasons for this, but they make up for this by being completely automated - it literally takes seconds to get a new certificate. If you are on a Windows machine, you can snag a tool from Github which not only handles the request for the certificate, but also installs it, updates your IIS bindings, and creates a scheduled task that will look at the state of your certificates and, if needed (generally after about 60 days - 30 days before they expire), automatically renew them. It is as close to a set it and forget it system, and you don't have to run the risk of letting your certificates expire while you wait for a new one.

However, having constantly (and automatically) changing SSL certificates can preset an interesting challenge if you use Octopus Deploy to deploy your websites. If you are only deploying a single website, then the answer is trivial - there is a [community step template](http://library.octopusdeploy.com/step-templates/bc81b8a6-dc56-4769-87b5-650af7a38162/actiontemplate-lets-encrypt-create-ssl-certificate) that takes care of most of the hard work for you. 

But this only renews the certificate when you do to deploy a project - if you could potentially go more than 90 days between deployments, you run the risk of the certificate expiring. There are features that help mitigate this, such as setting up Octopus notifications when a certificate is about to expire, but ultimately you still need to perform a deployment to get the new certificate.
Additionally, this really only works if you have a single website. If you are deploying multiple websites to the same IIS service, and you want to bind all of them to port 443 (the default SSL port), you need a SAN certificate, which the current template doesn't really addresses. 

There is a currently outstanding [UserVoice suggestion](https://octopusdeploy.uservoice.com/forums/170787-general/suggestions/15045072-support-letsncrypt-for-octopus-certificates) to address this by adding Lets Encrypt a first-class citizen, but it looks like that's part of ['Phase 3'](https://github.com/OctopusDeploy/Issues/issues/2701) of managing certificates. In the meantime, there is a post on the UserVoice suggestion that offers a workaround: run a script that finds a given certificate by Subject name (often the primary DNS Name the certificate was issued for), and update the thumbprint in the deployment process with whatever certificate is currently there.

For now, this seems like a better option. It doesn't require any unnecessary deployments - the Lets Encrypt certificate is presumably scheduled to be updated on the server periodically, and the current thumbprint is simply obtained and used at deployment time. This allows for greater flexibility in how certificates are managed, by being able to use SAN or (eventually) wildcard certificates. While they aren't being managed by Octopus, which can be argued is the best place to have them, at the very least having a new certificate for the site won't interfere with the deployment process. 

However, there is one last hurdle - the certificate is identified by Subject, which may not necessarily represent the DNS entry for the website we're deploying to. In the case of a SAN certificate, if it covers 4 websites, you will need to provide this detail (and hope it doesn't change) for 3 out of the 4 websites. This is less than ideal, as a change in one project may mean a change in others.

After much trial and error, I found that altering the script a little bit allows us to search for a certificate that covers the website we're deploying. I've turned this into a Step Template so we can re-use the logic in multiple projects, but this is the core of the script:

```powershell
Write-Host "Search Enabled: $SearchDynamicCertificate"
Write-Host "Default Thumbprint: $DefaultThumbprint"
Write-Host "DNS Subject: $DnsSubject"

$disoveredThumbprint = $DefaultThumbprint

if ($SearchDynamicCertificate){
    $thumbprint = Get-ChildItem Cert:\LocalMachine\My -Recurse | Where-Object {
        $_.NotAfter -gt (get-date) -and $_.NotBefore -le (get-date)
    } | Select-Object -Property Thumbprint,@{
        Name="San";
        Expression={$_.Extensions | Where-Object {$_.Oid.FriendlyName -eq "subject alternative name"}}
    } | Where-Object {
        $_.San.Count -gt 0 -and 
        $_.San.Format(1).Contains("DNS Name=$DnsSubject")
    } | Select-Object -ExpandProperty Thumbprint
    
    if (-not [string]::IsNullOrWhiteSpace($thumbprint)){
        Write-Host "Found certificate $thumbprint that covers $DnsSubject"
        $disoveredThumbprint = $thumbprint
    }
}

Write-Host "Using $disoveredThumbprint for DiscoveredThumbprint"
Set-OctopusVariable -name "DiscoveredThumbprint" -value "$disoveredThumbprint"
```

`DefaultThumbprint`, `SearchDynamicCertificate` and `DnsSubject` are all parameters of the template, and in our case they map directly to the original `thumbprint` project variable, a new checkbox project variable, and the binding DNS project variable, respectively. 

In a nutshell, it goes through all certificates in the certificate store, filters out any that are expired or predated, extracts the SAN entries, and returns the thumbprint of any certificates that have a SAN entry for the given address, which is then set as an output variable (`DiscoveredThumbprint`). This _could_ return multiple certificates, if they cover the same DNS entries, but generally speaking you shouldn't see that very much, if at all.

If no certificate is found, or if `SearchDynamicCertificate` is false, then it just uses the default thumbprint provided. This process does assume that IIS is setup correctly (only one certificate per port, regardless of how many websites bind to that port), and does not do any validation that the certificate found is the correct one to be used. There are probably additional checks that could be performed around that area, but for now this works for "happy-path" deployments.