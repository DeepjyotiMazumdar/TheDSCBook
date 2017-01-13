# LCM and Pull Server Communications
I find that a lot of troubleshooting is easier once you know what's _supposed_ to be happening under the hood. So, I set up an unencrypted pull server and a node, and packet-captured their communications. You can do the same in a troubleshooting scenario. Note that I focused on Pull Server Model v2 communications, meaning this isn't applicable to a WMF4 client. Incidentally, you can check out https://github.com/powershellorg/tug, in the References folder, for copies of the WireShark traces I used to figure all this out.

By the way - doing this kind of sniffing is the only reason I will switch a Pull server to HTTP, and I do so only on a closed-off virtual network.

### Node Registration
Whenever a node first contacts a pull server, it needs to _register_, using the RegistrationKey data you provide in its LCM configuration. This happens per pull server and per reporting server, even if you have them all running on the same server machine. Like most of these communications, the node will send an HTTP PUT request to the pull server. The URL will be in the format:

```
/PSDSCPullServer.svc/Nodes(AgentId='91E51A37-B59F-11E5-9C04-14109FD663AE')
```
Note that the AgentId is the one configured in the LCM; a brand-new LCM might not show one when you run Get-DSCLCMConfiguration, but after applying a meta-configuration to the LCM, an AgentId will always exist and be visible. The body will contain a JSON (JavaScript Object Notation) object:

```
{ "AgentInformation":
	{ 
	  "LCMVersion": "2.0",
	  "NodeName": "hostname",
	  "IPAddress": "ip_address(1)"
	},
	
  "ConfigurationNames":
  	[ (array of strings) ],
  	
  "RegistrationInformation":
  	{
  	  "CertificateInformation":
  	  	{
  	  	  "FriendlyName": "name",
  	  	  "Issuer": "issuer",
  	  	  "NotAfter": "date",
  	  	  "NotBefore": "date",
  	  	  "Subject": "subject",
  	  	  "PublicKey": "key",
  	  	  "Thumbprint": "thumbprint",
  	  	  "Version": "int"
  	  	},
  	  "RegistrationMessageType": "ConfigurationRepository(2)"
  	}
}
```

(1) IP Address can be a semicolon-delimited list of IP addresses, including both IPv4 and IPv6.

(2) Will be "ReportServer" when registering with a Reporting Server; will not include ConfigurationNames in that case.

The certificate information included in the above is a self-generated certificate created by the LCM. By providing the pull server with a public key, the pull server can now encrypt things that the LCM, and only the LCM, will be able to decrypt. This certificate information isn't used in the native Windows pull server, but the Azure Automation pull server _does_ use it. After initial registration, this certificate is used as a way to authenticate clients. The native pull server stores the information, but doesn't use it.

It's really important to know that this is the one request sent via HTTP PUT, not by POST. The node expects an HTTP 204 reply, with no content in the body of the request. It simply indicates a successful registration; a status of 400 (Bad Request) indicates a malformed request. It's possible for a Web server to block PUT requests, while allowing POST requests. Such a configuration would prevent new registrations.

The LCM actually performs this registration **each and every time** it checks in with the pull server. However, given that the LCM has created an AgentId now stored on the Pull Server (which Get-DSCLocalConfigurationManager will now show). 

The node inserts the RegistrationKey into the HTTP request _headers_:

```
Authorization: Shared xxxxxxxxx
```

The "xxxxxxxx" is where the authorization information goes - but it isn't just the RegistrationKey that you configured for the node. It consists of:

1. The body of the HTTP request, digitally signed using the node's self-generated private key. Bear in mind that the body of the request contains the public key - the pull server can therefore validate the signature. The current date - included in another HTTP header - is added into the signed information.
2. The signature itself is then run through an HMAC hash process, using the RegistrationKey. 

So in order to validate the Authorization: header, the pull server must repeat the signature process, and then create the HMAC using each RegistrationKey that the pull server knows about. If one of the the results matches what the node sent, then it is considered authorized.

This Authorization header is re-sent with _every registration packet_, which happens _every time the node runs a consistency check_. However, remember that the node doesn't actually "remember" that it successfully registered in the past, and the pull server isn't obligated to re-check the Authorization header. My understanding is that, for the native pull server, the Authorization header is ignored if the AgentId is known to the pull server. Given that the AgentId - a GUID - is pretty hard to guess, it seems that on the second-and-subsequent registrations, the AgentId alone is sufficient to re-authorize the node. That isn't the case on Azure Automation's pull server, which as I've mentioned relies on a sort of client certificate authentication, based on the last certificate the node sent.

The node's self-generated certificate is good for one year, is automatically renewed or re-generated, and is sent to the pull server on each registration. _After_ authorizing the node, the pull server should take whatever certificate information is in the registration request as canonical, and store it.

### Node Check-In
This occurs each time the node runs a consistency check.

The protocol specification states that the server can respond with "Ok" (you're fine, don't do anything); "Retry", "GetConfiguration" (your config is out of date, grab it again), or "UpdateMetaConfiguration." However, that last one, in my tests, generates an error on the LCM, stating that it's not a valid response.

The URL looks like this:
```
Nodes(AgentId='91E51A37-B59F-11E5-9C04-14109FD663AE')/GetDscAction
```

The body of the request will contain a JSON object like this, when the server is configured to use a single configuration (e.g., not partials):

```
{
	"ClientStatus": [
    {
    	"Checksum": "checksum",
        "ChecksumAlgorithm": "SHA-256"
    }
    ]
}
```

For nodes configured to use partials, this will look a bit more complex:

```
                {
                    "ClientStatus": [
                        {
                            "Checksum": "checksum",
                            "ConfigurationName": "name",
                            "ChecksumAlgorithm": "SHA-256"
                        },
                        {
                            "Checksum": "checksum",
                            "ConfigurationName": "name",
                            "ChecksumAlgorithm": "SHA-256"
                        }
                    ]
                }
```

The difference is that, with a single configuration, _the server is expected to know what the configuration name is_, because the node supplied it at registration. So the server must store that in its database. For partials, the node must supply the configuration names, since each will have a unique checksum. There's actually intent behind the single-configuration check-in not including the configuration name, and it's so that the server, if it wants to, _can assign a different configuration than the one the client expected_. The native pull server feature in Windows doesn't do this, but Azure Automation's pull server _can_. The idea here is to be able to centrally supply a new configuration without having to tell the node. After all, the node shouldn't care - it should regard any MOF on the pull server as "authoritative" and just do what it says.

The server responds with an HTTP 200, and a JSON response:

```
                    {
                        "odata.metadata": "http://server-address:port/api-endpoint/$metadata#MSFT.DSCNodeStatus",
                        "NodeStatus": "Ok",
                        "Details": {
                            [
                                {
                                    "ConfigurationName": "config-name",
                                    "Status": "Ok"
                                }
                            ]
                        }
                    }
```

As near as I can tell, only the top-level NodeStatus matters. When testing with partial configurations, Details was often blank.

### Requesting a Configuration
Once a node _knows_ that it needs a new MOF, it requests one. For nodes using partial configurations, a separate request is sent for each. URLs take this form:

```
/Nodes(AgentId='agent-id-guid')/Configurations(ConfigurationName='desired-name')/ConfigurationContent
```

There's no special information in the request headers apart from the standard ProtocolVersion (2.0) and Host headers. The body of the request is empty, as all needed information is in the URL. The server responds with 404 for a missing resource, and with a 200 for successful queries. In the event of a successful query, the server response is an application/octet-stream containing the requested MOF. This means the MOF is transmitted as a binary object, rather than as a stream of plain-text.

### Requesting a Module
When a node has been configured to pull modules - which it can do from a "normal" pull server, or from a pull server that's only hosting modules - it submits this URL:

```
/Modules(ModuleName='module-name',ModuleVersion='x.y.z')/ModuleContent
```

You'll notice that none of the above includes the AgentId, which might make you wonder if it's possible to simply grab modules from a pull server by guessing module names. It isn't; the AgentId is actually passed in the HTTP headers for these requests:

```
AgentId: xxxxxxxxx
```

The pull server should validate the AgentId and make sure that the request is coming from a known (registered) node. As with a MOF request, the body contains no information, and the server responds with 404 or 200. In the case of a 200, the response is an application/octet-stream with the desired module's ZIP file.

### Reporting
When configured to use a Report Server, nodes will routinely - often several times in one consistency check - report in. The URL looks like this:

```
/Nodes(AgentId='agent-id-guid')/SendReport
```

The body of the request is an extensive JSON object containing the entire report for that node. The chapter on Reporting contains more details on what this object contains. Note that it's possible for a report's body to exceed the size that the web server permits for requests; in that case, you need to reconfigure your web server. 

One item included in the report is the "CurrentChecksum," which is included for each configuration that the node processes, and matches the contents of the MOF checksum file on the pull server. On its next check-in, this checksum will be sent.

The server responds with a 200 for successful reports, and actually sends a JSON response in the body:

```
                    {
                        "odata.metadata": "http://server-address:port/api-endpoint/$metadata#MSFT.DSCNodeStatus",
                        "value": "SavedReport"
                     }
```

So, you know, you at least know the report got through ;). The server can also respond with 400 (bad request) if it didn't accept the report, which usually means it didn't recognize the AgentId as one registered with it.

## The Database
The Pull Server's Devices.edb database essentially holds two tables. One, RegistrationData, includes:

* AgentId
* LCMVersion
* NodeName
* IPAddresses
* ConfigurationNames

In other words, about everything the node sent during registration. 

And, the StatusReport table, which is the Reporting Server role, you'll find:

* StartTime
* EndTime
* LastModifiedTime
* Id (the AgentId)
* OperationType
* RefreshMode
* LCMVersion
* ReportFormatVersion
* ConfigurationVersion
* IPAddresses
* RebootRequested
* Errors (a text field)
* StatusData (a text field containing JSON data)

