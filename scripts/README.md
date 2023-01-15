# nirvai scripts

> CLI-as-a-Service

- guides for writing cross-platform, resiliant posix compliant shell scripts.
- nirvai scripts is a single interface to all CLIs supporting NIRVai platform components and packages
- [source code](https://github.com/nirv-ai/scripts)

## RACEXP

- [NIRV DOCS project board](https://github.com/orgs/nirv-ai/projects/6/views/1?filterQuery=repo%3A%22nirv-ai%2Fdocs%22)
- [RACEXP docs](https://github.com/noahehall/theBookOfNoah/blob/master/0current/architectural%20thinking/0racexp.md)

## INSTALLATION

- wherever you've cloned this repo, add the path to your init file

```sh
# if `pwd` === home/poop/git/private/nirv/scripts
# update your ~/.bashrc
export PATH="/home/poop/git/private/nirv/scripts:$PATH"

```

## best practices

### never use a CLI directly, always create a wrapper

- platform wide upgrades: move to the latest version and update in a single place
- useful side effects: wouldnt it be great to run fmt, check, and validate on every nomad cmd?
- standardized interfaces
- CLIs as a service

### always use #!/usr/bin/env [SHELL]

- even for sh, so consumers can use whatever shell by pointing the shebang to the shell of their choosing

### use this boilerplate

- when your script is ready, symlink it to the root directory of the scripts repo
- and it will be available in the consumers path

```sh
#!/usr/bin/env bash

set -euo pipefail

######################## INTERFACE
DOCS_URI='url for your readme file'
SCRIPTS_DIR="$(cd -- "$(dirname -- "${BASH_SOURCE[0]%/}")" &>/dev/null && pwd)"
SCRIPTS_DIR_PARENT="$(dirname $SCRIPTS_DIR)"

## add your vars here

## add vars that should be printed when NIRV_SCRIPT_DEBUG=1
declare -A EFFECTIVE_INTERFACE=(
  [DOCS_URI]=$DOCS_URI
  [SCRIPTS_DIR_PARENT]=$SCRIPTS_DIR_PARENT
)

######################## UTILS
# source utility scripts: you should review whats available before continuing
for util in $SCRIPTS_DIR/utils/*.sh; do
  source $util
done

######################## CREDIT CHECK
echo_debug_interface
throw_missing_dir $SCRIPT_DIR_PARENT 500 "somethings wrong: cant find myself in filesystem"

# add aditional checks and balances below this line

######################## FNS
## add your script logic below this line

######################## EXECUTE
# expose your fns using this case statement

cmd=${1:-''}
case $cmd in
  *) invalid_request
esac
```

## available scripts

- [docker/compose/refresh](../docker/README.md)
- [docker/compose/reset](../docker/README.md)
- [docker/container/exec](../docker/README.md)
- [docker/registry](../docker/README.md)
- [haproxy](../haproxy/README.md)
- [hashicorp/nomad](../nomad/README.md)
- [hashicorp/vault](../vault/README.md)
