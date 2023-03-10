# NIRVai CFSSL

- [scripting architecture & guidance](../scripts/README.md)
- [source code](https://github.com/nirv-ai/scripts/blob/develop/cloudflare/script.ssl.sh)
- [configuration](https://github.com/nirv-ai/configs/tree/develop/cfssl)
- [based heavily on this doc by hashicorp](https://developer.hashicorp.com/nomad/tutorials/transport-security/security-enable-tls)

## RACEXP

- [NIRVai security project board](https://github.com/orgs/nirv-ai/projects/6/views/1?filterQuery=repo%3A%22nirv-ai%2Fsecurity%22)
- [RACEXP docs](https://github.com/noahehall/theBookOfNoah/blob/master/0current/architectural%20thinking/0racexp.md)

## NIRVai Mindset

> if it doesnt copypasta, it doesnt belong in your stack

https://user-images.githubusercontent.com/10324554/213013912-dd5e8229-d7b0-4124-a67e-648f4f3644e9.mp4

## WHY CFSSL ?

- there are quite few respectable alternatives: [ejcba](https://www.ejbca.org/),[openvpns easy-rsa](https://github.com/OpenVPN/easy-rsa), [smallsteps step-ca](https://github.com/smallstep/certificates) - even vault has a [pki secrets engine](https://developer.hashicorp.com/vault/docs/secrets/pki)
- we chose [cfssl](https://blog.cloudflare.com/introducing-cfssl/) to achieve the following goals:
  - straight forward to install and automate
  - completely configurable with 0 opinions
  - self-contained while effectively integrating with our stack (specifically vault, consul and nomad)

## Setting up CFSSL

### REQUIREMENTS

```sh
# cfssl: option 1 install from source @see https://github.com/cloudflare/cfssl
# cfssl: option 2 install via apt-get @see https://packages.ubuntu.com/search?keywords=golang-cfssl
# jq: @see https://stedolan.github.io/jq/manual/

# directory structure matches:
├── scripts             # @see https://github.com/nirv-ai/scripts
├── configs             # @see https://github.com/nirv-ai/configs
│   └── cfssl
│   │   ├── cfssl.json  # the default cfssl.json
│   │   └── $CA_CN
│   │   │   ├── csr.cli.cli.json       # conf for CLI TLS certs
│   │   │   ├── csr.client.client.json # conf for client TLS certs
│   │   │   ├── csr.root.ca.json       # conf for root private cert authority
│   │   │   ├── csr.server.server.json # conf for server TLS certs
│   │   │   ├── custom.cffsl.json      # optional cffsl json if not using default
├── secrets             # chroot jail, a temporary folder or private git repo
│   └── $CA_CN
│   │   └── tls         # we will persist created files to this directory
```

### INTERFACE

```sh
## set the cert authority's common name, e.g.
export CA_CN=mesh.nirv.ai

## vars are available for modification
# export CA_PEM_NAME=ca
# export CLI_NAME=cli
# export CLIENT_NAME=client
# export CONFIG_DIR_NAME=configs
# export SECRET_DIR_NAME=secrets
# export SERVER_NAME=server
# export TLS_DIR_NAME=tls

## lookup order for $CFSSL_CONFIG_NAME
## if: configs/$CA_CN/$CFSSL_CONFIG_NAME
## elif: configs/$CFSSL_CONFIG_NAME
## else: configs/cfssl.json
# export CFSSL_CONFIG_NAME=cfssl.json

```

### USAGE

```sh
### prefix all cmds with script.ssl.sh

### ROOT CA
# create root ca and save as ca{-key}.{pem,csr}
create rootca

# create root ca for a different CA_CN and save as ca{-key}.{pem,csr}
create rootca mesh.prod.nirv.ai

# create root ca for a different CA_CN but save as somethingelse{-key}.{pem,csr}
create rootca mesh.test.nirv.ai somethingelse


### FYI ON SERVER/CLIENT/CLI CERT CREATION
## the TOTAL always represents how many SHOULD exist
## not necessarily how many you want to create
## e.g. if you want 10 and 0 exists
# create 10 will create 10 new certs starting at 0
## e.g. if you want 10 and 5 already exist
# create 10 will create 5 additional certs in index order (0-9) filling in any gaps
# that way you can delete cert at index X (e.g. its been compromised) and it will be recreated

### SERVER CERT
# create 1 server cert and save as server-0{-key}.{pem,csr}
create server
# create arbitrary amount of server certs using above vars
create server 7

# create arbitrary amount of server certs specifying options
# total ca_cn ca_name cert_name config_name
create server 77 mesh.nirv.ai ca server server.cfssl.json


### CLIENT CERT
# create 1 client cert and save as client-0{-key}.{pem,csr}
create client
# create arbitrary amount of client certs
create client 7

# create arbitrary amount of client certs specifying options
# total ca_cn ca_name cert_name config_name
create client 77 mesh.nirv.ai ca client client.cfssl.json


### BROWSER P12 CERT
# create a p12 cert for your browser so you can access UIs over https
# you will need to provide an arbitrary password
# you will need to install the p12 cert into your browser (google it)
# this uses the client-0 created earlier to for the p12 cert
create p12 client-0

# p12 cert specifying the ca_cn
create p12 client-0 mesh.nirv.ai


### CLI CERT
# create 1 cli cert and save as cli-0{-key}.{pem,csr}
create cli
# create arbitrary amount of cli certs
create cli 7

# create arbitrary amount of cli certs specifying options
# total ca_cn ca_name cert_name config_name
create cli 77 mesh.nirv.ai ca cli cli.cfssl.json


### CERT and CSR info
# get info on cert file
info cert ca
info cert cli-0
info cert client-0
info cert server-0

# get info on a csr file
info csr ca
info csr cli-0
info csr client-0
info csr server-0
```

#### next steps

- Congrats! you have a private CA and the ability to create server, client and CLI certs to encrypt communication between your services
- if you're using these TLS certs with NOMAD/CONSUL
  - [see the env docs for how to set up /etc/ssl/certs](../env/README.md)
- We have a little secret:

> _you can bootstrap your entire stack with this copypasta_

##### copypasta: cfssl initialization

```sh
export CA_CN=mesh.nirv.ai
script.ssl.sh create rootca
# 2 consul servers
script.ssl.sh create server 2
# 3 client applications
script.ssl.sh create client 3
# 1 operator
script.ssl.sh create cli
# 1 p12 cert for the browser
script.ssl.sh create p12 client-0

export CA_CN=mad.nirv.ai
script.ssl.sh create rootca
# 2 nomad server agents
script.ssl.sh create server 2
# 3 nomad client agents
script.ssl.sh create client 3
# 1 operator
script.ssl.sh create cli 1
# 1 p12 cert for the browser
script.ssl.sh create p12 client-0
```

##### copypasta: cfssl validation

```sh
# you can repeat for server, client and cli certs as well
export CA_CN=mad.nirv.ai
script.ssl.sh info cert ca

export CA_CN=mesh.nirv.ai
script.ssl.sh info cert ca
```
