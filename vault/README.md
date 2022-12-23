# nirvai docs

- documentation for VAULT @ nirvai

## RACEXP

- [NIRV DOCS project board](https://github.com/orgs/nirv-ai/projects/6/views/1?filterQuery=repo%3A%22nirv-ai%2Fdocs%22)
- [RACEXP docs](https://github.com/noahehall/theBookOfNoah/blob/master/0current/architectural%20thinking/0racexp.md)

## VAULT

- vault is an opensource database for sensitive data, and an API for access management
- it can be used wherever authentication and authorization is required

### the philosophy at NIRVai

> if you cant commit it, vault it

### Why Vault

- NIRVai is a [zero trust](https://www.nist.gov/publications/zero-trust-architecture) open source platform
- our architecture patterns will be a bit more extreme than others

### if your application has network access

- remove all your `.env` and other configuration files from your `.gitignore`
- move all your sensitive data into vault
- refactor your application to be resiliant despite unmet requirements (where are my creds?)
- ensure your application uses a modern TLS version for all communication
- ensure your application encrypts data in transit
  - if persisting to disk, dont, fkn vault it
- commit your remaining data & configuration to git (which should contain no PII/etc)

### if your application is `air-gapped`

- ensure your application runs from a [chroot](https://www.howtogeek.com/441534/how-to-use-the-chroot-command-on-linux/)
- continue with [12-factor](https://12factor.net/)

### How do I know if this data is sensitive?

- it is

## Scripting with vault

- actively used for interacting with a secured vault server behind a secured proxy
- [scripting architecture & guidance](../scripts/README.md)

### links

- [curl](https://curl.se/docs/manpage.html)
- [jq](https://stedolan.github.io/jq/manual/)

```sh
####################### requirements
# curl
# jq


####################### FYI
# append `--output-policy` to see the policy needed to execute a cmd


####################### basic workflow


####################### interface
VAULT_ADDR=$VAULT_ADDR
VAULT_TOKEN=$VAULT_TOKEN


####################### usage
./script.vault.sh poop poop poop

# enable a secret engine e.g. kv-v2
enable secret secretEngineType

# enable approle engine e.g. approle
enable approle approleType

# list all approles
list approles

# list enabled secrets engines
list secret-engines

# list provisioned keys for a postgres role
list postgres leases dbRoleName

# create a secret-id for roleName
create approle-secret roleName

# upsert approle appRoleName with a list of attached policies
create approle appRoleName pol1,polX

# create kv2 secret(s) at secretPath
# dont prepend `secret/` to secretPath
# e.g. create secret kv2 poo/in/ur/eye '{"a": "b", "c": "d"}'
create secret kv2 secretPath jsonString

# get dynamic postgres creds for database role dbRoleName
get postgres creds dbRoleName

# get the secret (kv-v2) at the given path, e.g. foo
# dont prepend `secret/` to path
get secret secretPath

# get the status (sys/healthb) of the vault server
get status

# get vault credentials for an approle
get creds roleId secretId

# get all properties associated with an approle
get approle info appRoleName

# get the approle role_id for roleName
get approle id appRoleName

# get the openapi spec for some path
help some/path/

```
