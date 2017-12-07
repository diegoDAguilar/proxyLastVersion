## Boulder-----Server VM-----DNO Proxy-----Client

## Steps 3 and 4 occur at the same time but one is from Client's point of view and the other from Boulder's
## 1. Client asks for certificate 
### 1. Client uses curl to ask for a certificate to the proxy.

`curl --cacert $proxyCert -H Content-Type: application/json -X POST -d $fullCsr https://certProxy:443/star/registration`

@proxyCert: certificate signed by 'certProxy', that is the name assigned in /etc/hosts to proxy's IP.   
@fullCsr contains: {"csr":".........", "lifetime":365, "validity":24}

### 2. DNO proxy's listens to the request

DNO is listening at :443 /star/registration, when a json is received it decodes it and checks whether requested lifetime and validity
are OK, then it parses the csr and extracts the Domain requested. It compares it with a list of valid domains to delegate.
If these checks are successfull it sends a message with the final parameters used by the DNO(they may have changed), then another message with a header: StatusCreated, and a body: Location: .............
Location contains the *completion url*.

Tweaked certbot is executed, a coroutine is created to serve requests to *completion url* and client's request information is storaged.

### 3.1 Certbot is executed creating a default account using a fake e-mail and client's csr 

Certbot runs as usual but with extra fields: 
```
{
    "recurrent": true,
    "recurrent-start-date": "2016-01-01T00:00:00Z",
    "recurrent-end-date": "2017-01-01T00:00:00Z",
    "recurrent-certificate-validity": 604800
  }
 
 ```
 
 These extra fields are added to the new-certificate request that is sent to Boulder.
 
 ### 3.2 Client receives *completion url* from step 2
 
 Client extracts the url from `Location: url`  and does a new curl to this *completion url*. It returns a json struct containing:
```
{
    "status": "valid", 
    "lifetime": 365, // lifetime of the registration in days,
                     
    "certificates": "https://acme-server.example.org/certificates/A51A3"
}

```

Last field contains the URI(Located in the server) from where certificate is ready to download and also where it will be renewed.

A GET to that URI will return the certificate and show it in screen plus storage it in the directory passed as argument with the desired
name. If a certificate with the same properties exists in that file it will replace it, as that is the nature of STAR.

### 4 Boulder receives the certificate request with the extra fields 

When Boulder's Web Front End *WFE* receives a new valid request with recurrent:true it executes STAR.
Relevant information is storaged. These info includes the *csr*, *validity* and the *renewal uri*(which has just been randomly assigned
by the *WFE* using UUID v4).

Boulder continues executing but using the validity extracted from *recurrent-star-date* and *recurrent-end-date* as the certificate's
lifetime.
These certificate is storaged together with the relevant info from the request and send back to the DNO.
At the same time the cert is made available at renewal uri. 

### 5 DNO receives cert and GETs the renewal uri 

DNO receives the certificate but the renewal uri remains unknown to him.
It get's that first certificate UUID and makes a GET to server's port :9898 that will return him the real renewal-uri.
This uri is send back to the Client in step 3.2 and also storaged in the proxy.

