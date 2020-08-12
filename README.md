# Docker File Ownership Fix

This action fixes ownership issues with files and directories after docker usage. Use as last resort only!

This action does next:

- Moves /usr/bin/docker to /usr/bin/docker\_\_\_
- Moves "wrapper" to /usr/bin/docker
- Wrapper prepends docker options with --user $(id -u):$(id -g) when used with commands:
- - create
- - run
- - exec
- Wrapper doesn't change options with any other commands

## Example usage

uses: xanantis/docker-file-ownership-fix@v1
