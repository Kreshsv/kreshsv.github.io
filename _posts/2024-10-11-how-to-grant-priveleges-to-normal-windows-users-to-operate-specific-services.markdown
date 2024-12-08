---
layout: post
title: "How to grant priveleges to normal windows users to operate specific services"
date: 2024-10-11 20:44:24 +0530
categories: Windows
---
# How to grant priveleges to normal windows users to operate specific services

As a Windows sys admin, one day I was stuck in a situation where one of the normal users of my company required privileges to start/stop the MSSQL service.  However, I cannot make the user an admin which would violate the principle of least privilege.  

Hence I came up with the solution provided by Windows - SDDL (Security Descriptor Definition Language). The following runbook provides steps to grant privileges to start/stop the ‘SysMain’ service. You can follow the same steps for any other service. 

**Note: Fire all the provided commands in the windows command prompt.**

## Get the service name of the service

- Open Windows Run using Win + R
- Type ‘services.msc’ and hit enter
- Search for the service and Right click on properties
- Copy the value of ‘Service name’ from the properties. 

*Note: The service name is different from the display name. The display name is the one which is displayed in the services list for human readability while the service name is the actual string that can be interpreted by SDDL.*

Alternatively, if you know the display name of the service:-

Command to get the service name using the service controller

```sc getkeyname <service_display_name>```

Example

```sc getkeyname sysmain```

The above command will return the service name that can be used to update the security descriptor. 

## Get SID of the user

In windows, each user/group has a unique SID (Security Identifier) required by the service controller to grant privileges. 

Command to get the SID of all users in your host.

```wmic useraccount get name,sid```

Copy and save the SID of the user to whom you want to grant additional privileges to be used later.

## Get SDDL of the service

Every service has its default security descriptor. Before modifying it we need a copy of the default security descriptor for backup and ease of modification. 

Command to show the current security descriptor of a service defined by SDDL

```sc sdshow <service_name>```

Example 

```sc sdshow SysMain ```

Sample output

```D:(A;;CCLCSWRPWPDTLOCRRC;;;SY)(A;;CCDCLCSWRPWPDTLOCRSDRCWDWO;;;BA)(A;;CCLCSWLOCRRC;;;IU)(A;;CCLCSWLOCRRC;;;SU)S:(AU;FA;CCDCLCSWRPWPDTLOCRSDRCWDWO;;;WD)```
Copy and keep this output for later use. 

**To better understand the returned SDDL: https://learn.microsoft.com/en-us/windows/win32/secauthz/ace-strings**

## Update the SDDL of the service 

*Note: You need to open command prompt as adiministrator to update the security descriptor.*

Now let’s add the below string to the existing security descriptor of the service. 

```(A;;<Added permissions>;;;<SID_of_target_user>)```

The above string will grant the target user the privilege to start, stop and restart the target service.

Example

```(A;;RPWP;;;S-1-5-21-3117389765-1167674771-1111219307-1018)```

Explanation:-

| Item  | Meaning |
| ---- | ------- |
| A  | To add permission |
| RP | Read permission  |
| WP | Write permission |
| S-1-5-21-3117389765-1167674771-1111219307-1018 | SID of the target user

After adding our new string to the old security descriptor, the new security descriptor would look something like this:-

```D:(A;;CCLCSWRPWPDTLOCRRC;;;SY)(A;;CCDCLCSWRPWPDTLOCRSDRCWDWO;;;BA)(A;;CCLCSWLOCRRC;;;IU)(A;;CCLCSWLOCRRC;;;SU)(A;;RPWP;;;S-1-5-21-3117389765-1167674771-1111219307-1018)S:(AU;FA;CCDCLCSWRPWPDTLOCRSDRCWDWO;;;WD)```

Note that I have added the new string at the end of D: section and before the beginning of the next section, S:.

Command to update the security descriptor with a new one

`sc sdset <service_name> "D:(A;;CCLCSWRPWPDTLOCRRC;;;SY)(A;;CCDCLCSWRPWPDTLOCRSDRCWDWO;;;BA)(A;;CCLCSWLOCRRC;;;IU)(A;;CCLCSWLOCRRC;;;SU)(A;;RPWP;;;S-1-5-21-3117389765-1167674771-1111219307-1018)S:(AU;FA;CCDCLCSWRPWPDTLOCRSDRCWDWO;;;WD)"
`
If you get the below output after executing the command, the SDDL has been updated successfully. 

`[SC] SetServiceObjectSecurity SUCCESS`

The SDDL usually takes effect on the immediate login of the user. If not try restarting windows. Now the normal user can start, stop and restart the target service. 

***

### Cheers to system administrators who uphold the principle of least privilege.:wink:












