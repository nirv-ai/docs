# NIRVai VAULT

- documentation for vault
- [scripting architecture & guidance](../scripts/README.md)
- [source code](https://github.com/nirv-ai/scripts/blob/develop/script.vault.sh)
- [configuration](https://github.com/nirv-ai/configs/tree/develop/vault)

## RACEXP

- [NIRV DOCS project board](https://github.com/orgs/nirv-ai/projects/6/views/1?filterQuery=repo%3A%22nirv-ai%2Fdocs%22)
- [RACEXP docs](https://github.com/noahehall/theBookOfNoah/blob/master/0current/architectural%20thinking/0racexp.md)

## WHY VAULT ?

- vault is an opensource database for sensitive data storage, access and control
- our goal is to have:
  - an immutable vault instance: automation from greenfield to prod
  - persistent storage: vault server life cycle should have 0 impact on data persistence

### NIRVai is a [zero trust](https://www.nist.gov/publications/zero-trust-architecture) open source platform

> if you cant commit it - _vault it_

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

- generally this should be the first step _in all microservices_ deployed at NIRVai
- `unless` your application requires access to 0 sensitive data and relies purely on configuration committed to git

### INTERFACE

- TODO: there should be 0 use of vault CLI commands
  - all invocations should go through the http api
- all green field vault instances require human intervention
  - create root pgp key
  - create root token and unseal database
  - create and distribute admin token(s)
  - store root token next to your grenade launchers

```sh

######################### FYI
# setup a chroot jail: @see https://www.howtogeek.com/441534/how-to-use-the-chroot-command-on-linux/

####################### requirements
# curl @see https://curl.se/docs/manpage.html
# jq @see https://stedolan.github.io/jq/manual/

######################### interface
# src dir for the vault server your working in
export VAULT_INSTANCE_DIR=apps/nirvai-core-vault/src
# base vault configs: https://github.com/nirv-ai/configs/tree/develop/vault
REPO_CONFIG_VAULT_PATH=../configs/vault/
# you should mount private configs & overrides at a separate location
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


# if logging in through the UI:
## copypasta the tokens and open the browser to $VAULT_ADDR
## if youve logged in at the same VAULT_ADDR created with a different root token
## you may need to clear your browser storage & cache
## get unseal tokens and use them to unseal db and login to vault
./script.vault.sh get_unseal_tokens

# if logging in via the CLI
## 1. export the appropriate token
## 2. unseal db if required
## 3. verify token configuration

## export root token
export VAULT_TOKEN=$(cat $JAIL/root.unseal.json \
  | jq -r '.root_token' \
  | base64 --decode \
  | gpg -dq \
)

## setting a different token (e.g. admin)
### requires completion of step: `create vault admin & token`
### manually open and verify $JAIL/admin_vault.json is valid json
### ^ logs may exist if created when NIRVAI_SCRIPT_DEBUG was set
## export admin_vault token
USE_VAULT_TOKEN=admin_vault
export VAULT_TOKEN=$(cat $JAIL/$USE_VAULT_TOKEN.json | jq -r '.auth.client_token')

## unseal the DB if its sealed
## (requires password used when creating the pgp key)
./script.vault.sh unseal

## verify your token configuration
./script.vault.sh get token self
```

### new vault server setup

#### create root and admin_vault tokens

```sh
######################### cd /nirv/core
# a human is required to create and distribute
# root & admin vault tokens and unseal keys

# create root and admin gpg keys if they dont exist in jail
gpg --gen-key # repeat for for each entity (root, admin) being assigned a gpg key

# base64 encode the `gpg: key` value of each gpg-key to $JAIL/_NAME_.asc
gpg --export ABCDEFGHIJKLMNOP | base64 > $JAIL/root.asc
gpg --export ABCDEFGHIJKLMNOP | base64 > $JAIL/admin_vault.asc

```

#### wipe, initialize and unseal vault

```sh

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

#### use root token to create admin policy & token

> this is the last time you should ever use the root token

```sh
# set root token & unseal db (@see `# INTERFACE`)

# create policy for vault administrator
ADMIN_POLICY_CONFIG=$VAULT_INSTANCE_DIR/config/000-000-vault-admin-init/policy_admin_vault.hcl
./script.vault.sh create poly $ADMIN_POLICY_CONFIG

# create token for vault administrator
ADMIN_TOKEN_CONFIG=$VAULT_INSTANCE_DIR/config/000-000-vault-admin-init/token_admin_vault.json
./script.vault.sh create token child $ADMIN_TOKEN_CONFIG > $JAIL/admin_vault.json

```

#### use admin token to create policies in policy dir

```sh
# set and verify admin token (@see `# INTERFACE`)

# create all policies in policy dir
POLICY_DIR=$VAULT_INSTANCE_DIR/config/000-001-policy-init
./script.vault.sh process policy_in_dir $POLICY_DIR
```

#### use admin token to create token roles in token role dir

```sh
# set and verify admin token (@see `# INTERFACE`)

# create all token roles in token role dir
TOKEN_ROLE_DIR=$VAULT_INSTANCE_DIR/config/000-002-token-role-init
./script.vault.sh process token_role_in_dir $TOKEN_ROLE_DIR
```

#### use admin token to enable features in feature dir

```sh
# set and verify admin token (@see `# INTERFACE`)

# filename template
## your/feature/dir/enable.THIS_THING.AT_THIS_PATH
## e.g. feature/dir/enable.kv-v2.secret # for versioned sensitive data
## e.g. feature/dir/enable.kv-v1.env # for immutable data with increased perf
## currently we only auto enable features at top-level paths
### wont work: enable.kv-v2.microfrontend.app1.snazzle
### will work: enable.kv-v2.mfe-app1-snazzle

# enable all features in feature dir
FEATURE_DIR=$VAULT_INSTANCE_DIR/config/001-000-enable-feature
./script.vault.sh process enable_feature $FEATURE_DIR

```

#### use admin token to configure auth schemes in auth dir

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

#### use admin token to configure secret engines in engine dir

```sh
# set and verify admin token (@see `# INTERFACE`)

# depending on the type of secret engine your configuring
# the filename template will have different formats:

## kv-v2: secret_kv2.PATH.config.json
### ^ configure the kv2 engine enabled at PATH/

## database config: secret_database.DB_NAME.config.json
### ^ configure the database DB_NAME

## database role config: secret_database.DB_NAME.role.ROLE_NAME.json
### ^ configure ROLE_NAME for database DB_NAME
### NOTE: we only support databases that support rotate-root creds
### the vault root creds will be automatically rotated

SECRET_ENGINE_DIR=$VAULT_INSTANCE_DIR/config/003-000-secret-engine-init
./script.vault.sh process engine_config $SECRET_ENGINE_DIR

```

#### next steps

- Congrats! you have enabled & configured your development environment for vault!
- If you want to do more with vault: [checkout the usage documentation](./usage.md)
- We have a little secret:

> _you can bootstrap your entire stack with this copypasta_

```sh
######################### FYI
# from hashicorp docs: a human is required to create the initial root and admin tokens
## complete section `create root token, initialize and unseal database`

----------
# if you're using the root token beyond this line: I have failed you
----------

######################### REQUIREMENTS
# your on a debian compatible host (ubuntu is free, and regolith is supa dupa fly)
# your directory structure matches:
./you-are-here
├── configs # clone https://github.com/nirv-ai/configs
├── ${CORE_SERVICE_DIR_NAME} # clone https://github.com/nirv-ai/core-service-template
│   ├── script.vault.sh # https://github.com/nirv-ai/scripts/blob/develop/script.vault.sh
│   ├── script.refresh.compose.sh # https://github.com/nirv-ai/scripts/blob/develop/script.refresh.compose.sh
│   ├── script.reset.compose.sh # https://github.com/nirv-ai/scripts/blob/develop/script.reset.compose.sh
├── scripts # clone https://github.com/nirv-ai/scripts
├── secrets # chroot jail, a temporary folder or private git repo
│   ├── dev
│   │   ├── apps
│   │   │   └── vault # following pgp keys must be manually created
│   │   │       ├── admin_vault.asc
│   │   │       ├── root.asc

######################### # COPYPASTA START

# INPUTS: edit these to match the core-service you're developing
CORE_SERVICE_DIR_NAME=core
PREFIX=nirvai
VAULT_INSTANCE_SRC_DIR=apps/${PREFIX}-core-vault/src
VAULT_DOMAIN_AND_PORT=dev.nirv.ai:8300
USE_VAULT_TOKEN=admin_vault
REPO_CONFIG_VAULT_PATH=../configs/vault/

ADMIN_POLICY_CONFIG=$VAULT_INSTANCE_SRC_DIR/config/000-000-vault-admin-init/policy_admin_vault.hcl
ADMIN_TOKEN_CONFIG=$VAULT_INSTANCE_SRC_DIR/config/000-000-vault-admin-init/token_admin_vault.json
POLICY_DIR=$VAULT_INSTANCE_SRC_DIR/config/000-001-policy-init
TOKEN_ROLE_DIR=$VAULT_INSTANCE_SRC_DIR/config/000-002-token-role-init
FEATURE_DIR=$VAULT_INSTANCE_SRC_DIR/config/001-000-enable-feature
AUTH_SCHEME_DIR=$VAULT_INSTANCE_SRC_DIR/config/002-000-auth-init
SECRET_ENGINE_DIR=$VAULT_INSTANCE_SRC_DIR/config/003-000-secret-engine-init


export JAIL="$(pwd)/secrets/dev/apps/vault"
export VAULT_ADDR="https://${VAULT_DOMAIN_AND_PORT}"
export NIRV_SCRIPT_DEBUG=0


cd $CORE_SERVICE_DIR_NAME
docker compose down


sudo rm -rf $VAULT_INSTANCE_SRC_DIR/data/*
rsync -a --delete $REPO_CONFIG_VAULT_PATH $VAULT_INSTANCE_SRC_DIR/config
./script.reset.compose.sh


export VAULT_TOKEN='initilize vault with root pgp key'
./script.vault.sh init


export VAULT_TOKEN=$(cat $JAIL/root.unseal.json \
  | jq -r '.root_token' \
  | base64 --decode \
  | gpg -dq \
)
./script.vault.sh unseal
./script.vault.sh create poly $ADMIN_POLICY_CONFIG
./script.vault.sh create token child $ADMIN_TOKEN_CONFIG > $JAIL/admin_vault.json


export VAULT_TOKEN="$(cat $JAIL/$USE_VAULT_TOKEN.json | jq -r '.auth.client_token')"
./script.vault.sh process policy_in_dir $POLICY_DIR
./script.vault.sh process token_role_in_dir $TOKEN_ROLE_DIR
./script.vault.sh process enable_feature $FEATURE_DIR
./script.vault.sh process auth_in_dir $AUTH_SCHEME_DIR
./script.vault.sh process engine_config $SECRET_ENGINE_DIR
```


https://user-images.githubusercontent.com/10324554/209608898-83135d43-35ff-4e7b-b72c-7ef5fcf8360a.mp4


