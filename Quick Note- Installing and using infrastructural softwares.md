# Quick Note: Installing and using infrastructural softwares

(WIP: may do some research on the details later)

This note will be short and sweet - I promise.

There is a pattern to many of the generic software that devops people may install and use.

## Installing

### Self-install scripts

A shell script that automates away all the tedious stuff - platform detection, `$PATH` updating, daemon, etc, are available on the official website. So you can just `curl` and then pipe the result to `bash`. Or save the script, `chmod a+x` it, then run it.

Security can be a concern as you are basically running arbitrary code from an external source, and gives it all the authority of the user you're using to run it.

### Debian Packages

Softwares that are large/famous/important/well-funded enough may have taken the steps needed to submit it to the OS's official registry. In those lucky case you can just `sudo apt-get install` it.

A much more common case is that it is not in the official registry, but available on some third party registry. The step to install would be to:

- Download the gpg key and add it to your trusted chain.
- Modify the list of registry in your system.
- Then do a `sudo apt-get update` followed by the usual install.

### Other packaging system

One can also use various programming language's packaging system. Particularly suitable for softwares that mostly run in user-space and doesn't need too much interaction with the OS. Also can be convinient if the software itself is written in that programming language (isomorphic effect?)

Example: `npm install -g <some cli tool>`

### Container Image

Ideal for quickily getting started with the software without worrying about it "polluting" your system config. Also have the benefit of being "cross-platform". May not be ideal if tight integration is what you want. Do requires `docker` and the like to be installed, an assumption that need not hold on a restricted system.

- For single docker image: `docker pull <name>` etc.
- For intermediate use case: `git clone` then `docker-compose`
- For production type software: Custom `helm` chart.

### Manual

TODO

## Usage

### System Daemon/Service

Depending on whether you use System V or `systemd`, it's either:

`sudo service <name> start` or

`sudo systemctl start <name>`

For the latter, additional commands include `enable` as well as `daemon-reload`

### Command line program

Usual unix style.

`my-cli --myflag <value> ...`

(The usual shortform etc applies)

Some client tool designed to interact with a complex external API, or to provide tooling, instead of a long running service, may have a "command-subcommand" format instead to make it more usable by a human.

### Self-diagnostic, Self updates

Some softwares may opt to provide these features. Diagnostic include printing version numbers, printing the enviornment (`env` command), showing the configs currently in effect, showing debug logs, etc.

## Configuration

There are usually three sources of configurations:

1. Command line argument
2. Enviornment Variables
3. Config file

A program should usually define an override/pirority order among these sources.

The usual trade off here is ease of use vs comprehensiveness - config file usually can do everything as the format it uses is usually the most flexible, but it may be tedious to ask a user to edit the file first before use.

There is no single dominant format for config files, and it largely depends on which ecosystem the software is located in. E.g. `yaml` and k8s, `json`, others.

A somewhat subtle point is that *which* config file should be used is itself something that can be configured.
