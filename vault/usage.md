## script.vault.sh documentation

- TODO: update this section before merging to develop
  - its missing massive amounts of documentation

```sh
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
