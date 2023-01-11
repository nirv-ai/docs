# NIRVai CFSSL

- documentation for cfssl
- [scripting architecture & guidance](.scripts/README.md)
- [source code](https://github.com/nirv-ai/scripts/blob/develop/cloudflare/script.ssl.sh)
- [configuration](https://github.com/nirv-ai/configs/tree/develop/cfssl)

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
