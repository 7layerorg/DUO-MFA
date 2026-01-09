# DUO-MFA

Duo MFA for pfSense OpenVPN - Quick Setup Steps

Components Required:

    1. pfSense (192.168.0.100) - OpenVPN Server with RADIUS Authentication 
    2. Active Directory (192.168.0.145) - User authentication and VPN group 
    3. Duo Authentication Proxy (192.168.0.145) - MFA gateway 

1. Active Directory Configuration
    Create service account: pfsense-service-account with password ******
    Create security group: VPN (verify DN: CN=VPN,CN=Users,DC=testdc,DC=local) 
    Add authorized users to VPN group 

2. Duo Admin Panel (admin-ced****.duosecurity.com)
    Applications → Protect an Application → LDAP Proxy → Protect 
    Note credentials: ikey, skey, api_host 
    Configure: User access = Enable for all users, Username normalization = Simple 
    Enroll users with username test and add aliases: testdc\test, testdc.local\test 

3. Install Duo Authentication Proxy (192.168.0.145)
    Download: https://dl.duosecurity.com/duoauthproxy-latest.exe 
    Install as Administrator to default location 

4. Configure Duo Auth Proxy
      Edit: C:\Program Files\Duo Security Authentication Proxy\conf\authproxy.cfg 
```bash
    [ad_client]
    host=192.168.0.145
    service_account_username=pfsense-service-account
    service_account_password=********
    search_dn=DC=testdc,DC=local
    security_group_dn=CN=VPN,CN=Users,DC=testdc,DC=local
    
    ### SLDAP SETUP STARTTLS###
    transport=starttls    						    ###### This needs to work via secure ldap                  ##  
    ssl_ca_certs_file=conf/ad_ca.pem 				###### If you remove these three lines it will still work  ##
    ssl_verify_hostname=false					    ###### but passwords will travel via cleartext!!!          ##

    [radius_server_auto]
    ikey=DIQM9X75Z00************
    skey=QuVjZhVk3cmUTpHf9ML**************
    api_host=api-ced*****.duosecurity.com
    radius_ip_1=192.168.0.100   ### This is the pfSense’s IP ###
    radius_secret_1=*******  ### This password should match in pfSense to this auth proxy radius access ###
    failmode=safe
    client=ad_client
    port=1812
    client_ip_attr=NAS-IP-Address
```

Run this in Powershell to extract the certificate:
```powershell
$dcCert = Get-ChildItem -Path Cert:\LocalMachine\My | Where-Object { $_.EnhancedKeyUsageList.ObjectId -contains "1.3.6.1.5.5.7.3.1" } | Sort-Object NotAfter -Descending | Select-Object -First 1 if ($dcCert) { Write-Host "Found certificate: $($dcCert.Thumbprint)" Write-Host "Subject: $($dcCert.Subject)" $certBytes = $dcCert.Export([System.Security.Cryptography.X509Certificates.X509ContentType]::Cert) $base64 = [System.Convert]::ToBase64String($certBytes) $pem = "-----BEGIN CERTIFICATE-----`n" for ($i = 0; $i -lt $base64.Length; $i += 64) { $length = [Math]::Min(64, $base64.Length - $i) $pem += $base64.Substring($i, $length) + "`n" } $pem += "-----END CERTIFICATE-----" $pem | Out-File "C:\ad_ca.pem" -Encoding ASCII -Force Copy-Item "C:\ad_ca.pem" -Destination "C:\Program Files\Duo Security Authentication Proxy\conf\ad_ca.pem" -Force Write-Host "Certificate exported and copied to Duo Proxy!" } else { Write-Host "ERROR: No LDAP certificate found!" }
```

5. Start Duo Auth Proxy Service
    Command: net start duoauthproxy or use Proxy Manager GUI 
    Verify logs: C:\Program Files\Duo Security Authentication Proxy\log\authproxy.log 

6. pfSense RADIUS Configuration
    System → User Manager → Authentication Servers → Add 
    Type: RADIUS, IP: 192.168.0.145, Port: 1812, Shared Secret: ******* 
    Protocol: PAP (critical - not MS-CHAPv2), Authentication Timeout: 60 seconds 
    Save and Test authentication 

7. pfSense OpenVPN Configuration
    VPN → OpenVPN → Servers → Edit your server 
    Backend for authentication: Select Duo RADIUS from dropdown 
    Save and restart OpenVPN service 

8. Test Authentication
    Connect with OpenVPN client using AD credentials 
    Approve Duo push notification on mobile device 
    Verify VPN connection established 

Key Notes:
    Only users in the VPN AD group can authenticate (enforced by security_group_dn) 
    PAP protocol required (MS-CHAPv2 not supported with Duo) 
    Username normalization handles test, testdc\test, testdc.local\test formats 
    Windows admins manage access via VPN AD group - no pfSense/Duo changes needed 


<img width="866" height="577" alt="image" src="https://github.com/user-attachments/assets/053da858-b7a4-4b94-801d-1ffc631084bb" />


