# Logitech-Sync-API-and-PowerShell
This article goes through how to use Logitech Sync API with PowerShell  
The Logitech Sync API is pretty easy to use with some basic understadning in PowerShell.  
Before you start you need to get access to the APIs.  While these are free as a trial, there is a cost to use the APIs after the triel period.  You can find more about syncc here: https://sync.logitech.com/hub/syncguides.  
The API falls under the support options, which can be found here: <A HREF="https://www.logitech.com/en-us/products/video-conferencing/room-solutions/select-comprehensive-service-plan.html?srsltid=AfmBOopaiYJMAUykdzW95g3wzgAjCtFVTvs6rwQ6xaZDtJbRr4S9i3sT#compare-plans">Logitech Services"</A>  
## Gathering the information needed  
You can find out how to get your certificates and OrgID from this article under the section <A HREF="https://sync.logitech.com/hub/syncguides/post/1-8-getting-started---api-quick-start-guide-QOU16rRoLmzBP9F">Generate your connection details in Sync</A>  
1. The two certificates must have the same names, but different extensions.  As an example, rename the PrivateKey.pem to certifcate.key.  You will now have a certificate.pem and certificate.key both in the same folder
2. Open a Terminal as admin and CD to the folder containing the two files.
3. Merge the two files into a pfx format that can be imported into a Windows' computer certificate store. This will ask you for a password so the cert can be viewed later, don't forget this password.
```PowerShell
 certutil -mergepfx .\certificate.pem .\certificate.pfx
``` 
5. You can then import the cert.  It is important to figure out if you store it in the computer store (Enterprise) or user Store.  If you plan on doing any kind of automation, I.E. through a scheduled task when nobody is logged on, then it should go under Enterprise. A user will not be able te export the certifacte's private key. After the import the certificate name will be shown.
```PowerShell
certutil -p <password> -importPFX Enterprise .\certificate.pfx
```
If you use Enterprise, the key will be located at Cert:\LocalMachine\Enterprise (this is important when creating the cert).  In some cases it may end up in the personal enterprise.  You can also import the PFX using the Certifcate MMC snapin.  
6. If you want to view the certifcate information, for example to get the thumbprint (also known as the Cert Hash(Sha1)) you can run the following command:  
```PowerShell
certutil -p <Password> -dumppfx .\certificate.pfx`'
```
  You will need the Hash later when making the connection to retrieve the key.
## To Connect to Sync Cloud
1. Get the certficate as a variable. The thumbprint.
   ```PowerShell
         $thumbprint = "0c65881beecbe5fcf0459f177436394694fxxxx"
         $cert = Get-Item Cert:\LocalMachine\Enterprise\$thumbprint
2. Create the connection URI
   ```PowerShell
      $OrgID = "oELr1HAMScRVcBluGumiVIsfFfLXXXX"
      $apiuri = "https://api.sync.logitech.com/v1/org/$OrgID/place"
3. Invoke the request
   ```PowerShell
      $return = Invoke-RestMethod -URI $apiUri -Method Get -Certificate $cert -Headers @{ "Accept" = "Application/json" }
   ```
## Making it easier to connect
That's not an easy way to remember connections, you have to know the OrgID and thumbprint and store it in a table.  Here is something to help make it easier so you only need to remember your Org Name or any other simple name. When you look at the certificate store, you get this, not very helpful.  
<img width="468" height="145" alt="image" src="https://github.com/user-attachments/assets/45514125-bbd5-43fb-985a-c0018ad5ce02" />  
Let's give it a friendly name
1.  You need to get the certificate thumbprint: 
   ```PowerShell
   certutil -p <Password> -dumppfx .\certificate.pfx`'
   ```
2. That will return something like this:   
   <img width="611" height="172" alt="image" src="https://github.com/user-attachments/assets/f1f99a85-c3cc-40db-b4d8-ce6e76596fef" />  
1.  What is needed from here is the Cert Hash, or Thumbprint (which I mentioned above).  
1.  You can now give the cert a friendly name, first you have to get the cert as an object then assign it a friendly name.  
   ```PowerShell
   $cert = Get-ChildItem Cert:\LocalMachine\Enterprise\ | Where-Object {$_.Subject -like "*0c65881beecbe5fcf0459f17743639xxxxxxxx"}
   $cert.FriendlyName = "Logitech Demo"
   ```
5. This will now give you a friendly name to work with  
   <img width="462" height="144" alt="image" src="https://github.com/user-attachments/assets/30924ef4-ac98-4b41-b205-f02088287a6d" />  
7.  You can now get the cert details by just knowing the Friendly Name
   ```PowerShell
   $cert = Get-ChildItem Cert:\LocalMachine\Enterprise\ | Where-Object {$_.FriendlyName -like "Logitech Demo"}
   ```
#### Identifying the OrgID from the cert
If you have multiple org ids in Sync, you will end up with a few certficates, keeping track of that can be tricky, but there is a method to figuring it out.  
The Subject of the certifcate has what you need.  The CN is the certifcate being used and the O is your Org ID.  This follows what you would expect.  
<img width="804" height="537" alt="image" src="https://github.com/user-attachments/assets/90505bc4-c02b-4f58-b84f-15f4872bc748" />  
To extract the OrgID from the cert you just need to parse it from the certifcate.  
1. We already have `$cert` from above from the friendly name.  
2. Parse out the string to get the orgID.  
```PowerShell
$orgID = ((($cert | select Subject).Subject) -split (", O="))[1]
```
### Putting this all together
```PowerShell
$cert = Get-ChildItem Cert:\LocalMachine\Enterprise\ | Where-Object {$_.FriendlyName -like "Logitech Demo"}
$orgID = ((($cert | select Subject).Subject) -split (", O="))[1]
$apiuri = "https://api.sync.logitech.com/v1/org/$OrgID/place"
$return = Invoke-RestMethod -URI $apiUri -Method Get -Certificate $cert -Headers @{ "Accept" = "Application/json" }
```
## Doing something worhtwhile with the data
It's great that you can get the information, but let's say you want to know any room that is not healthy.  
1. To start I need to give a brief explination of what is returned.  
2. When you return just `$return` you will get this:  
   <img width="324" height="103" alt="image" src="https://github.com/user-attachments/assets/5f9807d2-fcfd-4b37-96da-f7be626c62d6" />  
4. The parent of this is places, that is key (no pun intended).  
5. This means, you can get actual data by running the following: `$return.places, which return each place, but with each device in that place under devices.
   <img width="2246" height="641" alt="image" src="https://github.com/user-attachments/assets/5b728f37-180a-4b34-a6db-a517a8eabb4d" />  
6. So to get devices, you can run the following `$return.places.devices`  This will return all the devices, however, you will not know which room the device is in.
   <img width="1484" height="945" alt="image" src="https://github.com/user-attachments/assets/861e16ba-94c9-49a2-9ec2-f3e2b087d054" />  
8. So for this example, we will pretend that our rooms healthStatus HAS issues.  We can return that information with the following script. For purposes of example, I am choising to healtStatus of no issues.  In the real world, you would pick -neq "NoIssues"  
```PowerShell
   $return.places | ForEach-Object {
     $place = $_
     $place.devices | Where-Object { $_.healthStatus -eq "NoIssues" } | ForEach-Object {
         [PSCustomObject]@{
             PlaceName  = $place.name
             DeviceName = $_.name
             Health     = $_.healthStatus
         }
     }
 }
 ```
7. This would return the following:
   <img width="632" height="168" alt="image" src="https://github.com/user-attachments/assets/5b4566d1-ea92-488a-bf7c-8cc9b164e139" />  
## So now what
Once you can get information from the API, the world is your oyster.  You can send the results off to EventViewer, where you can then use Performance MOntiroing to watch for an event ID you created to send an email. Or use an existing device monitor to do whatever when it sees that event ID.

## More Details
So this is all great, but let's get some more details  
To get this information, we need to change the string query a little bit more  
We will start with the query string.  According to the spec file, we can find information on Spot at the following: place.device.sensors  
However, we also nee the device as well which is held at: place.device  
So now we need to build our new API query:  
```PowerShell
$apiuri = "https://api.sync.logitech.com/v1/org/$OrgID/place?limit=100&projection=place.info,place.occupancy,place.device,place.device.info,place.device.status,place.device.sensors"
$return = Invoke-RestMethod -URI $apiUri -Method Get -Certificate $cert -Headers @{ "Accept" = "Application/json" }
```
This now stors all the sensor data in $return as an object, 
<img width="1701" height="108" alt="image" src="https://github.com/user-attachments/assets/de98bbc2-08ec-4d49-9c1d-05c704537974" />
do get this we need to cycle through the object
```PowerShell
foreach ($place in $return.places) {
        # Print room/desk info
        if ($place.PSObject.Properties.Name -contains 'name') {
            Write-Host "Place: $($place.name)"
        }
        if ($place.PSObject.Properties.Name -contains 'location') {
            Write-Host "Location: $($place.location)"
        }
        if ($place.PSObject.Properties.Name -contains 'occupancy') {
            Write-Host "Occupancy: $($place.occupancy)"
        }
        Write-Host ""

        # Loop devices
        foreach ($device in $place.devices) {
            Write-Host "  Device: $($device.name)"

            # Offline check
            if ($device.PSObject.Properties.Name -contains 'status' -and $device.status -eq 'Offline') {
                Write-Host "    offline"
                Write-Host ""
                continue
            }
            # Print IP address if available
            if ($device.PSObject.Properties.Name -contains 'network' -and $device.network) {
                if ($device.network.PSObject.Properties.Name -contains 'ip') {
                    Write-Host "    IP Address: $($device.network.ip)"
                }
            }

            # Sensor readings
            if ($device.PSObject.Properties.Name -contains 'sensors' -and $device.sensors) {
                Write-Host "    CO2: $($device.sensors.co2)"
                Write-Host "    Temp: $($device.sensors.temperature)"
                Write-Host "    Humidity: $($device.sensors.humidity)"
                Write-Host "    Pressure: $($device.sensors.pressure)"
                Write-Host "    TVOC: $($device.sensors.tvoc)"
                Write-Host "    VOC Index: $($device.sensors.vocIndex)"
                Write-Host "    PM2.5: $($device.sensors.pm25)"
                Write-Host "    PM10: $($device.sensors.pm10)"
                Write-Host "    Presence: $($device.sensors.presence)"
                Write-Host "    Timestamp: $($device.sensors.latestTs)"
                Write-Host ""
            }
        }
        Write-Host "---------------------------------------------"
    }
```
and what you will end up with is something like this.  You can extend this information into writing into a an Event View Log that you could then subscribe to and then take action on.  
<img width="500" height="340" alt="image" src="https://github.com/user-attachments/assets/183515f2-86b4-402a-b79f-1047e2b4fe12" />
## Understanding the Projections
The API doesn’t automatically give you all the details about rooms and devices. Instead, it uses a projection parameter, which is like a shopping list of the fields you want back. If you don’t ask for them, you’ll only get the bare minimum (like IDs).  
If you look at the openapi spec, and search for projections, they are listed there  
* place.info → basic details about the room or desk (name, location, group)  
* place.occupancy → how many people are in the room right now  
* place.device → makes sure devices are included at all    
* place.device.info → device details (name, version, network info like IP address)  
* place.device.status → whether the device is Online, Offline, or InUse  
* place.device.sensors → the actual environmental readings (CO₂, temperature, humidity, etc.)  
From that list, I picked the ones that matched what you wanted:  
* Sensors → place.device.sensors  
* Status → place.device.status (so we can print “offline”)  
* IP address → place.device.info (because network info lives there)  
* Context → place.info and place.occupancy (so you know where the readings came from and how many people were in the room)  
## working with the other schema options

