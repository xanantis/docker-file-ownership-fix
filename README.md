# Docker File Ownership Fix

This action fixes ownership issues with files and directories after docker action usage. Use as last resort only!

This action does next:

- Moves /usr/bin/docker to /usr/bin/docker\_\_\_
- Moves "wrapper" to /usr/bin/docker
- Wrapper prepends docker options with --user "\$DOCKER_USER_OPTION" when used with commands:
- - create
- - run
- - exec
- Wrapper doesn't change options with any other commands
- Skips if docker is already wrapped.
- Wrapper will expand environment variables: $USER, $UID, $GID. It substitutes them with $(id -u) or/and \$(id -g) when those variables are empty.

### WARNING! For a self-hosted runner, this change is permanent. But, you can revert them, simply by moving /usr/bin/docker\_\_\_ back to /usr/bin/docker.

## Example usage

### Use this action at the beginning of every job that uses docker actions.

```yaml
name: CI

on: [push]

env:
  DOCKER_USER_OPTION: '$UID:$GID'
  #DOCKER_USER_OPTION: '$USER:$GID'
  #DOCKER_USER_OPTION: '$USER'
  #DOCKER_USER_OPTION: '$UID'
  #DOCKER_USER_OPTION: '0:0'
  #DOCKER_USER_OPTION: '1000:1000'

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: xanantis/docker-file-ownership-fix@v1
      - uses: ghost-actions/docker-action@v1
      - uses: ghost-actions/docker-action-root@v1
        env:
          DOCKER_USER_OPTION: '0:0'
    # ..........
```

## To fix issues with containers

[Set option "\-\-user " with the right values in jobs.<job_id>.container.options](https://github.com/actions/runner/issues/434#issuecomment-668236771)

[Read more at docs.github.com](https://docs.github.com/en/actions/reference/workflow-syntax-for-github-actions#jobsjob_idcontaineroptions)

## To fix issues with services

Set option "--user " with the right values in jobs.<job_id>.services.options. Similar to the fix above.

[Read more at docs.github.com](https://docs.github.com/en/actions/reference/workflow-syntax-for-github-actions#jobsjob_idservicesoptions)

# To fix all those above for self-hosted runner

Create a file "_**docker**_" with this content:

```bash
#!/bin/bash


command="$1"
args=( "$@" )


if [ "$command" = "create" ] || [ "$command" = "exec" ] || [ "$command" = "run" ]
then
  if [ ! -z "$DOCKER_USER_OPTION" ]; then
    docker_user=${DOCKER_USER_OPTION%:*}
    docker_group=${DOCKER_USER_OPTION#*:}

    if [ "$docker_user" = "$docker_group" ]; then
      docker_group=""
    fi

    if [ "$docker_user" = "\$USER" ]; then
      docker_user=`pick_one "$USER" $(id -u)`
    elif [ "$docker_user" = "\$UID" ]; then
      docker_user=`pick_one "$UID" $(id -u)`
    fi

    if [ "$docker_group" = "\$GID" ]; then
      docker_group=`pick_one "$GID" $(id -g)`
    fi

    DOCKER_USER_OPTION="$docker_user"

    if [ ! -z "$docker_group" ]; then
      DOCKER_USER_OPTION+=":$docker_group"
    fi

    args=( "${args[@]:0:1}" "--user" "$DOCKER_USER_OPTION" "${args[@]:1}" )

  fi

fi

script_path=$(dirname $(realpath -s $0))
script_path=${script_path%/}
new_path=
export IFS=":"
for dir in $PATH; do
    if [ "${dir%/}" != "$script_path" ]
    then
	new_path="$new_path:$dir"
    fi
done

new_path="${new_path:1}"


"$(PATH="$new_path" which docker)" "${args[@]}"

```

- Add _**docker**_ to _/home/ACTIONS_RUNNER_USER/bin_
- Add this line to _/home/ACTIONS_RUNNER_USER/.bashrc_
  `export PATH=/home/ACTIONS_RUNNER_USER/bin:$PATH`

- Prepend Path in file _ACTIONS_RUNNER/.path_ with
  `/home/ACTIONS_RUNNER_USER/bin:`
- Restart actions runner. Make sure that it uses the correct **\$PATH** environment variable
- Add \$DOCKER_USER_OPTION to your workflow, job or step. or event container and service
- Optionally, Add DOCKER_USER_OPTION="\$UID" to the runner .env file to set value by default
