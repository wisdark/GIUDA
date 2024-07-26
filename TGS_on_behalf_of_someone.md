Content:
1. What is TGT
2. Logon session
3. How the LSA requests Kerberos tickets
4. TGT Ticket Theft
5. TGT is TGS
6. Conclusion

There are various ways to abuse a user's session on a device. But did you know that it is possible to spoof a TGT ticket receipt by a user using legitimate Windows functions?

Giuda allows you to get the tickets of the logged-in user even without having his password! Today we will understand how GIUDA works

What is TGT
A TGT is a special type of ticket that is issued to a user upon successful authentication in the Kerberos system. This process is carried out using the user's password and prevents the password from being transmitted over the network.

Here's how TGT works.

1. User authentication:
The user enters his login and password.
The Kerberos authentication server verifies that the password is correct.
If authentication is successful, the server generates a TGT.
2. TGT Issuance:
The TGT contains information about the user, the time of their authentication, and other metadata.
The TGT also contains a session key that will be used for further communication.
3. Using TGT to get other tickets:
A user can use TGT to request tickets to access various resources on the network.
Each ticket contains information about a specific resource and is protected by a session key, which ensures the security of the transfer.


Logon session
When a user is authorized in Windows, a user session is created, in which all user data is stored. A new session is created for all new users. For example, if there are two users on the machine at the same time, there will be two sessions.

![immagine](https://github.com/user-attachments/assets/b47233c7-510a-401a-9a10-207c7c8e722b)

Each session has a LUID (Locally Unique Identifier) name. As the name suggests, LUID is unique for each session. The information is stored in a structure format.

Pascal
```
type
    _LUID = record
    LowPart: DWORD;
    HighPart: LongInt;
  end;
```

C
```
typedef struct _LUID {
ULONG LowPart;
LONG HighPart;
} LUID, *PLUID;
```

The LUID itself is represented by two values: ULONG and LONG. In this case, usually only the field is filled in LowPartand HighPart matters.

This structure is used by all WinAPI functions that are somehow related to user sessions.

Using the GetTokenInformation() you can find out the custom LUID. To do this, the function must be passed a token of the process running on behalf of the current user.

Now it's time to show how Kerberos tickets are requested by the LSA itself. This will help us spoof the LUID and get someone else's ticket.

How the LSA requests Kerberos tickets
To request a TGS ticket, the LSA receives an SPN (service principal name) and passes it to the KDC. We can request TGS tickets ourselves. For this, there is a function LsaCallAuthenticationPackage().

Pascal
```
function LsaCallAuthenticationPackage (
    LsaHandle : THandle;
    AuthenticationPackage : ULONG;
    ProtocolSubmitBuffer : pointer;
    SubmitBufferLength : ULONG;
    var ProtocolReturnBuffer : pointer;
    var ReturnBufferLength : ULONG;
    var ProtocolStatus : NTStatus) : NTSTATUS; stdcall;
```

C
```
NTSTATUS LsaCallAuthenticationPackage(
  [in]  HANDLE  LsaHandle,
  [in]  ULONG   AuthenticationPackage,
  [in]  PVOID   ProtocolSubmitBuffer,
  [in]  ULONG   SubmitBufferLength,
  [out] PVOID   *ProtocolReturnBuffer,
  [out] PULONG  ReturnBufferLength,
  [out] PNTSTATUS ProtocolStatus
);
```

where:

LsaHandle - A handle pointing to the LSA service, which can be retrieved using LsaRegisterLogonProcess() or LsaConnectUntrusted();

AuthenticationPackage - The AP number that you want to interact with.

ProtocolSubmitBuffer — the buffer being passed, we will give KERB_RETRIEVE_TKT_REQUEST;

SubmitBufferLength is the size of the buffer to be transferred;

ProtocolReturnBuffer is the response from AuthenticationPackage. The structure of the KERB_RETRIEVE_TKT_RESPONSE will fly to us;

ReturnBufferLength — the size of the buffer with the response;

ProtocolStatus is a value that will contain the error code from the AP.


So, how to fill it out KERB_RETRIEVE_TKT_REQUESTto get a TGS ticket? The structure looks like this:

```
KERB_RETRIEVE_TKT_REQUEST =record
    MessageType:KERB_PROTOCOL_MESSAGE_TYPE;
    LogonId:_LUID;
    TargetName:UNICODE_STRING;
    TicketFlags:ULONG;
    CacheOptions:ULONG;
    EncryptionType:LONG;
    CredentialsHandle:SecHandle;
    end;
    PKERB_RETRIEVE_TKT_REQUEST=^KERB_RETRIEVE_TKT_REQUEST;
```

```
typedef struct _KERB_RETRIEVE_TKT_REQUEST {
  KERB_PROTOCOL_MESSAGE_TYPE MessageType;
  LUID             LogonId;
  UNICODE_STRING       TargetName;
  ULONG            TicketFlags;
  ULONG            CacheOptions;
  LONG             EncryptionType;
  SecHandle          CredentialsHandle;
} KERB_RETRIEVE_TKT_REQUEST, *PKERB_RETRIEVE_TKT_REQUEST;
```

where:
MessageType is what we need to get from the AP. Specify KerbRetrieveEncodedTicketMessage;
LogonID is the LUID of the session on behalf of which the AP is accessed. This is when the LUID will be substituted. The problem is that if we connect to the LSA via LsaConnectUntrusted(), we won't be able to specify the LUID of someone else's session here — the LSA will throw a 0x5 ERROR_ACCESS_DENIED error, but if we connect via LsaRegisterLogonProcess(), we can pass any desired LUID here. And in this way we will be able to request tickets from someone else's session;
TargetName — here specify the SPN of the service to which you want to get a ticket;
CacheOptions - Options related to the LSA cache. The LSA cache is a kind of storage in which tickets are stored. There are some peculiarities here too. If we immediately specify the KERB_RETRIEVE_TICKET_AS_KERB_CRED (the value for obtaining a ticket in the form KRB_CRED, immediately with the session key; details are in my other article), then there is a chance that you will not get a ticket. The problem is that the LSA cache may not have a ticket for the service we want to go to. And if we immediately indicate KERB_RETRIEVE_TICKET_AS_KERB_CRED, then the LSA may simply not return any ticket, since there is nothing to return. Therefore, you will have to call the LsaCallAuthenticationPackage() function twice. The first time is with the meaning of KERB_RETRIEVE_TICKET_DEFAULT, the second time is with KERB_RETRIEVE_TICKET_AS_KERB_CRED. … DEFAULT is responsible for requesting a ticket. That is, we ask the LSA to contact the KDC and get a ticket;
EncryptionType - The desired type of encryption for the requested ticket. Specify KERB_ETYPE_DEFAULT — the type of encryption is not important to us;
CredentialsHandle - Used for SSPI, it doesn't matter in this case.


TGT Ticket Theft
We've figured out how Kerberos ticket request works on a local system. It's time to move on to operation!
First we list all the available sessions with undetected method "KLIST SESSIONS" --- ;-)

![immagine](https://github.com/user-attachments/assets/1668edd0-ad09-4ef5-aa85-9ba261847f30)

The next step is to connect to the LSA using LsaRegisterLogonProcess()to pass the LUID to someone else's session. To call this function, you need the SeTcbPrivilege.

```
LsaRegisterLogonProcess:function(
     LogonProcessName:PLSA_STRING;
     LsaHandle:PHANDLE;
     SecurityMode:PLSA_OPERATIONAL_MODE
  ):NTSTATUS;stdcall;
```


```
NTSTATUS LsaRegisterLogonProcess(
  [in]  PLSA_STRING           LogonProcessName,
  [out] PHANDLE               LsaHandle,
  [out] PLSA_OPERATIONAL_MODE SecurityMode
);
```

The next step is to use LsaLookupAuthenticationPackage() get the Kerberos AP number.

```
  LsaLookupAuthenticationPackage:function(
       LsaHandle:HANDLE;
       PackageName:PLSA_STRING;
       AuthenticationPackage:PULONG
 ):NTSTATUS ;stdcall;
```

```
NTSTATUS LsaLookupAuthenticationPackage(
  [in]  HANDLE      LsaHandle,
  [in]  PLSA_STRING PackageName,
  [out] PULONG      AuthenticationPackage
);
```

Finally, we have the handle, the LUID, and the Kerberos AP number. It's time for GIUDA to betray

To do this I used the function kuhl_m_kerberos_ask(target:string;export_:bool=false;logonid:int64=0):NTSTATUS;

This function uses LsaCallAuthenticationPackage to call the LSA, and LSA will now contact the KDC and receive a new ticket. If we try to extract it right away, it will not be valid. More precisely, it will not have a session key and you will not be able to use it.

Therefore, after making sure that the call was successful, change CacheOptions to KERB_RETRIEVE_TICKET_AS_KERB_CRED and refer to the LSA.
```
(first call to LsaCallAuthenticationPackage)
pKerbRetrieveRequest^.CacheOptions :=  KERB_RETRIEVE_TICKET_DEFAULT;
...
(if success change the CacheOptions)
pKerbRetrieveRequest^.CacheOptions:= pKerbRetrieveRequest^.CacheOptions or  KERB_RETRIEVE_TICKET_AS_KERB_CRED;
```

```
(first call to LsaCallAuthenticationPackage)
pKerbRetrieveRequest->CacheOptions = KERB_RETRIEVE_TICKET_DEFAULT;
...
(if success change the CacheOptions)
pKerbRetrieveRequest->CacheOptions = KERB_RETRIEVE_TICKET_AS_KERB_CRED;
```

Ticket got on behalf of other user without password, GIUDA betrayed!

Then?

TGT is TGS
It would seem that getting a TGS ticket is a great result! But you always want more, right? Did you know that a TGT ticket is actually a TGS ticket, but for the krbtgt service? It turns out that we have a TGS ticket for krbtgt, and the krbtgt service allows us to issue other TGS tickets. That's all.

Hei Gringo your car is vavavumaaaaaaa
We can request other people's TGT tickets! Thus, if during a pentest it was possible to capture a host where users go, then using TGSThief it will be possible to get TGT of these users. Moreover, TGT tickets will be absolutely fresh, new, just requested.