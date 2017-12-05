### Boulder-----Server VM-----DNO Proxy-----Client

### 1. Client asks for certificate
1. Client uses curl to ask for a certificate to the proxy.

curl --cacert $proxyCert -H Content-Type: application/json -X POST -d $fullCsr https://certProxy:443/star/registration

@proxyCert: certificate signed by 'certProxy', that is the name assigned in /etc/hosts to proxy's IP.
@fullCsr contains: {"csr":".........", "lifetime":365, "validity":24}

2. DNO proxy's listens to the request

DNO is listening at :443 /star/registration, when a json is received it decodes it and checks whether requested lifetime and validity
are OK, then it parses the csr and extracts the Domain requested. It compares it with a list of valid domains to delegate.
If these checks are successfull it sends a header: StatusCreated, and a body: Location: .............
Location contains the *completion url*.

Tweaked certbot is executed, a coroutine is created to serve requests to *completion url* and client information is saved.
