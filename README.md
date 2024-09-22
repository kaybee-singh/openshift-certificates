# openshift-certificates

1. Replacing the Ingress Certificate in Openshift cluster.


By default openshift is configured with `self-signed` cert for ingress. Many applications like browsers and security scanning tools do not recognize the ingress CA and as a result they display a  self-signed cert warning.

If we try to access the Console URl with **curl** then we get similar `self-signed` error

```bash
$ curl https://console-openshift-console.apps.lab.testing.com
curl: (60) SSL certificate problem: self-signed certificate in certificate chain
More details here: https://curl.se/docs/sslcerts.html

curl failed to verify the legitimacy of the server and therefore could not
establish a secure connection to it. To learn more about this situation and
how to fix it, please visit the web page mentioned above.

```

If required ingress cert can be replaced with the a custom cert signed by the in house CA or a certified CA. Below are steps listed to Configure ingress to use a custom certificate.

- Check if there is already a certificate specified for ingress. In below output we do not have any cert specified.

```bash
oc get ingresscontroller/default -n openshift-ingress-operator -o jsonpath='{.spec.clientTLS.clientCA}'
{"name":""}% 
```
- Let's create a certificate by following the steps from here if you not have a certificate.
- Before Creating the certificate for ingress below are the prequisites.
  1. A certificate with `.apps` subdomain
  2. subjectAltName extension showing `*.apps.<clustername>.<domain>.`
- Now create the certs by following below steps
```bash
yum install openssl -y
cd ;git clone https://github.com/kaybee-singh/openshift-certificates
cd openshift-certificates
mkdir -p ~/openshift-certificates/ca/{newcerts,private,cacerts};cd ~/openshift-certificates/ca
touch index.txt
echo 1000 > serial
```
- Generate CA private key and self signed cert.

```bash
openssl genrsa -out ~/openshift-certificates/ca/private/cakey.pem 2048
openssl req -new -x509 -config ~/openshift-certificates/ca_openssl.cnf -key ~/openshift-certificates/ca/private/cakey.pem -out ~/openshift-certificates/ca/cacerts/ca.pem -days 3650 -extensions v3_ca
``` 
- Create the server CSR to get signed by the CA created in above steps.

```bash

mkdir -p ~/openshift-certificates/servercerts/{private,certs};cd ~/openshift-certificates/servercerts
openssl genrsa -out ~/openshift-certificates/servercerts/private/serverkey.pem 2048
openssl req -new -key ~/openshift-certificates/servercerts/private/serverkey.pem -out ~/openshift-certificates/servercerts/server.csr -config ~/openshift-certificates/servercerts/server-openssl.cnf
```

- Verify that CSR has the DNS Altname section.
```bash
openssl req -in ~/openshift-certificates/servercerts/server.csr -noout -text
Certificate Request:
    Data:
        Version: 1 (0x0)
        Subject: C=US, ST=State, L=City, O=Organization, OU=Organizational Unit, CN=*.apps-crc.testing
        Subject Public Key Info:
            Public Key Algorithm: rsaEncryption
                Public-Key: (2048 bit)

                Exponent: 65537 (0x10001)
        Attributes:
            Requested Extensions:
                X509v3 Subject Alternative Name: 
                    DNS:*.apps-crc.testing, DNS:api.crc.testing
    Signature Algorithm: sha256WithRSAEncryption


```
- Sign the CSR by CA.
```bash
openssl ca -config ~/openshift-certificates/ca_openssl.cnf -in ~/openshift-certificates/servercerts/server.csr -out ~/openshift-certificates/servercerts/certs/server.crt -days 365 -extensions v3_req
```
Verify that certificate has got the DNS Alt name section.
```bash
openssl x509 -in ~/openshift-certificates/servercerts/certs/server.crt -noout -text
Certificate:
    Data:
        Version: 3 (0x2)
        Serial Number: 4100 (0x1004)
        Signature Algorithm: sha256WithRSAEncryption
        Issuer: C=US, ST=State, L=City, O=Organization, OU=Organizational Unit, CN=my-ca
        Validity
            Not Before: Aug  6 19:50:28 2024 GMT
            Not After : Aug  6 19:50:28 2025 GMT
        Subject: C=US, ST=State, O=Organization, CN=*.apps-crc.testing
        Subject Public Key Info:
        X509v3 extensions:
            X509v3 Subject Alternative Name: 
                DNS:*.apps-crc.testing, DNS:api.crc.testing
            X509v3 Subject Key Identifier: 
                3D:20:DC:79:81:63:C3:67:A3:C8:02:5F:CC:12:24:E4:C9:48:8A:8B

```
- Create a TLS secret to use with ingresscontroller.
```bash
oc create secret tls ingress-custom-secret --cert ~/openshift-certificates/servercerts/certs/server.crt --key ~/openshift-certificates/servercerts/private/serverkey.pem -n openshift-ingress
oc get secret -n openshift-ingress
```
- Patch the ingresscontroller configuration to use the secret
```bash
oc patch ingresscontroller.operator/default \\n-n openshift-ingress-operator --type=merge \\n--patch='{"spec": {"defaultCertificate": {"name": "ingress-custom-secret"}}}'
```
- Get the console URL.
```bash
console=`oc whoami --show-console`
```
- Try to access the console URL with CURL, it fails with error
```bash
curl $console
curl: (60) SSL certificate problem: unable to get local issuer certificate
More details here: https://curl.se/docs/sslcerts.html

curl failed to verify the legitimacy of the server and therefore could not
establish a secure connection to it. To learn more about this situation and
how to fix it, please visit the web page mentioned above.
```
- Now try to access it by providing the cacert. It should work this time.
```bash
curl $console --cacert ~/openshift-certificates/servercerts/certs/server.crt
<!DOCTYPE html>
<html lang="en" class="no-js">
  <head>
    <base href="/" />
    <meta charset="utf-8" />
    <meta http-equiv="X-UA-Compatible" content="IE=edge,chrome=1" />
       
    <title>Red Hat OpenShift</title>
    <meta name="application-name" content="Red Hat OpenShift" />
         
------------------------------------------------
-----------------------------------------------
```
