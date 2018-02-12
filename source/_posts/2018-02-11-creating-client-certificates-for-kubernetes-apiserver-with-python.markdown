---
layout: post
title: "Creating client certificates for kubernetes apiserver with python"
date: 2018-02-11 11:13:29 +0700
comments: true
categories: 
---
Kubernetes features a complex, multi-user permissions system to control access to the cluster. 
Those options are bound to contexts and users can be given access to those contexts. 
For that, a user must authenticate itself to the api-server and there are various ways for doing that.

[The authentication options are documented here](https://kubernetes.io/docs/admin/authentication/)

One of those options is authentication through x509 client certificates.

A user wanting to authenticate needs to have a certificate with his/her username as the subject's commonName. 
And the certificate needs to be signed by the cluster's ca. 
Although the concept is quite easy, creating certificates using the openssl-cli often proves to be difficult.
So here is a python script that creates/takes a ca and creates new client certificates for a kubernetes user.

Be sure apiserver runs with `--client-ca-file=yourca.crt`

```python
#!/usr/bin/env python

from socket import gethostname

import yaml
import random
from OpenSSL import crypto
import base64

CA_CERT_FILE = "ca.crt"
CA_KEY_FILE = "ca.key"
CLIENT_CERT_FILE = "client.crt"
CLIENT_KEY_FILE = "client.key"


def generate_self_signed_ca():
    k = crypto.PKey()
    k.generate_key(crypto.TYPE_RSA, 1024)

    # create a self-signed ca
    ca = crypto.X509()
    ca.get_subject().C = "DE"
    ca.get_subject().ST = "Duesseldorf"
    ca.get_subject().L = "Duesseldorf"
    ca.get_subject().O = "Dummy GmbH"
    ca.get_subject().OU = "Dummy GmbH"
    ca.get_subject().CN = gethostname()
    ca.set_serial_number(1000)
    ca.gmtime_adj_notBefore(0)
    ca.gmtime_adj_notAfter(10 * 365 * 24 * 60 * 60)
    ca.set_issuer(ca.get_subject())
    ca.set_pubkey(k)

    ca.add_extensions([
        crypto.X509Extension(b"basicConstraints", True,
                             b"CA:TRUE, pathlen:0"),
        crypto.X509Extension(b"keyUsage", True,
                             b"keyCertSign, cRLSign"),
        crypto.X509Extension(b"subjectKeyIdentifier", False, b"hash",
                             subject=ca),
    ])
    ca.add_extensions([
        crypto.X509Extension(b"authorityKeyIdentifier", False, b"keyid:always", issuer=ca)
    ])

    ca.sign(k, 'sha1')

    open(CA_CERT_FILE, "wb").write(
        crypto.dump_certificate(crypto.FILETYPE_PEM, ca))
    open(CA_KEY_FILE, "wb").write(
        crypto.dump_privatekey(crypto.FILETYPE_PEM, k))

    return ca, k


def load_cert():
    with open(CA_CERT_FILE, "rb") as certfile:
        catext = certfile.read()

    with open(CA_KEY_FILE, "rb") as keyfile:
        keytext = keyfile.read()

    return (
        crypto.load_certificate(crypto.FILETYPE_PEM, catext),
        crypto.load_privatekey(crypto.FILETYPE_PEM, keytext, None)
    )


def generate_client_cert(ca_cert, ca_key, username):
    client_key = crypto.PKey()
    client_key.generate_key(crypto.TYPE_RSA, 2048)

    client_cert = crypto.X509()
    client_cert.set_version(2)
    client_cert.set_serial_number(random.randint(50000000, 100000000))

    client_subj = client_cert.get_subject()
    client_subj.commonName = username
    # client_subj.organizationName = "user-group"

    client_cert.add_extensions([
        crypto.X509Extension(b"basicConstraints", False, b"CA:FALSE"),
        crypto.X509Extension(b"subjectKeyIdentifier", False, b"hash", subject=client_cert),
    ])

    client_cert.add_extensions([
        crypto.X509Extension(b"authorityKeyIdentifier", False, b"keyid:always", issuer=ca_cert),
        crypto.X509Extension(b"extendedKeyUsage", False, b"clientAuth"),
        crypto.X509Extension(b"keyUsage", False, b"digitalSignature"),
    ])

    client_cert.gmtime_adj_notBefore(0)
    client_cert.gmtime_adj_notAfter(10 * 365 * 24 * 60 * 60)

    client_cert.set_subject(client_subj)

    client_cert.set_issuer(ca_cert.get_issuer())
    client_cert.set_pubkey(client_key)
    client_cert.sign(ca_key, 'sha256')

    with open(CLIENT_CERT_FILE, "wb") as f:
        f.write(crypto.dump_certificate(crypto.FILETYPE_PEM, client_cert))

    with open(CLIENT_KEY_FILE, "wb") as f:
        f.write(crypto.dump_privatekey(crypto.FILETYPE_PEM, client_key))

    return client_cert, client_key


def new_client_cert(username):
    ca_cert, ca_key = None, None

    try:
        ca_cert, ca_key = load_cert()
    except Exception as e:
        print(e)
        ca_cert, ca_key = generate_self_signed_ca()

    client_cert, client_key = generate_client_cert(ca_cert, ca_key, username)

    return client_cert, client_key, username


def create_user_config(client_cert, client_key, username):
    with open("admin.conf") as adminconfigtext:
        config = yaml.load(adminconfigtext)

    config["users"][0]["name"] = username
    config["users"][0]["user"]["client-certificate-data"] = base64.b64encode(
        crypto.dump_certificate(crypto.FILETYPE_PEM, client_cert)).decode()
    config["users"][0]["user"]["client-key-data"] = base64.b64encode(
        crypto.dump_privatekey(crypto.FILETYPE_PEM, client_key)).decode()

    config["contexts"][0]["context"]["user"] = username
    config["contexts"][0]["name"] = username + "@kubernetes"
    config["current-context"] = username + "@kubernetes"

    print(config["users"][0]["user"]["client-certificate-data"])

    with open("user.conf", "w") as userconfigtext:
        yaml.dump(config, userconfigtext)


if __name__ == "__main__":
    client_cert, client_key, username = new_client_cert("myusername")
    try:
        create_user_config(client_cert, client_key, username)
    except FileNotFoundError:
        print("admin.conf needed to create user.conf")

```

If you have a `admin.conf` file, then the script also generates a kubectl configfile for the user.

```
kubectl --kubeconfig user.conf 
```

Easy peasy. Hope it'll be useful to someone.