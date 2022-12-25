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
- refactor your application to be resiliant despite unmet requirements
- ensure your application uses a modern TLS version for all communication
- ensure your application encrypts data in transit
  - if persisting sensitive data to disk, dont, fkn vault it
- commit your remaining data & configuration to git (which should contain no PII/etc)

### if your application is `air-gapped`

- ensure your application runs from a [chroot](https://www.howtogeek.com/441534/how-to-use-the-chroot-command-on-linux/)
- continue with [12-factor](https://12factor.net/)

### How do I know if this data is sensitive?

- it is

## Setting up vault

- all green field vault instances require human intervention
  - create root pgp key
  - create root token and unseal database
  - create admin token
  - store root token next to your grenade launchers
- auto unseal: all have costs
  - alicloud kms
  - aws kms
  - azure key vault
  - gcp cloud kms
  - oci kms
  - hsm pkcs11
- auto unseal: free
  - vault transit
- preferred automation [todo]: auto unseal with seal transit
- preferred automation: auto unseal with aws kms
  - [cant afford it](https://aws.amazon.com/kms/pricing/) use seal transit instead

```sh

######################### FYI
# setup a chroot jail: @see https://www.howtogeek.com/441534/how-to-use-the-chroot-command-on-linux/

# havent had any success in curling hcl to vault http api, convert to json via this tool
# @see https://www.convertsimple.com/convert-hcl-to-json/
# haha ^ didnt work either
# fear the copypasta: https://gist.github.com/v6/f4683336eb1c4a6a98a0f3cf21e62df2

######################### interface
# wherever you will temporarily store created secrets on disk
export JAIL="../secrets/dev/apps/vault"

# address where you expect the vault server to be running
export VAULT_ADDR=https://dev.nirv.ai:8300

# config directory for the vault server instance you are bootstrapping
export VAULT_INSTANCE_DIR=apps/nirvai-core-vault/src

# run `unset NIRV_SCRIPT_DEBUG && history -c` when debugging complete
export NIRV_SCRIPT_DEBUG=1

## setting root token
## requires completion of step: `create root pgp keys` (see below)
export VAULT_TOKEN=$(cat $JAIL/root.unseal.json \
  | jq -r '.root_token' \
  | base64 --decode \
  | gpg -dq \
)

## setting admin token
### manually open and verify $JAIL/admin_vault.json is valid json
### requires completion of step: `create vault admin & token` (see below)
export VAULT_TOKEN=$(cat $JAIL/admin_vault.json | jq -r '.auth.client_token')

# if exporting tokens: verify configuration
./script.vault.sh get token self

# if logging in through the UI: copypasta the tokens
## if youve logged in at the same VAULT_ADDR but with a different VAULT_INSTANCE
## you will need to clear your browser storage & cache
./script.vault get_unseal_tokens
```

### greenfield: create root token, initialize and unseal database

```sh
######################### cd /nirv/core: create a root gpg key and restart vault instance
# a human is required to create and distribute
# root & admin vault tokens and unseal keys

# create root and admin gpg keys if they dont exist in jail
gpg --gen-key # repeat for for each entity (root, admin) being assigned a gpg key

# base64 encode the `gpg: key` value of each gpg-key to $JAIL/_NAME_.asc
gpg --export ABCDEFGHIJKLMNOP | base64 > $JAIL/root.asc
gpg --export ABCDEFGHIJKLMNOP | base64 > $JAIL/admin_vault.asc

# stop any running containers
docker compose down

# remove previous vault data {e.g. raft,vault}.db
sudo rm -rf $VAULT_INSTANCE_DIR/data/*

# forcefully sync vault dev configs into the vault instance config dir
rsync -a --delete ../configs/vault/ $VAULT_INSTANCE_DIR/config

# finally: reset the vault server
./script.reset.compose.sh core_vault

######################### initial and unseal vault
# confirm `Vault IS NOT initialized`
vault operator init -status

# inititialize vault & distribute each unseal_keys_b64 to the appropriate people
export VAULT_TOKEN='poop' # bypass token requirement, wont work if your token is named poop
./script.vault.sh init

# verify `Vault IS initialized`
vault operator init -status

# unseal vault: will require you to enter password set on pgp key
./script.vault.sh unseal

```

### greenfield: use root token to create vault admin token and policy

```sh
# set and verify root token (see above)

# create policy for vault administrator
./script.vault.sh create poly $VAULT_INSTANCE_DIR/config/000-000-vault-admin-init/policy_admin_vault.hcl

# create token for vault administrator
./script.vault.sh create token child $VAULT_INSTANCE_DIR/config/000-000-vault-admin-init/token_admin_vault.json > $JAIL/admin_vault.json

# set VAULT_TOKEN to the admin token and verify access (see above)

# restart the vault server and verify the admin token can unseal it
./script.refresh.compose.sh core_vault
./script.vault.sh unseal
```

### greenfield: use admin token to create policies

```sh
# ensure vault_token points to the admin_vault token
# (see above)

# forcefully sync vault dev configs into vault app
rsync -a --delete ../configs/vault/ $VAULT_INSTANCE_DIR/config

# create all policies in policy dir
./script.vault.sh process policy_in_dir $VAULT_INSTANCE_DIR/config/001-000-policy-init
```

### greenfield: enable auth schemes

```sh
# verify you can access vault with root token
# ensure vault_token points to the admin_vault token
# (see above)

```

## script.vault.sh documentation

- [scripting architecture & guidance](../scripts/README.md)
- [source code](https://github.com/nirv-ai/scripts/blob/develop/script.vault.sh)

```sh

####################### requirements
# curl @see https://curl.se/docs/manpage.html
# jq @see https://stedolan.github.io/jq/manual/

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
