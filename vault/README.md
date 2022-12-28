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

### REQUIREMENTS

- if your directory structure does not match below
- simplify modify the inputs in `# INTERFACE` section
- [get the scripts from here](https://github.com/nirv-ai/scripts)
- [get the configs from here](https://github.com/nirv-ai/scripts)

```sh

######################### REQUIREMENTS
# curl @see https://curl.se/docs/manpage.html
# jq @see https://stedolan.github.io/jq/manual/

# debian compatible host (only tested on ubuntu)
# directory structure matches:
├ you-are-here
├── configs
├── ${CORE_SERVICE_DIR_NAME}
│   ├── compose.yaml
│   ├── apps/{app1..X}/{...}
│   ├── script.vault.sh
│   ├── script.refresh.compose.sh
│   ├── script.reset.compose.sh
├── secrets # chroot jail, a temporary folder or private git repo
│   ├── dev
│   │   ├── apps
│   │   │   └── vault # following pgp keys must be manually created (see below)
│   │   │       ├── admin_vault.asc
│   │   │       ├── root.asc

```

### INTERFACE

- all green field vault instances require human intervention
  - create root pgp key
  - create root token and unseal database
  - create and distribute admin token(s)
  - store root token next to your grenade launchers

```sh

######################### FYI
# setup a chroot jail: @see https://www.howtogeek.com/441534/how-to-use-the-chroot-command-on-linux/


######################### interface
# monorepo containing all of your applications
# @see https://github.com/nirv-ai/core-service-template
CORE_SERVICE_DIR_NAME=core

# prefix of your monorepo services,
# e.g. /core/apps/PREFIX-nodejs-app/package.json
# e.g. /core/apps/PREFIX-reactjs-frontend/package.json
PREFIX=nirvai

# the name of your monorepo vault server instance
# e.g /core/apps/nirvai-MICRO-SERVICE-NAME/...
VAULT_INSTANCE_DIR_NAME=core-vault

# the path to your monorepo vault server instance src dir
VAULT_INSTANCE_SRC_DIR=apps/$PREFIX-$VAULT_INSTANCE_DIR_NAME/src

# the maps directly to VAULT_ADDR set in your vault configuration
VAULT_DOMAIN_AND_PORT=dev.nirv.ai:8300

# what to call the admin token
USE_VAULT_TOKEN=admin_vault

# where your reusable vault configs are stored
REPO_CONFIG_VAULT_PATH=../configs/vault/

# @see https://github.com/nirv-ai/configs/tree/develop/vault
# path to the vault admin policy
ADMIN_POLICY_CONFIG=$VAULT_INSTANCE_SRC_DIR/config/000-000-vault-admin-init/policy_admin_vault.hcl

# path to the vault admin token config
ADMIN_TOKEN_CONFIG=$VAULT_INSTANCE_SRC_DIR/config/000-000-vault-admin-init/token_admin_vault.json

# directory filled with HCL files
## each HCL file is a policy to create
POLICY_DIR=$VAULT_INSTANCE_SRC_DIR/config/000-001-policy-init

# directory filled with json files
## each file configures a token role to create
TOKEN_ROLE_DIR=$VAULT_INSTANCE_SRC_DIR/config/000-002-token-role-init

# directory filled with empty files
## each filename denotes a feature and the path at which it should be enabled
FEATURE_DIR=$VAULT_INSTANCE_SRC_DIR/config/001-000-enable-feature

# directory filled with json files
## each file configures an authentication scheme
AUTH_SCHEME_DIR=$VAULT_INSTANCE_SRC_DIR/config/002-000-auth-init

# directory filled with json files
## each filename specifies parameters for a secret engine
## each file contains data specific to thing being initialized
SECRET_ENGINE_DIR=$VAULT_INSTANCE_SRC_DIR/config/003-000-secret-engine-init

# directory filled with empty files
## each filename create a token role token for a downstream service
TOKEN_INIT_DIR=$VAULT_INSTANCE_SRC_DIR/config/004-000-token-init

# directory filled with json files
## each filename denotes a secret engine, path enabled, path to store secret
SECRET_DATA_INIT_DIR=$VAULT_INSTANCE_SRC_DIR/config/005-000-secret-data-init

# wherever you will temporarily store created secrets on disk
export JAIL="$(pwd)/secrets/dev/apps/vault"

# the vault addr with protocol specified, always use HTTPS, even in dev
export VAULT_ADDR="https://${VAULT_DOMAIN_AND_PORT}"

# set to 1 to turn on debugging and log statements made via script.vault.sh
# run `unset NIRV_SCRIPT_DEBUG && history -c` to delete history after debugging
## this will create invalid json files if set 1, as log statements will be in the file
## this bug is a feature, so you dont forget to turn it off
export NIRV_SCRIPT_DEBUG=0

# every CMD after this line expects you to be in the root of your monorepo
cd $CORE_SERVICE_DIR_NAME

# resync vault configs into your instance dir
rsync -a --delete $REPO_CONFIG_VAULT_PATH $VAULT_INSTANCE_SRC_DIR/config


```

### LOG INTO THE UI

#### login through the UI

```sh

# the below steps require you to _atleast_ complete:
## section: `create root and admin_vault tokens`
## section: `use root token to create admin policy & token`

## I recommended you come back
## after you've reached the end of this file instead

# if logging in through the UI:
## copypasta the tokens and open the browser to $VAULT_ADDR
## if you've logged in at the same VAULT_ADDR created with a different root token
## you may need to clear your browser storage & cache


############################################
################# DANGER ###################
# this logs all your unseal tokens & vault token as plain text in your shell
# this logs the minimum amount of unseal tokens required to unseal vault
./script.vault.sh get_unseal_tokens
# or you can retrieve a single unseal token only (wont log your vault token)
./script.vault.sh get_single_unseal_token 0 # or 1 for the second, or 2 for third, etc.
################# DANGER ###################
############################################
```

### NEW VAULT SERVER SETUP

#### create root and admin_vault pgp keys

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

#### wipe and initialize vault server

```sh
# stop any running containers
docker compose down

# remove previous vault data {e.g. raft,vault}.db
sudo rm -rf $VAULT_INSTANCE_SRC_DIR/data/*

# finally: reset the vault server
./script.reset.compose.sh core_vault

######################### initial and unseal vault
export VAULT_TOKEN='initialzing vault'

# confirm `Vault *IS NOT* initialized`
./script.vault.sh get status

# inititialize vault & distribute each unseal_keys_b64 to the appropriate people
./script.vault.sh init

# verify `Vault *IS* initialized`
./script.vault.sh get status

```

#### set root token and unseal db

```sh
## export root token
export VAULT_TOKEN=$(cat $JAIL/root.unseal.json \
  | jq -r '.root_token' \
  | base64 --decode \
  | gpg -dq \
)

## (requires password used when creating the pgp key)
./script.vault.sh unseal
./script.vault.sh get status
./script.vault.sh get token self
```

#### use root token to create admin policy & token

> this is the last time you should ever use the root token

```sh
# create policy then token for vault administrator
./script.vault.sh create poly $ADMIN_POLICY_CONFIG
./script.vault.sh create token child $ADMIN_TOKEN_CONFIG > $JAIL/admin_vault.json
```

#### set admin token and unseal db

```sh

## using admin (or any previously created) token to authenticate to vault
USE_VAULT_TOKEN=admin_vault
export VAULT_TOKEN=$(cat $JAIL/$USE_VAULT_TOKEN.json | jq -r '.auth.client_token')

## unseal the DB if its sealed and verify vault server status & token info
## (requires password used when creating the pgp key)
./script.vault.sh unseal
./script.vault.sh get status
./script.vault.sh get token self

```

#### use admin token to create policies in policy dir

```sh
# set and verify admin token (@see `# set admin token and unseal db`)

./script.vault.sh process policy_in_dir $POLICY_DIR
```

#### use admin token to create token roles in token role dir

```sh
# set and verify admin token (@see `# set admin token and unseal db`)

./script.vault.sh process token_role_in_dir $TOKEN_ROLE_DIR
```

#### use admin token to enable features in feature dir

```sh
# set and verify admin token (@see `# set admin token and unseal db`)

# filename template
## your/feature/dir/enable.THIS_THING.AT_THIS_PATH
## e.g. feature/dir/enable.kv-v2.secret # for versioned sensitive data
## e.g. feature/dir/enable.kv-v1.env # for immutable data with increased perf
## currently we only auto enable features at top-level paths
### wont work: enable.kv-v2.microfrontend.app1.snazzle
### will work: enable.kv-v2.mfe-app1-snazzle

# enable all features in feature dir
./script.vault.sh process enable_feature $FEATURE_DIR

```

#### use admin token to configure auth schemes in auth dir

```sh
# set and verify admin token (@see `# set admin token and unseal db`)

# create a directory containing json config files, filename syntax:
## your/auth/dir/auth_approle_role_ROLE_NAME.json
## e.g. auth/dir/auth_approle_role_backend.json
## e.g. auth/dir/auth_approle_role_cache.json
### will create/update approle(s) using the configuration in each file

./script.vault.sh process auth_in_dir $AUTH_SCHEME_DIR

```

#### use admin token to configure secret engines in engine dir

```sh
# set and verify admin token (@see `# set admin token and unseal db`)

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

# this requires a postgres DB to be running, see configs
./script.vault.sh process engine_config $SECRET_ENGINE_DIR

```

#### use admin token to create initial auth tokens for downstream services

```sh
# set and verify admin token (@see `# set admin token and unseal db`)

# depending on the type of authentication scheme
# the filename template will have different formats:

## app role template: token_create_approle.ROLE_NAME.FILE_NAME
## e.g. token_create_approle.auth_approle_role_bff.bff
## ^ save role-id and secret-id as $JAIL/auth_approle_role_bff.bff.json

## token role template: token_create_token_role.ROLE_NAME.FILE_NAME
## e.g. token_create_token_role.batch_infra.cd
## ^ save batch token for role batch_infra as $JAIL/batch_infra.cd.json
#### by default batch tokens expire after 20 minutes: good for CD
## e.g. token_create_token_role.periodic_infra.ci
## ^ save periodic token for role periodic_infra as $JAIL/periodic_infra.ci.json
#### by default periodic must renew within 30 days: good for CI

./script.vault.sh process token_in_dir $TOKEN_INIT_DIR
```

#### use admin token to hydrate initial secret data for downstream services

```sh
# set and verify admin token (@see `# set admin token and unseal db`)

# depending on the type of secret engine
# the filename template will have different formats:

## KV V1 secret engine: hydrate_kv1.ENABLED_AT_THIS_PATH.STORE_SECRET_AT_THIS_PATH
## e.g. secret_kv1.env.auth_approle_role_bff.json
## ^ upsert data at secret-path auth_approle_role_bff in the kv1 engine enabled at env

## KV V2 secret engine: hydrate_kv2.ENABLED_AT_THIS_PATH.STORE_SECRET_AT_THIS_PATH
## e.g. secret_kv2.secret.auth_approle_role_bff.json
## ^ upsert data at secret-path auth_approle_role_bff in the kv2 engine enabled at secret
./script.vault.sh process secret_data_in_dir $SECRET_DATA_INIT_DIR
```

#### next steps

- Congrats! you have a zero trust development environment backed by VAULT!
- If you want to do more with vault: [checkout the usage documentation](./usage.md)
- We have a little secret:

> _you can bootstrap your entire stack with this copypasta_

```sh
######################### FYI
# from hashicorp docs: a human is required to create the initial root and admin tokens
## complete section `create root and admin_vault tokens`
## before continuiing with copypasta


########################## COPYPASTA START

# INPUTS: edit to match the your directory structure
CORE_SERVICE_DIR_NAME=core
PREFIX=nirvai
VAULT_INSTANCE_DIR_NAME=core-vault
VAULT_INSTANCE_SRC_DIR=apps/$PREFIX-$VAULT_INSTANCE_DIR_NAME/src
VAULT_DOMAIN_AND_PORT=dev.nirv.ai:8300
USE_VAULT_TOKEN=admin_vault
REPO_CONFIG_VAULT_PATH=../configs/vault/

# CONFIGS: edit to modify all supported vault features
ADMIN_POLICY_CONFIG=$VAULT_INSTANCE_SRC_DIR/config/000-000-vault-admin-init/policy_admin_vault.hcl
ADMIN_TOKEN_CONFIG=$VAULT_INSTANCE_SRC_DIR/config/000-000-vault-admin-init/token_admin_vault.json
POLICY_DIR=$VAULT_INSTANCE_SRC_DIR/config/000-001-policy-init
TOKEN_ROLE_DIR=$VAULT_INSTANCE_SRC_DIR/config/000-002-token-role-init
FEATURE_DIR=$VAULT_INSTANCE_SRC_DIR/config/001-000-enable-feature
AUTH_SCHEME_DIR=$VAULT_INSTANCE_SRC_DIR/config/002-000-auth-init
SECRET_ENGINE_DIR=$VAULT_INSTANCE_SRC_DIR/config/003-000-secret-engine-init
TOKEN_INIT_DIR=$VAULT_INSTANCE_SRC_DIR/config/004-000-token-init
SECRET_DATA_INIT_DIR=$VAULT_INSTANCE_SRC_DIR/config/005-000-secret-data-init

export JAIL="$(pwd)/secrets/dev/apps/vault"
export VAULT_ADDR="https://${VAULT_DOMAIN_AND_PORT}"
export NIRV_SCRIPT_DEBUG=0


cd $CORE_SERVICE_DIR_NAME
docker compose down


# default: wipe all containers, images and volumes
# then recreate everything as if its the first time
sudo rm -rf $VAULT_INSTANCE_SRC_DIR/data/*
rsync -a --delete $REPO_CONFIG_VAULT_PATH $VAULT_INSTANCE_SRC_DIR/config
# this will start $CORE_SERVICE_DIR_NAME/compose.yaml
./script.reset.compose.sh

# alternative 1: only wipe specific services
## sudo rm -rf $VAULT_INSTANCE_SRC_DIR/data/*
## rsync -a --delete $REPO_CONFIG_VAULT_PATH $VAULT_INSTANCE_SRC_DIR/config
## ./script.reset.compose.sh core_proxy 1
## ./script.reset.compose.sh core_postgres 1
## ./script.reset.compose.sh core_vault 1

# alternative 2: simple restart of specific service(s)
## ./script.refresh.compose.sh core_proxy
## ./script.refresh.compose.sh core_postgres
## ./script.refresh.compose.sh core_vault

export VAULT_TOKEN='initilize vault with root pgp key'
./script.vault.sh init


# skip this section if admin token has already been created
export VAULT_TOKEN=$(cat $JAIL/root.unseal.json \
  | jq -r '.root_token' \
  | base64 --decode \
  | gpg -dq \
)
./script.vault.sh unseal
./script.vault.sh create poly $ADMIN_POLICY_CONFIG
./script.vault.sh create token child $ADMIN_TOKEN_CONFIG > $JAIL/admin_vault.json


export VAULT_TOKEN="$(cat $JAIL/$USE_VAULT_TOKEN.json | jq -r '.auth.client_token')"
./script.vault.sh unseal
./script.vault.sh process policy_in_dir $POLICY_DIR
./script.vault.sh process token_role_in_dir $TOKEN_ROLE_DIR
./script.vault.sh process enable_feature $FEATURE_DIR
./script.vault.sh process auth_in_dir $AUTH_SCHEME_DIR
./script.vault.sh process engine_config $SECRET_ENGINE_DIR
./script.vault.sh process token_in_dir $TOKEN_INIT_DIR
./script.vault.sh process secret_data_in_dir $SECRET_DATA_INIT_DIR
########################## COPYPASTA END

########################## VALIDATION
# via UI: get tokens then open browser to $VAULT_DOMAIN_AND_PORT
# ./script.vault.sh get_unseal_tokens

# via cli
## validate vault & admin token
./script.vault.sh get status
./script.vault.sh get token self
./script.vault.sh get_unseal_tokens
## validate dynamic db creds
./script.vault.sh get postgres creds readonly
./script.vault.sh get postgres creds readwrite
./script.vault.sh get postgres creds readstatic

```
