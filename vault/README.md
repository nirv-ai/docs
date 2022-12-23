# nirvai VAULT

- documentation for VAULT @ nirvai

## RACEXP

- [NIRV DOCS project board](https://github.com/orgs/nirv-ai/projects/6/views/1?filterQuery=repo%3A%22nirv-ai%2Fdocs%22)
- [RACEXP docs](https://github.com/noahehall/theBookOfNoah/blob/master/0current/architectural%20thinking/0racexp.md)

## WHY VAULT ?

- vault is an opensource database for sensitive data storage, access and control

### NIRVai is a [zero trust](https://www.nist.gov/publications/zero-trust-architecture) open source platform

> if you cant commit it, vault it

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

## Setting up vault

### greenfield: create root token and unseal database

- all green field vault instances requires human intervention
  - create root pgp key
  - unseal database and download root token
  - create admin token and never use the root token again

```sh
# create a new gpg key for initializing the vault database
gpg --gen-key
# base64 encode the `gpg: key` value
gpg --export ABCDEFGHIJKLMNOP | base64 > root.asc

# start the vault container with a green state
# first: remove the {raft,vault}.db if they exist in the core/app/vault dir
sudo rm -rf apps/nirvai-core-vault/src/data/*
# next: start the vault server
./script.reset.compose.sh core_vault

# unseal the database: open https://dev.nirv.ai:8200/
## select create a new raft cluster: atleast 5 key shares, at least 3 key threshold
## enter the root.asc file as text into both pgp key fields and download the keys
cat root.asc
# dowload your bank keys, it should match the following
{
  "keys": ["SOME_KEY1", "SOME_KEY2", "SOME_KEY3", ...],
  "keys_base64": ["SOME_KEY1", "SOME_KEY2", "SOME_KEY3", ...],
  "root_token": "ROOT_TOKEN_GUARD_WITH_YOUR_LIFE"
}

# decode the keys
# unseal token: decode each of the keys_base64 in the json provided by vault
echo keys_base64[] | base64 --decode | gpg -dq
# root token: decode root_token in the json provded by vault
echo root_token | base64 --decode | gpg -dq


```

## scripts

- [scripting architecture & guidance](../scripts/README.md)

### script.vault.sh

- actively used for interacting with a secured vault server behind a secured proxy
- [source code](https://github.com/nirv-ai/scripts/blob/develop/script.vault.sh)

```sh

####################### requirements
# curl @see https://curl.se/docs/manpage.html
# jq @see https://stedolan.github.io/jq/manual/

####################### interface
VAULT_ADDR=$VAULT_ADDR
VAULT_TOKEN=$VAULT_TOKEN

####################### configuring postgres dynamic creds
####################### creating new database engine
- disable verify connection
- create connection to postgres



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
