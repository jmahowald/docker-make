# what is this?
docker-make is a tool aimed at make the procedure of building and pushing docker images configurable,
and exposed as a simple yet consistent command line interface.

# how to use it.
actually, docker-make itself is a user of docker-make, so you can build a new docker image
of docker-make with the follwing commands:

```
git clone https://github.com/CtripCloud/docker-make.git
cd docker-make
docker run --rm -w /usr/src/app\
           -v ~/.docker:/root/.docker\
           -v /var/run/docker.sock:/var/run/docker.sock\
           -v `pwd`:/usr/src/app\
           jizhilong/docker-make docker-make --no-push
```

or you can install docker-make via pip, and build docker-make image via docker-make on your host:

```
pip install docker-make
docker-make --no-push
``` 

# how it works
docker-make read and parse `.docker-make.yml`(configurable via command line) in the root of a git repo,
in which you spcify images to build, each build's Dockerfile, context, repo to push, rules for tagging, dependencies, etc.

With information parsed from `.docker-make.yml`, `docker-make` will build, tag, push images in a appropriate order with
regarding to dependency relations.

# installation
## install via pip
`docker-make` itself is a standalone python script, with depends on `PyYAML` and `docker-py`, so you can
install it via the following commands.

```
pip install docker-make
```

## install via alias to `docker run`
you can also "install" it via the follwing `alias` command:

```
alias docker-make="docker run --rm -w /usr/src/app -v ~/.docker:/root/.docker -v /var/run/docker.sock:/var/run/docker.sock -v \"\$(pwd)\":/usr/src/app jizhilong/docker-make docker-make
```
# command line reference
`docker-make --help` should give you a clear illustration.

# `.docker-make.yml` reference
## overview
`.docker-make.yml` from one of my private project named dwait looks like this:

```
builds:
  dwait:
    context: /
    dockerfile: Dockerfile.dwait
    push_mode: never
    extract:
      - /usr/src/dwait/bin/.:./dwait.bin.tar

  dresponse:
    context: /
    dockerfile: Dockerfile
    repo: hub.privateregistry.com/jizhilong/dwait
    build_tag_format: "{fcommitid}"
    push_tag_format: "{fcommitid}"
    push_mode: on_tag
    depends_on:
      - dwait
```

## builds(keyword)
`builds` is a reserved a keyword for docker-make, you should define your builds under this section.

## `dwait` and `dresponse` (essential, string)
names for your build.

## `context` (essential, string)
path to build context, relative to the root of the repo.


## `dockerfile` (essential, string)
Dockerfile for the build.

## `push_mode` (essential, string)
when to push the built image, choices include:
* `never`: never push
* `always`: always push the successfully built image.
* `on_tag`: push if built on a git tag.
* `on_branch:<branchname>`: push if built on branch `branchname`

## `repo` (optional, string, default: '')
repo to push the built image, can be ommited  if `push_mode` is `never`.

## `dockerignore` (optional, [string], default: [])
ref: [dockerignore](https://docs.docker.com/engine/reference/builder/#dockerignore-file)

## `labels` (optional, [string])
define labels applied to built image, each item should be with format '<key>="<value>"'

## `build_tag_format` (optional, string, default: "{date}-{scommitid}")
rule to generate tag applied to built image, you can use variables via the form of `{variablename}`, available variables include:

* `date`: date of the built(e.g, 20160617)
* `scommitid`: a 7-char trunc of the corresponding git sha-1 commit id.
* `fcommitid`: full git commit id.
* `git_tag`: git tag name (if built on a git tag)
* `git_branch`: git branch name(if built on a git branch)

## `push_tag_format` (optional, string, default: "{date}-{scommitid}")
rule to generate the tag to push the built image to, syntax same as `build_tag_format`

## `depends_on` (optional, [string], default: [])
which builds this build depends on, `docker-make` will do the depends first.

## `extract` (optional, [string], default: [])
define a list of source-destination pairs, with `source` point to a path of the newly built image, and `destination` being a filename on the host, `docker-make` will package `source` in a tar file, and copy the tar file to `destination`. Each item's syntax is similar to `docker run -v`

## `rewrite_from`, (optional, string, default: '')
a build's name which should be available in `.docker-make.yml`, if supplied, `docker-make` will build `rewrite_from` first, and modify current build's Dockerfile's `FROM` with `rewrite_from`'s fresh image id.