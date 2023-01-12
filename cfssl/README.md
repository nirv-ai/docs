# NIRVai CFSSL

- documentation for cfssl
- [scripting architecture & guidance](.scripts/README.md)
- [source code](https://github.com/nirv-ai/scripts/blob/develop/cloudflare/script.ssl.sh)
- [configuration](https://github.com/nirv-ai/configs/tree/develop/cfssl)
- [based heavily on this doc by hashicorp](https://developer.hashicorp.com/nomad/tutorials/transport-security/security-enable-tls)

## RACEXP

- [NIRVai security project board](https://github.com/orgs/nirv-ai/projects/6/views/1?filterQuery=repo%3A%22nirv-ai%2Fsecurity%22)
- [RACEXP docs](https://github.com/noahehall/theBookOfNoah/blob/master/0current/architectural%20thinking/0racexp.md)

## WHY CFSSL ?

- there are quite few respectable alternatives like [ejcba](https://www.ejbca.org/),[openvpns easy-rsa](https://github.com/OpenVPN/easy-rsa), [smallsteps step-ca](https://github.com/smallstep/certificates) - even vault has a [pki secrets engine](https://developer.hashicorp.com/vault/docs/secrets/pki)
- we chose [cfssl](https://blog.cloudflare.com/introducing-cfssl/) because we're application developers that live on the commandline
  - must be easy to install and easy to automate
  - must be backed by people smarter than us
  - must be completely configurable with 0 opinions
  - must taste good with steak, i mean integrate with our stack

## Setting up CFSSL

### REQUIREMENTS

```sh
# cfssl: option 1 install from source @see https://github.com/cloudflare/cfssl
# cfssl: option 2 install via apt-get @see https://packages.ubuntu.com/search?keywords=golang-cfssl
# jq: @see https://stedolan.github.io/jq/manual/

# directory structure matches:
├── scripts # git clone git@github.com:nirv-ai/scripts.git
├── configs # git clone git@github.com:nirv-ai/configs.git
│   └── cfssl
│   │   ├── cfssl.json # default cfssl configuration
│   │   └── arbitrary.domain.name # init files for this CA
│   │   │   ├── csr.root.ca.json # root ca configuration named ca
│   │   │   ├── csr.cli.cli.json # leaf cert configuration for cli named cli
│   │   │   ├── csr.client.client.json # leaf cert configuration for client named client
│   │   │   ├── csr.server.server.json # leaf cert configuration for server named server
├── secrets # chroot jail, a temporary folder or private git repo
│   └── arbitrary.domain.com
│   │   └── tls # we will persist all files to this directory

```

# USAGE

```sh

### ROOT CA
# create root ca and save as ca[-key].{pem,csr}
export CA_NAME=mesh.nirv.ai
export CA_PEM_NAME=ca
create rootca

# create root ca for a config and save as ca[-key].{pem,csr}
create rootca a.b.c

# create root ca for a config but save as somethingelse[-key].{pem,csr}
create rootca x.y.z somethingelse

### SERVER CERT
# create 1 server cert and save as server-0[-key].{pem,csr}
export CA_NAME=mesh.nirv.ai
export CA_PEM_NAME=ca
export SERVER_NAME=server
create server
# create arbitrary amount of server certs using above vars
create server 7

# create arbitrary amount of server certs specifying options
# ^ server.cfssl.json should be sibling to the server config, NOT next to the default cfssl.json
# ^ this way customizations are kept within the CA configuration dir
create server 77 mesh.nirv.ai ca server server.cfssl.json

### CLIENT CERT
# create 1 client cert and save as client-0[-key].{pem,csr}
export CA_NAME=mesh.nirv.ai
export CA_PEM_NAME=ca
export CLIENT_NAME=client
create client
# create arbitrary amount of client certs using above vars
create client 7

# create arbitrary amount of client certs specifying options
# ^ client.cfssl.json should be sibling to the client config, NOT next to the default cfssl.json
# ^ this way customizations are kept within the CA configuration dir
create client 77 mesh.nirv.ai ca client client.cfssl.json

### CLI CERT
# create 1 cli cert and save as cli-0[-key].{pem,csr}
export CA_NAME=mesh.nirv.ai
export CA_PEM_NAME=ca
export CLI_NAME=cli
create cli
# create arbitrary amount of cli certs using above vars
create cli 7

# create arbitrary amount of cli certs specifying options
# ^ cli.cfssl.json should be sibling to the client config, NOT next to the default cfssl.json
# ^ this way customizations are kept within the CA configuration dir
create cli 77 mesh.nirv.ai ca cli cli.cfssl.json
```
