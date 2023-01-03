# NIRVai VAULT

- documentation for vault
- [scripting architecture & guidance](.scripts/README.md)
- [source code](https://github.com/nirv-ai/scripts/blob/develop/vault)
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
- nirvai maintains a `core` vault application which you can integrate with by adding a vault app to your monorepo
  - all directions that follow assume this is your intent
  - scope all of your things: every service integrates with the same upstream vault server

### REQUIREMENTS

- if your directory structure does not match below
- modify the inputs in the `# INTERFACE` section that follows
- [get the scripts from here](https://github.com/nirv-ai/scripts)
  - [add them to your path](../scripts/README.md)
- [get the configs from here](https://github.com/nirv-ai/configs/vault)

```sh

######################### REQUIREMENTS
# curl @see https://curl.se/docs/manpage.html
# jq @see https://stedolan.github.io/jq/manual/
# debian compatible host (only tested on ubuntu)

# directory structure matches:
├ you-are-here
├── configs
│   └── vault
│   │   └── core # init files for upstream vualt server
│   │   └── ${REPO_DIR} # init files for this monorepo's vault instance
│   │   │   ├── # ^ directories above may contain any of: (in initialization order)
│   │   │   ├── vault-admin/*
│   │   │   ├── policy/*
│   │   │   ├── token-role/*
│   │   │   ├── enable-feature/*
│   │   │   ├── auth/*
│   │   │   ├── secret-engine/*
│   │   │   ├── token/*
│   │   │   ├── secret-data/*
├── ${REPO_DIR}
│   ├── compose.yaml
│   ├── apps/{appX..Y}/...
├── secrets # chroot jail, a temporary folder or private git repo
│   └── dev
│   │   └── apps
│   │   │   └── vault # pgp.asc files must be manually created (see below)
│   │   │   │   └── token
│   │   │   │   │   ├── root/* # directory for root and unseal tokens
│   │   │   │   │   ├── admin/* # directory for admin token


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
BASE_DIR=`pwd`

# the directory name of your mono repo
# e.g. /git/web/**/*
REPO_DIR=$BASE_DIR/web

# the directory containing your monorepo apps, as apposed to packages
# e.g. /git/repo-dir/apps/{app1, app2}/package.json
REPO_APPS_DIR=$REPO_DIR/apps

# APP_PREFIX of your monorepo services,
# e.g. /web/apps/nirvai-appname/package.json
APP_PREFIX=nirvai

# the name of your monorepo vault server instance
# e.g /core/apps/prefix-web-vault/...
VAULT_INSTANCE_DIR_NAME=web-vault

# service name of your app in your compose.yaml
VAULT_SERVICE_NAME=web_vault

# the path to your monorepo vault instance src dir
export VAULT_INSTANCE_SRC_DIR=$REPO_APPS_DIR/$APP_PREFIX-$VAULT_INSTANCE_DIR_NAME/src

# vault instance will use these configs as starter templates
VAULT_BASE_CONFIG_DIR=$BASE_DIR/configs/vault/

# this maps directly to VAULT_ADDR set in your vault configuration
VAULT_DOMAIN_AND_PORT=dev.nirv.ai:8300

export JAIL="$BASE_DIR/secrets/dev/apps/vault"

# the vault addr with protocol specified, always use HTTPS, even in dev
export VAULT_ADDR="https://${VAULT_DOMAIN_AND_PORT}"

# set to 1 to turn on debugging and log statements made via script.vault.sh
# run `unset NIRV_SCRIPT_DEBUG && history -c` to delete history after debugging
## this will create invalid json files if set 1, as log statements will be in the file
## this bug is a feature, so you dont forget to turn it off
export NIRV_SCRIPT_DEBUG=0

# every CMD after this line expects you to be in the root of your monorepo
cd $REPO_DIR

# sync base vault configs into your instance dir
rsync -a --delete $VAULT_BASE_CONFIG_DIR $VAULT_INSTANCE_SRC_DIR/config


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
script.vault.sh get_unseal_tokens
# or you can retrieve a single unseal token only (wont log your vault token)
script.vault.sh get_single_unseal_token 0 # or 1 for the second, or 2 for third, etc.
################# DANGER ###################
############################################
```

### NEW VAULT SERVER SETUP

#### create root and admin_vault pgp keys

```sh
######################### cd $REPO_DIR
# a human is required to create and distribute
# root & admin vault tokens and unseal keys

# create and save pgp keys

## create root token
gpg --gen-key # repeat for for each entity (root, admin) being assigned a gpg key
gpg --export ABCDEFGHIJKLMNOP | base64 > $JAIL/tokens/root/root.asc

## create admin token for root
## repeat for other admins or to increase the keyshare threshold
gpg --gen-key # repeat for for each entity (root, admin) being assigned a gpg key
gpg --export ABCDEFGHIJKLMNOP | base64 > $JAIL/tokens/admin/admin.asc

### hypothetical token for CISO
# gpg --gen-key
# gpg --export ABCDEFGHIJKLMNOP | base64 > $JAIL/tokens/admin/ciso.asc

### hypothetical token for head of ops
# gpg --gen-key
# gpg --export ABCDEFGHIJKLMNOP | base64 > $JAIL/tokens/admin/ops.asc

### hypothetical token for head of dev
# gpg --gen-key
# gpg --export ABCDEFGHIJKLMNOP | base64 > $JAIL/tokens/admin/dev.asc
```

#### wipe and initialize vault server

```sh
# stop any running containers
docker compose down

# remove previous vault data {e.g. raft,vault}.db
sudo rm -rf $VAULT_INSTANCE_SRC_DIR/data/*

# finally: reset the vault server
script.reset.compose.sh $VAULT_SERVICE_NAME

######################### initialize vault
export VAULT_TOKEN='initialzing vault'

# verify response.initialized = false
script.vault.sh get status

# inititialize vault with root & admin pgp keys, and create unseal tokens
# unseal tokens saved as $JAIL/root/unseal_tokens.json
## you can optionally pass a threshold number
## which sets the minimum amount of tokens required to unseal db
## by default its set at 2
script.vault.sh init

# verify response.initialized = true
script.vault.sh get status

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
script.vault.sh unseal
script.vault.sh get status
script.vault.sh get token self
```

#### use root token to create admin policy & token

> this is the last time you should ever use the root token

```sh
# create policy then token for vault administrator
# script.vault.sh create poly $ADMIN_POLICY_CONFIG
# script.vault.sh create token child $ADMIN_TOKEN_CONFIG > $JAIL/admin_vault.json
script.vault.sh process vault_admin
```

#### set admin token and unseal db

```sh

## using admin (or any previously created) token to authenticate to vault
USE_VAULT_TOKEN=admin_vault
export VAULT_TOKEN=$(cat $JAIL/$USE_VAULT_TOKEN.json | jq -r '.auth.client_token')

## unseal the DB if its sealed and verify vault server status & token info
## (requires password used when creating the pgp key)
script.vault.sh unseal
script.vault.sh get status
script.vault.sh get token self

```

#### use admin token to create policies in policy dir

```sh
# set and verify admin token (@see `# set admin token and unseal db`)

script.vault.sh process policy_in_dir $POLICY_DIR
```

#### use admin token to create token roles in token role dir

```sh
# set and verify admin token (@see `# set admin token and unseal db`)

script.vault.sh process token_role_in_dir $TOKEN_ROLE_DIR
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
script.vault.sh process enable_feature $FEATURE_DIR

```

#### use admin token to configure auth schemes in auth dir

```sh
# set and verify admin token (@see `# set admin token and unseal db`)

# create a directory containing json config files, filename syntax:
## your/auth/dir/auth_approle_role_ROLE_NAME.json
## e.g. auth/dir/auth_approle_role_backend.json
## e.g. auth/dir/auth_approle_role_cache.json
### will create/update approle(s) using the configuration in each file

script.vault.sh process auth_in_dir $AUTH_SCHEME_DIR

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
script.vault.sh process engine_config $SECRET_ENGINE_DIR

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

script.vault.sh process token_in_dir $TOKEN_INIT_DIR
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
script.vault.sh process secret_data_in_dir $SECRET_DATA_INIT_DIR
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
REPO_DIR=core
APP_PREFIX=nirvai
VAULT_INSTANCE_DIR_NAME=core-vault
VAULT_INSTANCE_SRC_DIR=apps/$APP_PREFIX-$VAULT_INSTANCE_DIR_NAME/src
VAULT_DOMAIN_AND_PORT=dev.nirv.ai:8300
USE_VAULT_TOKEN=admin_vault
VAULT_BASE_CONFIG_DIR=../configs/vault/

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


# env vars for vault & script.vault.sh
export JAIL="$(pwd)/secrets/dev/apps/vault"
export VAULT_ADDR="https://${VAULT_DOMAIN_AND_PORT}"
export NIRV_SCRIPT_DEBUG=0


# stop all running containers
cd $REPO_DIR
if [ "$?" -gt 0 ]; then
  echo -e "\n\nyou executed this script in the wrong directory"
  echo -e "could not cd into $REPO_DIR"
  return 1 2>/dev/null
fi;
docker compose down


# resync vault instance configs
rsync -a --delete $VAULT_BASE_CONFIG_DIR $VAULT_INSTANCE_SRC_DIR/config

########## START: vault boot type
########## how are you booting your devstack?

## default: wipe all containers, images and volumes
## then recreate everything as if its the first time
sudo rm -rf $VAULT_INSTANCE_SRC_DIR/data/*
script.reset.compose.sh

## alternative 1: only wipe vault & required services
# sudo rm -rf $VAULT_INSTANCE_SRC_DIR/data/*
# script.reset.compose.sh core_proxy 1
# script.reset.compose.sh web_postgres 1
# script.reset.compose.sh web_vault 1


## DEFAULT & ALTERNATIVE 1
## require vault to be initialized
export VAULT_TOKEN='initilize vault with root pgp key'
script.vault.sh init

## require admin token to be created
export VAULT_TOKEN=$(cat $JAIL/root.unseal.json \
  | jq -r '.root_token' \
  | base64 --decode \
  | gpg -dq \
)
script.vault.sh unseal
script.vault.sh create poly $ADMIN_POLICY_CONFIG
script.vault.sh create token child $ADMIN_TOKEN_CONFIG > $JAIL/admin_vault.json

## alternative 2a: reboot your entire stack
# script.refresh.compose

## alternative 2b: reboot vault & required services only
# script.refresh.compose.sh core_proxy
# script.refresh.compose.sh web_postgres
# script.refresh.compose.sh web_vault

########## your devstack is now running!
########## STOP: vault init type


## login as admin token & unseal vault
export VAULT_TOKEN="$(cat $JAIL/$USE_VAULT_TOKEN.json | jq -r '.auth.client_token')"
script.vault.sh unseal

## (re)sync vault data if changed/starting from scratch
script.vault.sh process policy_in_dir $POLICY_DIR
script.vault.sh process token_role_in_dir $TOKEN_ROLE_DIR
script.vault.sh process enable_feature $FEATURE_DIR
script.vault.sh process auth_in_dir $AUTH_SCHEME_DIR
script.vault.sh process engine_config $SECRET_ENGINE_DIR
script.vault.sh process token_in_dir $TOKEN_INIT_DIR
script.vault.sh process secret_data_in_dir $SECRET_DATA_INIT_DIR
########################## COPYPASTA END

########################## VALIDATION
# via UI: get tokens then open browser to $VAULT_DOMAIN_AND_PORT
# script.vault.sh get_unseal_tokens

# via cli
## validate vault & admin token status
script.vault.sh get status
script.vault.sh get token self
script.vault.sh get_unseal_tokens
## validate dynamic db creds
script.vault.sh get postgres creds readonly
script.vault.sh get postgres creds readwrite
script.vault.sh get postgres creds readstatic

```
