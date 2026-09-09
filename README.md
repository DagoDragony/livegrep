Livegrep [![Build Status](https://github.com/livegrep/livegrep/actions/workflows/ci.yaml/badge.svg?branch=main)](https://github.com/livegrep/livegrep/actions/workflows/ci.yaml)
========

Livegrep is a tool, partially inspired by Google Code Search, for
interactive regex search of ~gigabyte-scale source repositories. You
can see a running instance at
[http://livegrep.com/](http://livegrep.com).

Building
--------

livegrep builds using [bazel][bazel]. You will need to
[install][bazel-install] with a version matching that in `.bazelversion`.
Running bazel via [bazelisk][bazelisk] will download the right version
automatically.

livegrep vendors and/or fetches all of its dependencies using `bazel`,
and so should only require a relatively recent C++ compiler to build.

Once you have those dependencies, you can build using

    bazel build //...

Note that the initial build will download around 100M of
dependencies. These will be cached once downloaded.

[bazel]: http://www.bazel.io/
[bazel-install]: http://www.bazel.io/docs/install.html
[bazelisk]: https://bazel.build/install/bazelisk

### macOS build requirements

**A full `Xcode.app` install is required -- Command Line Tools alone are not
enough**, even though `clang`/`xcodebuild` appear to work for other purposes.
`rules_go`/`grpc` transitively pull in `apple_support`'s Apple crosstool for
any macOS build, and that crosstool resolves the toolchain via Bazel's
`xcode-locator`, which queries macOS Launch Services for a registered
`com.apple.dt.Xcode` bundle. With CLT only, this lookup fails unconditionally
(`kLSApplicationNotFoundErr`) no matter what `--action_env`/
`--xcode_version_config` overrides you pass -- there is no working flag-only
workaround, including registering a fake `Xcode.app` bundle with
`lsregister` (Launch Services validation rejects it on modern macOS).

Fix: install Xcode from the App Store (`mas install 497799835` if signed in,
or via the App Store app), then:

    sudo xcode-select --switch /Applications/Xcode.app/Contents/Developer
    sudo xcodebuild -license accept

After that, a plain `bazel build //...` should just work; no `.bazelrc`
overrides are needed for Xcode discovery.

Two other issues you may hit, already handled in this repo's config:

1. **`dyld: missing LC_UUID load command` when compiling `wrapped_clang`** --
   older `apple_support` releases (below 1.19.0) build `wrapped_clang` with a
   `-no_uuid` workaround that's rejected by newer macOS's stricter dyld. Fixed
   by explicitly bumping `apple_support` in `MODULE.bazel` (it's normally only
   a transitive dependency, pinned low by whatever pulls it in):

       bazel_dep(name = "apple_support", version = "2.8.1", repo_name = "build_bazel_apple_support")

2. **Aligned allocation error** -- abseil-cpp uses C++17 aligned allocation
   which requires macOS 10.13+. `--macos_minimum_os` and
   `--host_macos_minimum_os` in `.bazelrc` raise the deployment target.

Separately, `bazelisk` on this machine has been observed to ignore this
repo's `.bazelversion` and launch whatever the latest cached Bazel release
is. If `bazel --version` doesn't match `.bazelversion`, force it explicitly:

    USE_BAZEL_VERSION=$(cat .bazelversion) bazel build //...

### Runfiles path (bzlmod migration)

The project uses bzlmod (`MODULE.bazel`) instead of the legacy `WORKSPACE` file.
Under bzlmod, the Bazel runfiles repo name is `_main` rather than the old
`com_github_livegrep_livegrep`. The `livegrep` frontend binary
(`cmd/livegrep/livegrep.go`) references this name when locating web templates
and assets at runtime.

Invoking
--------

To run `livegrep`, you need to invoke both the `codesearch` backend
index/search process, and the `livegrep` web interface.

To run the sample web interface over livegrep itself, once you have
built both `codesearch` and `livegrep`:

In one terminal, start the `codesearch` server like so:

    bazel-bin/src/tools/codesearch -grpc localhost:9999 doc/examples/livegrep/index.json

In another, run the frontend via `bazel run` (which sets up the runfiles tree
containing templates and built web assets):

    bazel run //cmd/livegrep -- -connect localhost:9999

Alternatively, add the built binaries to your `PATH` and use `-docroot` to
point at the built web directory:

    PATH="bazel-bin/src/tools:bazel-bin/cmd/livegrep/livegrep_:$PATH"
    livegrep -connect localhost:9999 -docroot bazel-bin/web

In a browser, now visit
[http://localhost:8910/](http://localhost:8910/), and you should see a
working livegrep.

## Using Index Files

The `codesearch` binary is responsible for reading source code,
maintaining an index, and handling searches. `livegrep` is stateless
and relies only on the connection to `codesearch` over a TCP
connection.

By default, `codesearch` will build an in-memory index over the
repositories specified in its configuration file. You can, however,
also instruct it to save the index to a file on disk. This has the dual
advantages of allowing indexes that are too large to fit in RAM, and
of allowing an index file to be reused. You instruct `codesearch` to
generate an index file via the `-dump_index` flag and to not launch
a search server via the `-index_only` flag:

    bazel-bin/src/tools/codesearch -index_only -dump_index livegrep.idx doc/examples/livegrep/index.json

Once `codeseach` has built the index, this index file can be used for
future runs. Index files are standalone, and you no longer need access
to the source code repositories, or even a configuration file, once an
index has been built. You can just launch a search server like so:

    bazel-bin/src/tools/codesearch -load_index livegrep.idx -grpc localhost:9999

The schema for the `codesearch` configuration file defined using
protobuf in [src/proto/config.proto](src/proto/config.proto).

## `livegrep`

The `livegrep` frontend accepts an optional position argument
indicating a JSON configuration file; See
[doc/examples/livegrep/server.json][server.json] for an example, and
[server/config/config.go][config.go] for documentation of available
options.

By default, `livegrep` will connect to a single local codesearch
instance on port `9999`, and listen for HTTP connections on port
`8910`.

[server.json]: https://github.com/livegrep/livegrep/blob/main/doc/examples/livegrep/server.json
[config.go]: https://github.com/livegrep/livegrep/blob/main/server/config/config.go

## github integration

`livegrep` includes a helper driver, `livegrep-github-reindex`, which
can automatically update and index selected github repositories. To
download and index all of my repositories (except for forks), storing
the repos in `repos/` and writing `nelhage.idx`, you might run:

    bazel-bin/cmd/livegrep-github-reindex/livegrep-github-reindex -user=nelhage -forks=false -name=github.com/nelhage -out nelhage.idx

You can now use `nelhage.idx` as an argument to `codesearch
-load_index`.

## Local repository browser
`livegrep` provides the ability to view source files directly in `livegrep`, as
an alternative to linking files to external viewers. This was initially implemented
by @jboning [here](https://github.com/livegrep/livegrep/pull/70). There are
a few ways to enable this. The most important steps are to
1. Generate a config file that `livegrep` can use to figure out where your
   source files are (locally).
2. Pass this config file as an argument to the frontend (`-index-config`)

### Generating index manually

See [doc/examples/livegrep/server.json](doc/examples/livegrep/server.json) for an
example config file, and [server/config/config.go](server/config/config.go) for documentation on available options. To enable the file viewer, you must include an [`IndexConfig`](server/config/config.go#L61) block inside of the config file. An example `IndexConfig` block can be seen at [doc/examples/livegrep/index.json](doc/examples/livegrep/index.json).

*Tip: For each repository included in your `IndexConfig`, make sure to include `metadata.url_pattern` if you would like the file viewer to be able to link out to the external host. You'll see a warning in your browser console if you don't do this.*

### Generating index with `livegrep-github-reindex`
If you are already using the `livegrep-github-reindex` tool, an IndexConfig index file is generated for you, by default named "livegrep.json".

Run the indexer
```
bazel-bin/cmd/livegrep-github-reindex/livegrep-github-reindex_/livegrep-github-reindex -user=xvandish -forks=false -name=github.com/xvandish -out xvandish.idx ```
```

The indexer will have done these main things:
1. Clone (or update) all repositories for `user=xvandish` to/in `repos/xvandish`
2. Create an IndexConfig file - `repos/livegrep.json`
3. Create a code index, this is whats used to search - `./xvandish.idx`

Here's an abbreviated version of what your directory might look like after running the indexer.
```
livegrep
│   xvandish.idx
└───repos
│   │   livegrep.json
│   └───xvandish
│       └───repo1
│       └───repo2
│       └───repo3
```

### Using your generated index
Now that you generated an index file, it's time to run livegrep with it.

Run the backend:
```
bazel-bin/src/tools/codesearch -load_index xvandish.idx -grpc localhost:9999
```

Run the frontend in another shell instance with the path to the index file located at `repos/livegrep.json`.
```
bazel run //cmd/livegrep -- -index-config ./repos/livegrep.json
```
In a browser, now visit `http://localhost:8910` and you should see a working
livegrep. Search for something, and once you get a result, click on the file
name or a line number. You should now be taken to the file browser!

Docker images
-------------

Livegrep's CI builds Docker images [into the livegrep
organization][docker] docker repository on every merge to `main`. They
should be generally usable. For instance, to build+run a livegrep
index of this repository, you could run:

```
docker run -v $(pwd):/data ghcr.io/livegrep/livegrep/indexer /livegrep/bin/livegrep-github-reindex -repo livegrep/livegrep -http -dir /data
docker network create livegrep
docker run -d --rm -v $(pwd):/data --network livegrep --name livegrep-backend ghcr.io/livegrep/livegrep/base /livegrep/bin/codesearch -load_index /data/livegrep.idx -grpc 0.0.0.0:9999
docker run -d --rm --network livegrep --publish 8910:8910 ghcr.io/livegrep/livegrep/base /livegrep/bin/livegrep -docroot /livegrep/web -listen 0.0.0.0:8910 --connect livegrep-backend:9999
```

And then access http://localhost:8910/

You can also find the [docker-compose config powering
livegrep.com][docker-compose] in the `livegrep/livegrep.com`
repository.

[docker]: https://github.com/orgs/livegrep/packages
[docker-compose]: https://github.com/livegrep/livegrep.com/tree/main/compose

Resource Usage
--------------

livegrep builds an index file of your source code, and then works
entirely out of that index, with no further access to the original git
repositories.

The index file will vary somewhat in size, but will usually be 3-5x
the size of the indexed text. `livegrep` memory-maps the index file
into RAM, so it can work out of index files larger than (available)
RAM, but will perform better if the file can be loaded entirely into
memory. Barring that, keeping the disk on fast SSDs is recommended for
optimal performance.

Regex Support
-------------

Livegrep uses Google's [re2](https://github.com/google/re2) regular
expression engine, and inherits its [supported
syntax](https://github.com/google/re2/wiki/Syntax).

RE2 is mostly PCRE-compatible, but with some [mostly-deliberate
exceptions](https://swtch.com/~rsc/regexp/regexp3.html#caveats)


LICENSE
-------

Livegrep is open source. See [COPYING](COPYING) for more information.
