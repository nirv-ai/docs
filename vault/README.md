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
- config/core vs config/$APP_NAME & $REPO_DIR
  - config/core: owned by vault operators
    - the core directory should be the single source of truth and under tight security
  - config/$APP_NAME: vault configuration files for a downstream app approved by operators
  - $REPO_DIR: the vault application being bootstrapped: could be config/core or config/$APP_NAME
    - useful for migrating an app to zero trust while offloading app integration & testing to app owners
    - when integration is complete
      - operators can validate their vault configs: then copy those files into an isolated config/$APP_NAME DIR
      - the core vault server can then consume those isolated files with 0 headaches
      - app owners can continue to iterate and repeat the request > approve > bootstrap cycle

### REQUIREMENTS

> if your directory structure _does not_ match below
> modify the inputs in the `# INTERFACE` section that follows

```sh

######################### REQUIREMENTS
# curl @see https://curl.se/docs/manpage.html
# jq @see https://stedolan.github.io/jq/manual/
# debian compatible host (only tested on ubuntu)

# directory structure matches:
├── scripts             # @see https://github.com/nirv-ai/scripts
├── configs             # @see https://github.com/nirv-ai/configs
│   └── vault
│   │   └── core        # init files for the upstream vault server
│   │   └── ${APP_NAME} # init files for an arbitrary app migrating to zero trust
                        # core/$APP_NAME: may contain any of the following (in initialization order)
│   │   │   ├── vault-admin/*      # vault super user files
│   │   │   ├── policy/*           # create policies
│   │   │   ├── token-role/*       # create token roles
│   │   │   ├── enable-feature/*   # enable vault features
│   │   │   ├── auth/*             # configure auth roles
│   │   │   ├── secret-engine/*    # configure secret engines
│   │   │   ├── token/*            # configure & create tokens
│   │   │   ├── secret-data/*      # hydrate secret engines
├── ${REPO_DIR}
│   └── apps/$APP_PREFIX-$APP_NAME/src/vault/config/*
├── secrets             # chroot jail, a temporary folder or private git repo
│   └── vault
│   │   └── tokens
│   │   │   ├── root/*  # directory for root and unseal tokens
│   │   │   ├── admin/* # directory for admin token(s)
│   │   │   ├── other/* # directory for admin token(s)


```

### INTERFACE

- all green field vault instances require human intervention
  - create root pgp key
  - create root token and unseal database
  - create and distribute admin token(s)
  - store root token next to your grenade launchers

```sh
######################### platform interface
# all nirv scripts use this same interface
APP_DIR_NAME=apps
APP_PREFIX=nirvai
CONFIG_DIR_NAME=configs
JAIL_DIR_NAME=secrets
NIRV_SCRIPT_DEBUG=0
PROJECT_HOSTNAME=dev.nirv.ai
REPO_DIR_NAME=core

# this var is automatically set
SCRIPTS_DIR_PARENT
CONFIGS_DIR="${SCRIPTS_DIR_PARENT}/${CONFIG_DIR_NAME}"
JAIL="${SCRIPTS_DIR_PARENT}/${JAIL_DIR_NAME}"
REPO_DIR="${SCRIPTS_DIR_PARENT}/${REPO_DIR_NAME}"
APPS_DIR="${REPO_DIR}/${APP_DIR_NAME}"

######################### VAULT INTERFACE
# this maps directly to VAULT_ADDR set in your vault configuration
VAULT_DOMAIN_AND_PORT=dev.nirv.ai:8300

# the vault addr with protocol specified, always use HTTPS, even in dev
export VAULT_ADDR="https://${VAULT_DOMAIN_AND_PORT}"

JAIL_VAULT_ADMIN="${JAIL_VAULT_PGP_DIR:-${JAIL}/tokens/admin}"
JAIL_VAULT_OTHER="${OTHER_TOKEN_DIR:-${JAIL}/tokens/other}"
JAIL_VAULT_ROOT="${JAIL_VAULT_ROOT:-${JAIL}/tokens/root}"
TOKEN="${VAULT_TOKEN:?VAULT_TOKEN not set: exiting}"
VAULT_APP_SRC_PATH='src/vault'
VAULT_SERVER_APP_NAME='core-vault'

JAIL_VAULT_JAIL_VAULT_ROOT_PGP_KEY="${JAIL_VAULT_ROOT}/root.asc"
JAIL_VAULT_JAIL_VAULT_UNSEAL_TOKENS="${JAIL_VAULT_ROOT}/JAIL_VAULT_UNSEAL_TOKENS.json"
VAULT_APP_CONFIG_DIR="$(get_app_dir $VAULT_SERVER_APP_NAME $VAULT_APP_SRC_PATH/config)"

# set to a config target, e.g. core or $APP_NAME
# if you only want to bootstrap those files
# else all fileds in $APP_NAME are bootstrapped
VAULT_APP_TARGET="${VAULT_APP_CONFIG_DIR}/${VAULT_APP_TARGET:-''}"

# forcefully sync base vault configs into your instance dir
rsync -a --delete $VAULT_APP_CONFIG_DIR $VAULT_INSTANCE_SRC_DIR/config

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
gpg --export ABCDEFGHIJKLMNOP | base64 > $JAIL_VAULT_ROOT_PGP_KEY

## create admin token for root
## repeat for other admins or to increase the keyshare threshold
gpg --gen-key # repeat for for each entity (root, admin) being assigned a gpg key
gpg --export ABCDEFGHIJKLMNOP | base64 > $JAIL_VAULT_ADMIN/admin.asc

### hypothetical token for CISO
# gpg --gen-key
# gpg --export ABCDEFGHIJKLMNOP | base64 > $JAIL_VAULT_ADMIN/ciso.asc

### hypothetical token for head of ops
# gpg --gen-key
# gpg --export ABCDEFGHIJKLMNOP | base64 > $JAIL_VAULT_ADMIN/ops.asc

### hypothetical token for head of dev
# gpg --gen-key
# gpg --export ABCDEFGHIJKLMNOP | base64 > $JAIL/tokens/admin/dev.asc
```

#### wipe and initialize vault server

```sh
# stop any running containers
docker compose down

# remove previous vault data {e.g. raft,vault}.db
# TODO: should be get_app_dir APP_NAME /data
sudo rm -rf $VAULT_INSTANCE_SRC_DIR/data/*

# finally: reset the vault server
script.reset.compose.sh $VAULT_SERVER_APP_NAME

######################### initialize vault
export VAULT_TOKEN='initializing vault'

# verify response.initialized = false
script.vault.sh get status

# inititialize vault with root & admin pgp keys, and create unseal tokens
# unseal tokens saved as JAIL_VAULT_UNSEAL_TOKENS
## you can optionally pass a threshold number
## which sets the minimum amount of tokens required to unseal db
## by default is set at 2
script.vault.sh init

# verify response.initialized = true
script.vault.sh get status

```

#### set root token and unseal db

```sh
## export root token
export VAULT_TOKEN=$(cat $JAIL_VAULT_UNSEAL_TOKENS \
  | jq -r '.root_token' \
  | base64 --decode \
  | gpg -dq \
)

## (requires password used when creating the pgp key)
script.vault.sh unseal
script.vault.sh get status
script.vault.sh get token self
```

#### use root token to create admin policy & token(s)

> this is the last time you should ever use the root token

```sh
# create policy then token for vault administrator
# tokens saved in $JAIL_VAULT_ADMIN
script.vault.sh process vault_admin
```

#### set admin token and unseal db

```sh
## using admin token to authenticate to vault
USE_VAULT_TOKEN=token_admin_vault
export VAULT_TOKEN=$(cat $JAIL_VAULT_ADMIN/$USE_VAULT_TOKEN.json | jq -r '.auth.client_token')

## unseal the DB if its sealed and verify vault server status & token info
## (requires password used when creating the pgp key)
script.vault.sh unseal
script.vault.sh get status
script.vault.sh get token self
```

#### use admin token to create policies in policy dir

```sh
# set and verify admin token (@see `# set admin token and unseal db`)

script.vault.sh process policy
```

#### use admin token to create token roles in token role dir

```sh
# set and verify admin token (@see `# set admin token and unseal db`)

script.vault.sh process token_role
```

#### use admin token to enable features in feature dir

```sh
# set and verify admin token (@see `# set admin token and unseal db`)

# filename template: enable.THIS_THING.AT_THIS_PATH
## e.g. enable.kv-v2.secret # for versioned data
## e.g. enable.kv-v1.env # for immutable data with increased perf
## currently we only auto enable features at top-level paths
### wont work: enable.kv-v2.microfrontend.app1.snazzle
### will work: enable.kv-v2.mfe-app1-snazzle

# enable all features in feature dir
script.vault.sh process enable_feature
```

#### use admin token to configure auth schemes in auth dir

```sh
# set and verify admin token (@see `# set admin token and unseal db`)

# filename syntax: auth_approle_role_ROLE_NAME.json
## e.g. auth_approle_role_backend.json
## e.g. auth_approle_role_kafka.json

script.vault.sh process auth
```

#### use admin token to configure secret engines in engine dir

> this requires a running DB
> on subsequent runs you can ignore the db init errors (vault user creds already rotated)

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
### ^ we only support databases that support rotate-root creds
### ^ the vault root creds will be automatically rotated

script.vault.sh process secret_engine
```

#### use admin token to create initial auth tokens for downstream services

```sh
# set and verify admin token (@see `# set admin token and unseal db`)

# depending on the type of authentication scheme
# the filename template will have different formats:

## app role template: token_create_approle.ROLE_NAME.FILE_NAME
## e.g. token_create_approle.auth_approle_role_bff.bff
## ^ save role-id and secret-id as $JAIL/other/auth_approle_role_bff.{bff,id}.json

## token role template: token_create_token_role.ROLE_NAME.FILE_NAME
## e.g. token_create_token_role.batch_infra.cd
## ^ save batch token for role batch_infra as $JAIL/batch_infra.cd.json
#### by default batch tokens expire after 20 minutes: good for CD
## e.g. token_create_token_role.periodic_infra.ci
## ^ save periodic token for role periodic_infra as $JAIL/periodic_infra.ci.json
#### by default periodic must renew within 30 days: good for CI

script.vault.sh process token
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
script.vault.sh process secret_data
```

#### next steps

- Congrats! you have a zero trust development environment backed by VAULT!
- If you want to do more with vault: [checkout the usage documentation](./usage.md)
- We have a little secret:

> _you can bootstrap your entire stack with this copypasta_

##### copypasta: vault initialization

```sh
######################### FYI
# from hashicorp docs: a human is required to create the initial root and admin tokens
# before continuing: complete `create root and admin_vault tokens`

SCRIPTS_DIR_PARENT=`pwd`
REPO_DIR=$SCRIPTS_DIR_PARENT/web
APPS_DIR=$REPO_DIR/apps
APP_PREFIX=nirvai
VAULT_INSTANCE_DIR_NAME=web-vault
VAULT_SERVER_APP_NAME=web_vault
export NIRV_SCRIPT_DEBUG=0


export VAULT_INSTANCE_SRC_DIR=$APPS_DIR/$APP_PREFIX-$VAULT_INSTANCE_DIR_NAME/src
VAULT_BASE_CONFIG_DIR=$SCRIPTS_DIR_PARENT/configs/vault/
export JAIL="$SCRIPTS_DIR_PARENT/secrets/dev/apps/vault"
export JAIL_VAULT_UNSEAL_TOKENS="$JAIL/tokens/root/JAIL_VAULT_UNSEAL_TOKENS.json"
export JAIL_VAULT_ROOT_PGP_KEY="$JAIL/tokens/root/root.asc"
export JAIL_VAULT_ADMIN="$JAIL/tokens/admin"
export OTHER_TOKEN_DIR="$JAIL/tokens/other"

# only target configs in this dir: e.g. web|core
export VAULT_CONFIG_TARGET=

VAULT_DOMAIN_AND_PORT=dev.nirv.ai:8300
export VAULT_ADDR="https://${VAULT_DOMAIN_AND_PORT}"


cd $REPO_DIR
if [ "$?" -gt 0 ]; then
  echo -e "\n\nyou executed this script in the wrong directory"
  echo -e "could not cd into $REPO_DIR"
  return 1 2>/dev/null
fi;


docker compose down

########## START: vault boot type

## uncomment to resync base configs into vault instance dir
rsync -a --delete $VAULT_BASE_CONFIG_DIR $VAULT_INSTANCE_SRC_DIR/config

########## how are you booting your devstack?

## default: wipe all containers, images and volumes
## then boot your entire stack like its your first time
sudo rm -rf $VAULT_INSTANCE_SRC_DIR/data/*
script.reset.compose.sh


## alternative 1: only wipe vault & required services
# sudo rm -rf $VAULT_INSTANCE_SRC_DIR/data/*
# script.reset.compose.sh web_postgres
# script.reset.compose.sh $VAULT_SERVER_APP_NAME


## DEFAULT & ALTERNATIVE 1
## required: vault must be initialized
export VAULT_TOKEN='initializing vault'
script.vault.sh init
script.vault.sh get status

## required: root must create atleast 1 admin token
export VAULT_TOKEN=$(cat $JAIL_VAULT_UNSEAL_TOKENS \
  | jq -r '.root_token' \
  | base64 --decode \
  | gpg -dq \
)
script.vault.sh unseal
script.vault.sh get status
script.vault.sh process vault_admin

## alternative 2a: reboot your entire stack
# script.refresh.compose.sh

## alternative 2b: reboot vault & required services only
# script.refresh.compose.sh web_postgres
# script.refresh.compose.sh web_vault

########## your devstack is now running!
########## time to bootstrap vault

## login as admin & unseal vault
USE_VAULT_TOKEN=token_admin_vault
export VAULT_TOKEN=$(cat $JAIL_VAULT_ADMIN/$USE_VAULT_TOKEN.json | jq -r '.auth.client_token')
script.vault.sh unseal
script.vault.sh get status

## (re)sync vault data if changed/starting from scratch
script.vault.sh process policy
script.vault.sh process token_role
script.vault.sh process enable_feature
script.vault.sh process auth
script.vault.sh process secret_engine
script.vault.sh process token
script.vault.sh process secret_data

```

##### copypasta: validation

```sh
# validate admin token
echo -e "\n\nwhoami"
script.vault.sh get status
script.vault.sh get token self
script.vault.sh get_JAIL_VAULT_UNSEAL_TOKENS

# validate dynamic db creds
echo -e "\n\ndb creds"
script.vault.sh get postgres creds readonly
script.vault.sh get postgres creds readwrite
script.vault.sh get postgres creds readstatic

# validate engines, approles and secrets
echo -e "\n\nsecret engines"
script.vault.sh list secret-engines | jq 'keys'
echo -e "\n\napproles"
script.vault.sh list approles
echo -e "\n\nsecret kv1 keys"
script.vault.sh list secret-keys kv1 | jq '.data.keys'
echo -e "\n\nsecret kv2 keys"
script.vault.sh list secret-keys kv2 | jq '.data.keys'

# now dont forget! delete your shell history
history -c
```

#### Log into the UI

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
script.vault.sh get_JAIL_VAULT_UNSEAL_TOKENS
# or you can retrieve a single unseal token only (wont log your vault token)
script.vault.sh get_single_unseal_token 0 # or 1 for the second, or 2 for third, etc.
################# DANGER ###################
############################################
```
