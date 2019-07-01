# Create a certificate authority

[form ](https://jamielinux.com/docs/openssl-certificate-authority/create-the-root-pair.html)

## Miscellaneous

To verify a received certificate rcvd.cert.pem with trusted certificate root-chain.cert.pem do:

```bash
openssl verify -CAfile roo-chain.cert.pem rcvd.cert.pem
```
## Creating CA

Acting as a certificate authority (CA) means dealing with cryptographic pairs of private keys and public certificates. 
The very first cryptographic pair we’ll create is the root pair. 
This consists of the root key (ca.key.pem) and root certificate (ca.cert.pem). This pair forms the identity of your CA.

Typically, the root CA does not sign server or client certificates directly. The root CA is only ever used to create 
one or more intermediate CAs, which are trusted by the root CA to sign certificates on their behalf. 
This is best practice. It allows the root key to be kept offline and unused as much as possible, 
as any compromise of the root key is disastrous.

*It’s best practice to create the root pair in a secure environment. Ideally, this should be on a fully encrypted, 
air gapped computer that is permanently isolated from the Internet. Remove the wireless card and fill the ethernet 
port with glue.*

Choose a directory for example `/root/ca` and cd into it. This would be our working directory

```bash
cd /root/ca
mkdir certs crl newcerts private
chmod 700 private
touch index.txt
echo 1000 > serial
```

Create `openssl.conf`

The `[ ca ]` section is mandatory. Here we tell OpenSSL to use the options from the [ CA_default ] section.

```bash
cat <<"EOF">openssl.conf
[ ca ]
# `man ca`
default_ca = CA_default
EOF
```

The `[ CA_default ]` section contains a range of defaults.

```bash
cat <<"EOF">>openssl.conf
[ CA_default ]
# Directory and file locations.
dir               = .
certs             = $dir/certs
crl_dir           = $dir/crl
new_certs_dir     = $dir/newcerts
database          = $dir/index.txt
serial            = $dir/serial
RANDFILE          = $dir/private/.rand

# The root key and root certificate.
private_key       = $dir/private/ca.key.pem
certificate       = $dir/certs/ca.cert.pem

# For certificate revocation lists.
crlnumber         = $dir/crlnumber
crl               = $dir/crl/ca.crl.pem
crl_extensions    = crl_ext
default_crl_days  = 30

# SHA-1 is deprecated, so use SHA-2 instead.
default_md        = sha256

name_opt          = ca_default
cert_opt          = ca_default
default_days      = 7300
preserve          = no
policy            = policy_strict
EOF
```
We’ll apply `policy_strict` for all root CA signatures, as the root CA is only being used to create intermediate CAs.

```bash
cat <<"EOF">>openssl.conf
[ policy_strict ]
# The root CA should only sign intermediate certificates that match.
# See the POLICY FORMAT section of `man ca`.
countryName             = match
stateOrProvinceName     = match
organizationName        = match
organizationalUnitName  = optional
commonName              = supplied
emailAddress            = optional
EOF
```
We’ll apply `policy_loose` for all intermediate CA signatures, as the intermediate CA is signing server and 
client certificates that may come from a variety of third-parties.

```bash
cat <<"EOF">>openssl.conf
[ policy_loose ]
# Allow the intermediate CA to sign a more diverse range of certificates.
# See the POLICY FORMAT section of the `ca` man page.
countryName             = optional
stateOrProvinceName     = optional
localityName            = optional
organizationName        = optional
organizationalUnitName  = optional
commonName              = supplied
emailAddress            = optional
EOF
```

Options from the `[ req ]` section are applied when creating certificates or certificate signing requests.

```bash
cat <<"EOF">>openssl.conf
[ req ]
# Options for the `req` tool (`man req`).
default_bits        = 2048
distinguished_name  = req_distinguished_name
string_mask         = utf8only

# SHA-1 is deprecated, so use SHA-2 instead.
default_md          = sha256

# Extension to add when the -x509 option is used.
x509_extensions     = v3_ca
EOF
```

The `[ req_distinguished_name ]` section declares the information normally required in a certificate signing request.
You can optionally specify some defaults.

```bash
cat <<"EOF">>openssl.conf
[ req_distinguished_name ]
# See <https://en.wikipedia.org/wiki/Certificate_signing_request>.
countryName                     = Country Name (2 letter code)
stateOrProvinceName             = State or Province Name
localityName                    = Locality Name
0.organizationName              = Organization Name
organizationalUnitName          = Organizational Unit Name
commonName                      = Common Name
emailAddress                    = Email Address

# Optionally, specify some defaults.
countryName_default             = GB
stateOrProvinceName_default     = England
localityName_default            =
0.organizationName_default      = penvpn
#organizationalUnitName_default =
#emailAddress_default           =
EOF
```

The next few sections are extensions that can be applied when signing certificates. For example, 
passing the `-extensions v3_ca` command-line argument will apply the options set in [ v3_ca ].

We’ll apply the v3_ca extension when we create the root certificate.

```bash
cat <<"EOF">>openssl.conf
[ v3_ca ]
# Extensions for a typical CA (`man x509v3_config`).
subjectKeyIdentifier = hash
authorityKeyIdentifier = keyid:always,issuer
basicConstraints = critical, CA:true
keyUsage = critical, digitalSignature, cRLSign, keyCertSign
EOF
```
We’ll apply the `v3_ca_intermediate` extension when we create the intermediate certificate. 
pathlen:0 ensures that there can be no further certificate authorities below the intermediate CA.

```bash
cat <<"EOF">>openssl.conf
[ v3_intermediate_ca ]
# Extensions for a typical intermediate CA (`man x509v3_config`).
subjectKeyIdentifier = hash
authorityKeyIdentifier = keyid:always,issuer
basicConstraints = critical, CA:true, pathlen:0
keyUsage = critical, digitalSignature, cRLSign, keyCertSign
EOF
```
We’ll apply the `usr_cert extension` when signing client certificates, such as those used for remote user authentication.

```bash
cat <<"EOF">>openssl.conf
[ usr_cert ]
# Extensions for client certificates (`man x509v3_config`).
basicConstraints = CA:FALSE
nsCertType = client, email
nsComment = "OpenSSL Generated Client Certificate"
subjectKeyIdentifier = hash
authorityKeyIdentifier = keyid,issuer
keyUsage = critical, nonRepudiation, digitalSignature, keyEncipherment
extendedKeyUsage = clientAuth, emailProtection
EOF
```
We’ll apply the `server_cert` extension when signing server certificates, such as those used for web servers.

```bash
cat <<"EOF">>openssl.conf
[ server_cert ]
# Extensions for server certificates (`man x509v3_config`).
basicConstraints = CA:FALSE
nsCertType = server
nsComment = "OpenSSL Generated Server Certificate"
subjectKeyIdentifier = hash
authorityKeyIdentifier = keyid,issuer:always
keyUsage = critical, digitalSignature, keyEncipherment
extendedKeyUsage = serverAuth
EOF
```
The `crl_ext` extension is automatically applied when creating certificate revocation lists.

```bash
cat <<"EOF">>openssl.conf
[ crl_ext ]
# Extension for CRLs (`man x509v3_config`).
authorityKeyIdentifier=keyid:always
EOF
```
We’ll apply the ocsp extension when signing the Online Certificate Status Protocol (OCSP) certificate.

```bash
cat <<"EOF">>openssl.conf
[ ocsp ]
# Extension for OCSP signing certificates (`man ocsp`).
basicConstraints = CA:FALSE
subjectKeyIdentifier = hash
authorityKeyIdentifier = keyid,issuer
keyUsage = critical, digitalSignature
extendedKeyUsage = critical, OCSPSigning
EOF
```

## Running commands

Create the root key

Create the root key (ca.key.pem) and keep it absolutely secure. Anyone in possession of the root key can issue trusted 
certificates. 
Encrypt the root key with AES 256-bit encryption and a strong password.

*Use 4096 bits for all root and intermediate certificate authority keys. You’ll still be able to sign server and client certificates of a shorter length.*

```bash
openssl genrsa -aes256 -out private/ca.key.pem 4096
```

Enter pass phrase for ca.key.pem: secretpassword
Verifying - Enter pass phrase for ca.key.pem: secretpassword
```bash
chmod 400 private/ca.key.pem
```

Create the root certificate

Use the root key `ca.key.pem` to create a root certificate `ca.cert.pem`. Give the root certificate a long expiry date, 
such as twenty years. Once the root certificate expires, all certificates signed by the CA become invalid.

*Whenever you use the `req` tool, you must specify a configuration file to use with the `-config` option, 
otherwise `OpenSSL` will default to `/etc/pki/tls/openssl.conf`*

```bash
openssl req -config openssl.conf -key private/ca.key.pem -new -x509 -days 7300 -sha256 -extensions v3_ca -out certs/ca.cert.pem
```

Enter pass phrase for ca.key.pem: secretpassword
You are about to be asked to enter information that will be incorporated
into your certificate request.

Country Name (2 letter code) [XX]:GB
State or Province Name []:England
Locality Name []:
Organization Name []:Alice Ltd
Organizational Unit Name []:Alice Ltd Certificate Authority
Common Name []:Alice Ltd Root CA
Email Address []:

```bash
chmod 444 certs/ca.cert.pem
```

Verify the root certificate

```bash
openssl x509 -noout -text -in certs/ca.cert.pem
```

The output shows:

1. the Signature Algorithm used
2. the dates of certificate Validity
3. the Public-Key bit length
4. the Issuer, which is the entity that signed the certificate
5. the Subject, which refers to the certificate itself
6. The Issuer and Subject are identical as the certificate is self-signed. Note that all root certificates are self-signed.

```
Signature Algorithm: sha256WithRSAEncryption
    Issuer: C=GB, ST=England,
            O=Alice Ltd, OU=Alice Ltd Certificate Authority,
            CN=Alice Ltd Root CA
    Validity
        Not Before: Apr 11 12:22:58 2015 GMT
        Not After : Apr  6 12:22:58 2035 GMT
    Subject: C=GB, ST=England,
             O=Alice Ltd, OU=Alice Ltd Certificate Authority,
             CN=Alice Ltd Root CA
    Subject Public Key Info:
        Public Key Algorithm: rsaEncryption
            Public-Key: (4096 bit)
```

The output also shows the `X509v3` extensions. We applied the `v3_ca` extension, so the options from `[ v3_ca ]` 
should be reflected in the output.

```
X509v3 extensions:
    X509v3 Subject Key Identifier:
        38:58:29:2F:6B:57:79:4F:39:FD:32:35:60:74:92:60:6E:E8:2A:31
    X509v3 Authority Key Identifier:
        keyid:38:58:29:2F:6B:57:79:4F:39:FD:32:35:60:74:92:60:6E:E8:2A:31

    X509v3 Basic Constraints: critical
        CA:TRUE
    X509v3 Key Usage: critical
        Digital Signature, Certificate Sign, CRL Sign
```

## Create the intermediate pair

An intermediate certificate authority (CA) is an entity that can sign certificates on behalf of the root CA. 
The root CA signs the intermediate certificate, forming a chain of trust.

The purpose of using an intermediate CA is primarily for security. The root key can be kept offline and used as 
infrequently as possible. If the intermediate key is compromised, the root CA can revoke the intermediate certificate 
and create a new intermediate cryptographic pair.

Prepare the directory

The root CA files are kept in `/root/ca`. Choose a different directory (`/root/ca/intermediate`) to store the 
intermediate CA files.

```bash
mkdir intermediate
```

Create the same directory structure used for the root CA files. It’s convenient to also create a `csr` 
directory to hold certificate signing requests.

```bash
cd intermediate
mkdir certs crl csr newcerts private
chmod 700 private
touch index.txt
echo 1000 > serial
```
Add a crlnumber file to the intermediate CA directory tree. crlnumber is used to keep track of certificate revocation lists.

```bash
echo 1000 > /root/ca/intermediate/crlnumber
```
Copy the previous `openssl.conf` configuration file to intermediate/openssl.conf and make these five changes:

```
[ CA_default ]
dir             = intermediate
private_key     = $dir/private/intermediate.key.pem
certificate     = $dir/certs/intermediate.cert.pem
crl             = $dir/crl/intermediate.crl.pem
policy          = policy_loose
```

Create the intermediate key (intermediate.key.pem). Encrypt the intermediate key with AES 256-bit encryption and a strong password.

```bash
cd ..
openssl genrsa -aes256 -out intermediate/private/intermediate.key.pem 4096
```
Enter pass phrase for intermediate.key.pem: secretpassword
Verifying - Enter pass phrase for intermediate.key.pem: secretpassword
```
chmod 400 intermediate/private/intermediate.key.pem
```

Create the intermediate certificate

Use the intermediate key to create a certificate signing request (CSR). 
The details should generally match the root CA. The Common Name, however, must be different.

*Make sure you specify the intermediate CA configuration file (intermediate/openssl.conf).*

```bash
# openssl req -config intermediate/openssl.conf -new -sha256 -key intermediate/private/intermediate.key.pem -out intermediate/csr/intermediate.csr.pem
```

Enter pass phrase for intermediate.key.pem: secretpassword
You are about to be asked to enter information that will be incorporated
into your certificate request.
```
Country Name (2 letter code) [XX]:GB
State or Province Name []:England
Locality Name []:
Organization Name []:Alice Ltd
Organizational Unit Name []:Alice Ltd Certificate Authority
Common Name []:Alice Ltd Intermediate CA
Email Address []:
```
To create an intermediate certificate, use the root CA with the v3_intermediate_ca extension to sign the intermediate CSR. 
The intermediate certificate should be valid for a shorter period than the root certificate. Ten years would be reasonable.

-------------------
This time, specify the root CA configuration file (openssl.conf).
-------------------

```bash
openssl ca -config openssl.conf -extensions v3_intermediate_ca -days 3650 -notext -md sha256 -in -out intermediate/certs/intermediate.cert.pem
```

Enter pass phrase for ca.key.pem: secretpassword
Sign the certificate? [y/n]: y
```bash
# chmod 444 intermediate/certs/intermediate.cert.pem
```

The index.txt file is where the OpenSSL ca tool stores the certificate database. 
Do not delete or edit this file by hand. It should now contain a line that refers to the intermediate certificate.

bash```
# cat index.txt
V 250408122707Z 1000 unknown ... /CN=Alice Ltd Intermediate CA
```
Verify the intermediate certificate

As we did for the root certificate, check that the details of the intermediate certificate are correct.

```bash
openssl x509 -noout -text -in intermediate/certs/intermediate.cert.pem
```

Verify the intermediate certificate against the root certificate. An OK indicates that the chain of trust is intact.

```bash
openssl verify -CAfile certs/ca.cert.pem intermediate/certs/intermediate.cert.pem
```
intermediate.cert.pem: OK

## Create the certificate chain file

When an application (eg, a web browser) tries to verify a certificate signed by the intermediate CA, 
it must also verify the intermediate certificate against the root certificate. 
To complete the chain of trust, create a CA certificate chain to present to the application.

To create the CA certificate chain, concatenate the intermediate and root certificates together. 
We will use this file later to verify certificates signed by the intermediate CA.

```bash
cat intermediate/certs/intermediate.cert.pem certs/ca.cert.pem > intermediate/certs/ca-chain.cert.pem
chmod 444 intermediate/certs/ca-chain.cert.pem
```

* Our certificate chain file must include the root certificate because no client application knows about it yet. 
A better option, particularly if you’re administrating an intranet, is to install your root certificate on every client 
that needs to connect. In that case, the chain file need only contain your intermediate certificate. *

## Sign server and client certificates

We will be signing certificates using our intermediate CA. You can use these signed certificates in a variety of 
situations, such as to secure connections to a web server or to authenticate clients connecting to a service.

* The best practice is to create a separate directory outside of the root ca and create the exact structure we used to create
intermediate folder setup.
Then to create csr, cd to that directory and execute commands *

* The steps below are from your perspective as the certificate authority. A third-party, however, 
can instead create their own private key and certificate signing request (CSR) without revealing their private key to you. 
They give you their CSR, and you give back a signed certificate. In that scenario, skip the genrsa and req commands.*

Create a key

Our root and intermediate pairs are 4096 bits. Server and client certificates normally expire after one year, 
so we can safely use 2048 bits instead.

* Although 4096 bits is slightly more secure than 2048 bits, it slows down TLS handshakes and significantly 
increases processor load during handshakes. For this reason, most websites use 2048-bit pairs.*

If you’re creating a cryptographic pair for use with a web server (eg, Apache), 
you’ll need to enter this password every time you restart the web server. 
You may want to omit the -aes256 option to create a key without a password.

```bash
openssl genrsa -aes256 -out private/www.example.com.key.pem 2048
chmod 400 private/www.example.com.key.pem
```

Create a certificate

Use the private key to create a certificate signing request (CSR). 
The CSR details don’t need to match the intermediate CA. 
For server certificates, the Common Name must be a fully qualified domain name (eg, www.example.com),
whereas for client certificates it can be any unique identifier (eg, an e-mail address). 
Note that the Common Name cannot be the same as either your root or intermediate certificate.

```bash
openssl req -config openssl.conf -key private/www.example.com.key.pem -new -sha256 -out csr/www.example.com.csr.pem
```
Enter pass phrase for www.example.com.key.pem: secretpassword
You are about to be asked to enter information that will be incorporated
into your certificate request.
```
Country Name (2 letter code) [XX]:US
State or Province Name []:California
Locality Name []:Mountain View
Organization Name []:Alice Ltd
Organizational Unit Name []:Alice Ltd Web Services
Common Name []:www.example.com
Email Address []:
```

After third parties created their CSR like above, they send their CSRs to you and you sign them with your 
intermediate certificate. Copy them to the `intermediate/csr` folder

To create a certificate, use the intermediate CA to sign the CSR. If the certificate is going to be used on a server, 
use the server_cert extension. If the certificate is going to be used for user authentication, 
use the usr_cert extension. Certificates are usually given a validity of one year, though a CA will typically give a 
few days extra for convenience.

```bash
openssl ca -config intermediate/openssl.conf -extensions server_cert -days 375 -notext -md sha256 -in intermediate/csr/www.example.com.csr.pem -out intermediate/certs/www.example.com.cert.pem

chmod 444 intermediate/certs/www.example.com.cert.pem
```
The intermediate/index.txt file should contain a line referring to this new certificate.
```
V 160420124233Z 1000 unknown ... /CN=www.example.com
```

Verify the certificate

```bash
openssl x509 -noout -text -in intermediate/certs/www.example.com.cert.pem
```
```
Signature Algorithm: sha256WithRSAEncryption
    Issuer: C=GB, ST=England,
            O=Alice Ltd, OU=Alice Ltd Certificate Authority,
            CN=Alice Ltd Intermediate CA
    Validity
        Not Before: Apr 11 12:42:33 2015 GMT
        Not After : Apr 20 12:42:33 2016 GMT
    Subject: C=US, ST=California, L=Mountain View,
             O=Alice Ltd, OU=Alice Ltd Web Services,
             CN=www.example.com
    Subject Public Key Info:
        Public Key Algorithm: rsaEncryption
            Public-Key: (2048 bit)
```
            
The output will also show the X509v3 extensions. When creating the certificate, you used either the server_cert or usr_cert extension. The options from the corresponding configuration section will be reflected in the output.
```
X509v3 extensions:
    X509v3 Basic Constraints:
        CA:FALSE
    Netscape Cert Type:
        SSL Server
    Netscape Comment:
        OpenSSL Generated Server Certificate
    X509v3 Subject Key Identifier:
        B1:B8:88:48:64:B7:45:52:21:CC:35:37:9E:24:50:EE:AD:58:02:B5
    X509v3 Authority Key Identifier:
        keyid:69:E8:EC:54:7F:25:23:60:E5:B6:E7:72:61:F1:D4:B9:21:D4:45:E9
        DirName:/C=GB/ST=England/O=Alice Ltd/OU=Alice Ltd Certificate Authority/CN=Alice Ltd Root CA
        serial:10:00

    X509v3 Key Usage: critical
        Digital Signature, Key Encipherment
    X509v3 Extended Key Usage:
        TLS Web Server Authentication
 ```       
Use the CA certificate chain file we created earlier (ca-chain.cert.pem) to verify that the new certificate has a valid chain of trust.

```bash
openssl verify -CAfile intermediate/certs/ca-chain.cert.pem \
      intermediate/certs/www.example.com.cert.pem
```
`www.example.com.cert.pem: OK`
