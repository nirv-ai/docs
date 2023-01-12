# NIRVai CFSSL

- [scripting architecture & guidance](.scripts/README.md)
- [source code](https://github.com/nirv-ai/scripts/blob/develop/cloudflare/script.ssl.sh)
- [configuration](https://github.com/nirv-ai/configs/tree/develop/cfssl)
- [based heavily on this doc by hashicorp](https://developer.hashicorp.com/nomad/tutorials/transport-security/security-enable-tls)

## RACEXP

- [NIRVai security project board](https://github.com/orgs/nirv-ai/projects/6/views/1?filterQuery=repo%3A%22nirv-ai%2Fsecurity%22)
- [RACEXP docs](https://github.com/noahehall/theBookOfNoah/blob/master/0current/architectural%20thinking/0racexp.md)

## WHY CFSSL ?

- there are quite few respectable alternatives like [ejcba](https://www.ejbca.org/),[openvpns easy-rsa](https://github.com/OpenVPN/easy-rsa), [smallsteps step-ca](https://github.com/smallstep/certificates) - even vault has a [pki secrets engine](https://developer.hashicorp.com/vault/docs/secrets/pki)
- we chose [cfssl](https://blog.cloudflare.com/introducing-cfssl/) because based on the following requirements
  - straight forward to install and automate
  - backed by people smarter than us
  - completely configurable with 0 opinions

## Setting up CFSSL

### REQUIREMENTS

```sh
# cfssl: option 1 install from source @see https://github.com/cloudflare/cfssl
# cfssl: option 2 install via apt-get @see https://packages.ubuntu.com/search?keywords=golang-cfssl
# jq: @see https://stedolan.github.io/jq/manual/

# directory structure matches:
├── scripts # git clone git@github.com:nirv-ai/scripts.git
├── configs # you can use ours: git clone git@github.com:nirv-ai/configs.git
│   └── cfssl
│   │   ├── cfssl.json # default cfssl configuration
│   │   └── arbitrary.domain.name # init files for this CA
│   │   │   ├── custom.cfssl.json # optional cfssl configuration, the default is used...by default
│   │   │   ├── csr.root.ca.json # root ca configuration named ca
│   │   │   ├── csr.cli.cli.json # leaf cert configuration for cli named cli
│   │   │   ├── csr.client.client.json # leaf cert configuration for client named client
│   │   │   ├── csr.server.server.json # leaf cert configuration for server named server
├── secrets # chroot jail, a temporary folder or private git repo
│   └── arbitrary.domain.com
│   │   └── tls # we will persist all created certs inside this directory

```

### INTERFACE

```sh
# you generally only need to set the CA_CN var when your directory structure matches
export CA_CN=mesh.nirv.ai

# else you can map the these values to the directory structure above
# export CA_PEM_NAME=ca
# export CFSSL_CONFIG_NAME=custom.cfssl.json
# export CLI_NAME=cli
# export CLIENT_NAME=client
# export CONFIG_DIR_NAME=configs
# export SECRET_DIR_NAME=secrets
# export SERVER_NAME=server
# export TLS_DIR_NAME=tls

# you shouldnt (but can) change these as well
# export CFSSL_DIR="${CFSSL_DIR:-$SCRIPTS_DIR_PARENT/$CONFIG_DIR_NAME/cfssl}"
# export JAIL="${JAIL:-$SCRIPTS_DIR_PARENT/$SECRET_DIR_NAME/$CA_CN}"
# export JAIL_TLS="${JAIL_TLS:-$JAIL/$TLS_DIR_NAME}"
# export CA_CERT="${CA_CERT:-$JAIL_TLS/$CA_PEM_NAME}.pem"
# export CA_PRIVKEY="${CA_PRIVKEY:-$JAIL_TLS/$CA_PEM_NAME}-key.pem"
```

### USAGE

```sh

### ROOT CA
# create root ca and save as ca[-key].{pem,csr}
create rootca

# create root ca for a config and save as ca[-key].{pem,csr}
create rootca a.b.c

# create root ca for a config but save as somethingelse[-key].{pem,csr}
create rootca x.y.z somethingelse

### SERVER CERT
# create 1 server cert and save as server-0[-key].{pem,csr}
create server
# create arbitrary amount of server certs using above vars
create server 7

# create arbitrary amount of server certs specifying options
# ^ server.cfssl.json should be sibling to the server config, NOT next to the default cfssl.json
# ^ this way customizations are kept within the CA configuration dir
create server 77 mesh.nirv.ai ca server server.cfssl.json

### CLIENT CERT
# create 1 client cert and save as client-0[-key].{pem,csr}
create client
# create arbitrary amount of client certs
create client 7

# create arbitrary amount of client certs specifying options
# ^ client.cfssl.json should be sibling to the client config, NOT next to the default cfssl.json
# ^ this way customizations are kept within the CA configuration dir
create client 77 mesh.nirv.ai ca client client.cfssl.json

### CLI CERT
# create 1 cli cert and save as cli-0[-key].{pem,csr}
create cli
# create arbitrary amount of cli certs
create cli 7

# create arbitrary amount of cli certs specifying options
# ^ cli.cfssl.json should be sibling to the client config, NOT next to the default cfssl.json
# ^ this way customizations are kept within the CA configuration dir
create cli 77 mesh.nirv.ai ca cli cli.cfssl.json

### CERT and CSR info
# first create a root, client or server cert using something above

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
