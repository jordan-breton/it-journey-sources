---
date: 2023-03-05
categories:
- SSL
- Sysadmin
- Dev environment
- Test environment
---

![SSL certificates in dev & test environments cover](/assets/images/blog/SSL-certificate-dev-test-environment/cover.jpg){ .cover }

# SSL certificates in dev & test environments

In simplest cases and for simplest projects, we don't bother with SSL certificates... until we start to
work with some feature that won't work without it like WebRTC or PWA. Same thing if you want to 
start working with third party APIs.

Generating self-signed certificate quickly become mandatory, and managing them between several projects
can be challenging if you don't do it properly...

Have you ever heard about Root Certificates? It makes our lives better :wink:

<!-- more -->

In dev environments for example, using self-signed certificates is mandatory to play with WebRTC features,
`Access-Control` headers and soo one.

This section will try to cover basics about setting up SSL root certificate and per-site certificates.

### Generating a Root Certificate

We'll work with [OpenSSL](https://www.openssl.org/).

To keep things tidy, we save each certificate in the same folder `~/SSLConfig` :

``` bash
mkdir -p ~/SSLConfig/cert/CA
cd ~/SSLConfig/cert/CA

openssl genrsa -out CA.key -des3 2048
openssl req -x509 -sha256 -new -nodes -days 3650 -key CA.key -out CA.pem
```

Now, you'll be able to generate as many certificates as you want, and you will just have to add your `CA.crt` to trusted CA lists to any device/browser
you want to be able to access to your services.

!!! example "Installing the root certificate in ubuntu"

    ```bash
    mkdir -p /usr/local/share/ca-certificates #if not exists
    sudo cp ~/SSLConfig/cert/CA/CA.crt /usr/local/share/ca-certificates
    sudo update-ca-certificates
    ```


### Generating a certificate

Since we generated a [root certificate](#generating-a-root-certificate), let's create our SSL certificate for our `demo.dev.local`
website :

``` bash
mkdir demo.dev.local
cd ./demo.dev.local
```

Then, let's create an `ext` file used to create our certificate :

=== "demo.dev.local.ext"

    ``` hl_lines="7 8"
    authorityKeyIdentifier = keyid,issuer
    basicConstraints = CA:FALSE
    keyUsage = digitalSignature, nonRepudiation, keyEncipherment, dataEncipherment
    subjectAltName = @alt_names
    
    [alt_names]
    DNS.1 = demo.dev.local
    IP.1 = 192.168.1.14
    ```

!!! note

    The last line allow you to use your SSL certificate when querying `https://192.168.1.14`. 
    As such, it's optional.

Then run the following commands to generate all required files :

``` bash
openssl genrsa -out my-birth-plan.dev.local.key -des3 2048
openssl req -new -key demo.dev.local.key -out demo.dev.local.csr
openssl x509 -req -in demo.dev.local.csr \
        -CA ../CA.pem -CAkey ../CA.key -CAcreateserial \
        -days 3650 -sha256 -extfile demo.dev.local.ext \
        -out demo.dev.local.crt
openssl rsa -in demo.dev.local.key -out demo.dev.local.decrypted.key
```

!!! warning

    When generating your certificates, you will be printed some questions about certificate 
    (Common Name, Organisation infos, mail, etc.).

    The CN (Common Name) must be **UNIQUE** for **EACH** certificate, including the root **CA**. 
    If not, NodeJS will reject your certificates and mark them as `self-signed`
    as described [here](https://stackoverflow.com/questions/68896243).

Now, you should have this folder structure :


<div disable-linenums></div>

``` bash
~/SSLConfig
└── cert
    └── CA
        ├── CA.key
        ├── CA.pem
        ├── CA.srl #This file track the latest serial number available for new certs
        └── demo.dev.local
            ├── demo.dev.local.crt # Public key.
            ├── demo.dev.local.csr # Contain cert information, used only at generation time.
            ├── demo.dev.local.decrypted.key # Decrypted key to provide to each server you want to receive SSL requests.
            ├── demo.dev.local.ext # The config file.
            └── demo.dev.local.key # The certificate key.
```

!!! example "Configuring the apache2 proxy with our certificate"

    ```
    <VirtualHost *:8080>
        ServerName demo.dev.local

        SSLEngine on
        SSLCertificateFile /home/[user]/SSLConfig/cert/CA/demo.dev.local/demo.dev.local.crt
        SSLCertificateKeyFile /home/[user]/SSLConfig/cert/CA/demo.dev.local/demo.dev.local.decrypted.key

        ProxyPreserveHost On
        ProxyRequests Off
        ProxyVia Off
        ProxyPass / ws://127.0.0.1:8081 retry=0
        ProxyPassReverse / ws://127.0.0.1:8081
    </VirtualHost>
    ```

To be able to use any of your certificates in a browser, you must [install the root certificate](#generating-a-root-certificate)
into your browser / system / smartphone, and all the generated certificates signed with this CA will be trusted too.