# NIRVai VAULT

- documentation for VAULT @ nirvai
- [scripting architecture & guidance](../scripts/README.md)
- [source code](https://github.com/nirv-ai/scripts/blob/develop/script.vault.sh)

## RACEXP

- [NIRV DOCS project board](https://github.com/orgs/nirv-ai/projects/6/views/1?filterQuery=repo%3A%22nirv-ai%2Fdocs%22)
- [RACEXP docs](https://github.com/noahehall/theBookOfNoah/blob/master/0current/architectural%20thinking/0racexp.md)

## WHY VAULT ?

- vault is an opensource database for sensitive data storage, access and control
- our goal is to have:
  - an immutable vault instance: automation from greenfield to prod
  - persistent storage: vault server life cycle should have 0 impact on data persistence

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

### INTERFACE

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

####################### requirements
# curl @see https://curl.se/docs/manpage.html
# jq @see https://stedolan.github.io/jq/manual/

######################### interface
# base config directory for the vault server instance you are bootstrapping
# base vault configs: https://github.com/nirv-ai/configs/tree/develop/vault
# you should mount private configs & overrides at a separate location
export VAULT_INSTANCE_DIR=apps/nirvai-core-vault/src
REPO_CONFIG_VAULT_PATH=../configs/vault/
rsync -a --delete $REPO_CONFIG_VAULT_PATH $VAULT_INSTANCE_DIR/config

# wherever you will temporarily store created secrets on disk
export JAIL="../secrets/dev/apps/vault"

# address where you expect the vault server to be running
export VAULT_ADDR=https://dev.nirv.ai:8300

# run `unset NIRV_SCRIPT_DEBUG && history -c` when debugging complete
export NIRV_SCRIPT_DEBUG=1

# the below steps require you to complete:
## section: `create root token, initialize and unseal database`
## section: `use root token to create admin policy & token`


# if logging in through the UI: copypasta the tokens and open the browser to $VAULT_ADDR
## if youve logged in at the same VAULT_ADDR created with a different root token
## you may need to clear your browser storage & cache
# get unseal tokens and use them to unseal db and login to vault
./script.vault.sh get_unseal_tokens

# if logging in via the CLI
## 1. export the appropriate token
## 2. unseal db if required
## 3. verify token configuration

## setting root token
export VAULT_TOKEN=$(cat $JAIL/root.unseal.json \
  | jq -r '.root_token' \
  | base64 --decode \
  | gpg -dq \
)

## setting a different token (e.g. admin)
### requires completion of step: `create vault admin & token`
### manually open and verify $JAIL/admin_vault.json is valid json
export VAULT_TOKEN=$(cat $JAIL/admin_vault.json | jq -r '.auth.client_token')

## unseal the DB if its sealed (requires password used when creating the pgp key)
./script.vault.sh unseal

## verify your token configuration
./script.vault.sh get token self
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

# finally: reset the vault server
./script.reset.compose.sh core_vault

######################### initial and unseal vault
# confirm `Vault *IS NOT* initialized`
vault operator init -status

# inititialize vault & distribute each unseal_keys_b64 to the appropriate people
## bypass token requirement, wont work if your token is named poop
export VAULT_TOKEN='poop'
./script.vault.sh init

# verify `Vault *IS* initialized`
vault operator init -status

```

### greenfield: use root token to create admin policy & token

- this is the last time you should ever use the root token

```sh
# set root token & unseal db (@see `# INTERFACE`)

# create policy for vault administrator
ADMIN_POLICY_CONFIG=$VAULT_INSTANCE_DIR/config/000-000-vault-admin-init/policy_admin_vault.hcl
./script.vault.sh create poly $ADMIN_POLICY_CONFIG


# create token for vault administrator
ADMIN_TOKEN_CONFIG=$VAULT_INSTANCE_DIR/config/000-000-vault-admin-init/token_admin_vault.json
./script.vault.sh create token child $ADMIN_TOKEN_CONFIG > $JAIL/admin_vault.json

```

### greenfield: use admin token to create policies in policy dir

```sh
# set and verify admin token (@see `# INTERFACE`)

# create all policies in policy dir
POLICY_DIR=$VAULT_INSTANCE_DIR/config/000-001-policy-init
./script.vault.sh process policy_in_dir $POLICY_DIR
```

### greenfield: use admin token to create tokes in token dir

```sh
# set and verify admin token (@see `# INTERFACE`)

# create all policies in policy dir
TOKEN_DIR=$VAULT_INSTANCE_DIR/config/000-002-token-init
./script.vault.sh process token_in_dir $TOKEN_DIR
```

### greenfield: use admin token to enable features in feature dir

```sh
# set and verify admin token (@see `# INTERFACE`)

# create a directory containing empty files, filename syntax:
## your/feature/dir/enable.THIS_THING.AT_THIS_PATH
## e.g. feature/dir/enable.kv-v2.secret
## e.g. feature/dir/enable.kv-v2.frontend
### will enable secret engine kv-v2 at paths secret/ && frontend/
### currently we only auto enable features at top-level paths
### bad: enable.kv-v2.microfrontend.app1.snazzle

# TODO: add kv1, for most cases you'll want kv1
# unless you need versioned secrets
# enable all features in feature dir
FEATURE_DIR=$VAULT_INSTANCE_DIR/config/001-001-enable-features
./script.vault.sh process enable_feature $FEATURE_DIR

```

### greenfield: use admin token to configure auth schemes in auth dir

```sh
# set and verify admin token (@see `# INTERFACE`)

# create a directory containing json config files, filename syntax:
## your/auth/dir/auth_approle_role_ROLE_NAME.json
## e.g. auth/dir/auth_approle_role_backend.json
## e.g. auth/dir/auth_approle_role_cache.json
### will create/update approle(s) using the configuration in each file

AUTH_SCHEME_DIR=$VAULT_INSTANCE_DIR/config/002-000-auth-init
./script.vault.sh process auth_in_dir $AUTH_SCHEME_DIR

```

### greenfield: use admin token to configure secret engines in engine dir

```sh
# set and verify admin token (@see `# INTERFACE`)

# TODO: this entire section should be executable by a single cmd
# SECRET_ENGINE_DIR=$VAULT_INSTANCE_DIR/config/003-000-secret-engine-init
# ./script.vault.sh process secret_in_dir $SECRET_ENGINE_DIR

# TODO: set kv1 path to secret: better perf than v2
# TODO: set kv2 path to vsecret: powerful usecases
### enable kv-v2: UI > secrets > secret
./script.vault.sh enable kv-v2 secret
### configure kv-v2
### todo

### enable database: UI > secrets > database
./script.vault.sh enable database database
### configure database
### todo

#### connect postgres database plugin
##### todo
##### todo

### enable ssh: UI > secrets > ...
# ./script.vault.sh enable ssh ssh
### configure ssh
### todo

### enable nomad: nomad > secrets > ...
# ./script.vault.sh enable nomad nomad
### configure nomad
### todo

### enable terraform-cloud: UI > secrets > ...
# ./script.vault.sh enable tfcloud tfcloud
### configure tfcloud
### todo
```

### greenfield: next steps

- Congrats! you have enabled & configured your development environment for vault!
- If you want to do more with vault, see the main documentation below

## script.vault.sh documentation

```sh
####################### configuring postgres dynamic creds
####################### creating new database engine
- disable verify connection
- create connection to postgres


####################### USAGE
# follow steps in `# INTERFACE` to setup your cli
# then invoke cmds below, e.g:
./script.vault.sh poop poop poop

############ approle
# all approle examples use the following role name
ROLE_NAME=auth_approle_role_bff

## enable approle: UI > access > approle
enable approle approle

## create/update an approle
create approle path/to/distinct_approle_config.json

## verify approle role was created
get approle info $ROLE_NAME

## get role-id for an approle
get approle id $ROLE_NAME

## create secret-id for an approle
create approle-secret $ROLE_NAME

# get approle credentials for approle roleId secretId
get approle-creds xyz-321-yzx-321 123-xyz-123-zyx

## lookup secret-id info for an approle
get approle secret-id $ROLE_NAME 123-xyz-123-zyx

## lookup secret-id-accessor info for an approle
get approle secret-id-axor $ROLE_NAME 123-xyz-123-zyx

## revoke a secret id for an approle
revoke approle-secret-id $ROLE_NAME 123-xyz-123-zyx

## revoke a secret id accessor for an approle
revoke approle-secret-id-axor $ROLE_NAME 123-xyz-123-zyx

## list accessors for for an approle role
list approle-axors $ROLE_NAME

## list all approle roles
list approles

## rm approle role
rm approle-role $ROLE_NAME



####################### PREVIOUS
####################### all of this should be grouped by endpoint
./script.vault.sh poop poop poop

# enable a kv-2 secret engine at path secret/
enable kv-v2 secret

# enable database secret engine at path database/
enable database database

# list enabled secrets engines
list secret-engines

# list provisioned keys for a postgres role
list postgres leases dbRoleName


# create kv2 secret(s) at secretPath
# dont prepend `secret/` to secretPath
# e.g. create secret kv2 poo/in/ur/eye '{"a": "b", "c": "d"}'
create secret kv2 secretPath jsonString

# get dynamic postgres creds for database role dbRoleName
get postgres creds dbRoleName

# get the secret (kv-v2) at the given path, e.g. foo
# dont prepend `secret/` to path
get secret secretPath

# get the status (sys/health) of the vault server
get status

# get the openapi spec for some path
help some/path/

```
