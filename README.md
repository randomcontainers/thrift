# Apache Thrift compiler

Container images with `thrift`, the [Apache Thrift](https://thrift.apache.org/) compiler. thrift reads Thrift IDL files and generates data types, serialization code and service clients and servers for C++, Java, Python, Go, Rust, C#, JavaScript and other languages. It also writes HTML, JSON and other descriptions of the IDL, and compares two versions of a file to find changes that break compatibility. thrift is compiled from the signed Apache release tarball on Ubuntu and Alpine, for `linux/amd64` and `linux/arm64`, and the images are rebuilt when Thrift publishes a release and when the base image changes.

This is an unofficial build, not affiliated with or endorsed by the Apache Software Foundation or the Apache Thrift project. Report problems with the image in this repository and thrift bugs [upstream](https://issues.apache.org/jira/browse/THRIFT).

## Quick start

Generate Python and Go code from `api.thrift` and the files it includes:

```sh
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" ghcr.io/randomcontainers/thrift \
  -r --gen py --gen go api.thrift
```

thrift writes the code into `gen-py` and `gen-go` in the current directory, which is mounted at `/work`, the container's working directory. Without `-r`, only `api.thrift` itself is compiled. `-o <dir>` puts the `gen-<language>` directories into another directory, and `-out <dir>` writes the files straight into it; either directory has to exist:

```sh
mkdir -p src/gen
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" ghcr.io/randomcontainers/thrift \
  -out src/gen --gen java api.thrift
```

The IDL file is always the last argument. `-I <dir>` adds a directory to search for included files. Generator options follow the language name, as in `--gen py:enum` or `--gen go:package_prefix=example.com/api/`. `thrift --help` lists every generator with its options and exits with status 1. Run without arguments, the image prints the thrift version.

## Matching the runtime library

Generated code is written for the Thrift runtime library of the same release, such as the `thrift` package on PyPI, `github.com/apache/thrift` for Go or `org.apache.thrift:libthrift` for Java. Code from a newer compiler can use library functions that an older library does not have. Pin the image to the release of the library your project uses, with a `<minor>` tag such as `0.25` or an exact `<version>` such as `0.25.0`, rather than `latest`.

The images start at Thrift 0.25.0, so there is no image for an older library release. Only the newest release is built: once a new minor release such as 0.26.0 is out, the `0.25` tags are no longer rebuilt and get no more distro security fixes.

## Checking for breaking changes

`--audit <old file>` compares the IDL file given last with an older version of it and reports changes that break compatibility with existing clients or servers, such as a removed or retyped field, a new required field, a field that became required or stopped being required, a removed method or enum value, or a changed return type. A renamed field or a changed default value is printed as a warning. thrift exits with status 2 when it finds a breaking change, so the check works in CI. Here `api.thrift` includes `common.thrift`, and the old versions of both come from the main branch:

```sh
mkdir -p old
git archive origin/main api.thrift common.thrift | tar -xf - -C old
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" ghcr.io/randomcontainers/thrift \
  --audit old/api.thrift api.thrift
```

thrift looks for an included file next to the file that includes it, then in the `-I` directories, so copy every file the old version includes into `old/` with it. A missing include is only a warning: the old file's references to it then fail to resolve, and the audit stops with an error or reports breaking changes that are not there. When the included files are in a directory you pass with `-I`, give the old copy of that directory with `-Iold <dir>` and the current one with `-Inew <dir>`. The audit compares only the definitions in the two files it is given, so run it on each included file as well.

`--audit-allow-optional-field-removal` accepts removing fields declared `optional`; removing a field without a requiredness keyword still counts as a breaking change. `--audit-allow-required-field-to-default` accepts dropping the `required` keyword from a field.

## What is in the image

- `thrift` in `/usr/local/bin`, with every code generator of the release compiled in. For 0.25.0 these are `c_glib`, `cl`, `cpp`, `d`, `dart`, `delphi`, `erl`, `go`, `gv`, `haxe`, `html`, `java`, `javame`, `js`, `json`, `kotlin`, `lua`, `markdown`, `mmd`, `netstd`, `ocaml`, `perl`, `php`, `py`, `rb`, `rs`, `st`, `xml` and `xsd`. The build fails if one of the generators that upstream enables is missing.

Not included: the Thrift runtime libraries, and the compilers and interpreters of the target languages. The CMake options, the compiler, the Bison and flex versions and the generator list are in `/usr/local/share/randomcontainers/thrift/buildinfo`.

## Default or slim

Thrift's default image adds no other tools, so `latest` and `slim` are the same image, with the contents listed above. Use `latest` to run it and the `slim` tags as a base for your own image.

## Tags

`<version>` is a Thrift release such as `0.25.0`. `<minor>` and `<major>` are its shorter forms, `0.25` and `0`, and follow the newest release in that series. Thrift's major version is 0, so `<major>` currently moves with every release, like `latest`. Each row lists the default tag and its `slim` twin, which point to the same image.

| Tags | Base |
|---|---|
| `latest`, `slim` | Ubuntu |
| `<version>`, `<version>-slim` | Ubuntu |
| `<minor>`, `<minor>-slim`, `<major>`, `<major>-slim` | Ubuntu |
| `ubuntu`, `slim-ubuntu` | Ubuntu |
| `<version>-ubuntu`, `<version>-slim-ubuntu` | Ubuntu |
| `<minor>-ubuntu`, `<minor>-slim-ubuntu`, `<major>-ubuntu`, `<major>-slim-ubuntu` | Ubuntu |
| `<version>-ubuntu26.04`, `<version>-slim-ubuntu26.04` | Ubuntu 26.04 |
| `alpine`, `slim-alpine` | Alpine |
| `<version>-alpine`, `<version>-slim-alpine` | Alpine |
| `<minor>-alpine`, `<minor>-slim-alpine`, `<major>-alpine`, `<major>-slim-alpine` | Alpine |
| `<version>-alpine3.24`, `<version>-slim-alpine3.24` | Alpine 3.24 |

The images are currently built on Ubuntu 26.04 and Alpine 3.24. Tags without a distro version move to the next distro release when the project does; tags ending in `ubuntu26.04` or `alpine3.24` stay on that release and are no longer rebuilt once the project moves to the next one. Every tag of the current Thrift version, including the exact version, is rebuilt in place (see [Updates](#updates)), so pin a digest when you need the same bytes every time.

## Platforms

`linux/amd64` and `linux/arm64`, for both Ubuntu and Alpine. Both are compiled natively on GitHub-hosted runners, without emulation.

## Files and permissions

The working directory is `/work`. The image runs as UID 1000, and any other UID works too: `HOME` is then `/`, and caches go to `/cache`, which anyone can write to. How to get output files owned by you depends on how you run containers:

| Runtime | Flag |
|---|---|
| Docker on Linux (rootful), GitHub Actions | `--user "$(id -u):$(id -g)"` |
| Rootless Podman | `--userns=keep-id` |
| Rootless Docker | `--user 0:0` (root in the container is your user on the host) |
| Docker Desktop on macOS or Windows | none, file ownership is mapped for you |

thrift writes generated code into the mounted directory, so run it with the flag above.

## Extending the slim image

Use a `slim` tag as the base for your own image. `slim`, `slim-ubuntu` and `slim-alpine` move to each new Thrift release and are rebuilt when the base image changes; `<version>-slim-ubuntu` and `<version>-slim-alpine` stay on one release. An image can carry IDL files that several projects include, for use with `-I`:

```dockerfile
FROM ghcr.io/randomcontainers/thrift:0.25.0-slim-ubuntu@sha256:...
COPY idl/ /usr/local/share/idl/
```

Run it with `-I /usr/local/share/idl`. To install distro packages, switch to `USER root` and back to `USER 1000:1000` afterwards. The entrypoint is `["tini", "--", "thrift"]`; set your own `ENTRYPOINT` if your image runs something else. To pick up base image fixes, let Dependabot or Renovate update the digest in your `FROM` line.

Everything the image adds is under `/usr/local`. `/usr/local/share/randomcontainers/thrift/` holds the version, the source URLs, the build options, the license files and `runtime-deps`, the list of distro packages thrift needs at run time.

## Verifying

Each image has a build provenance attestation from this repository's GitHub Actions run, signed by the shared build workflow in `randomcontainers/ci`:

```sh
gh attestation verify oci://ghcr.io/randomcontainers/thrift:latest \
  --repo randomcontainers/thrift --signer-repo randomcontainers/ci
```

Each platform image also carries an SPDX SBOM that lists every distro package with its version:

```sh
docker buildx imagetools inspect ghcr.io/randomcontainers/thrift:latest --format '{{ json .SBOM }}'
```

Before compiling, the build checks the tarball against the SHA-256 recorded in `package.yml` and its signature against Jens Geyer's release signing key in `keys/thrift-release.gpg` (fingerprint `8CD8 7F18 6F06 E958 EFCA 963D 76BD 340F C4B7 5865`), which signed every release from 0.13.0 to 0.25.0. The key has no expiry date. CMake configures and compiles without network access, so nothing else is downloaded.

## Updates

The project checks the releases of [apache/thrift](https://github.com/apache/thrift/releases) every 15 minutes. A release is picked up once it is 24 hours old and its tarball and signature are on the Apache download site. The tarball is checked against the release's `thrift-<version>.tar.gz.sha256` file, the new version and the tarball's SHA-256 are committed to `package.yml`, and the images are rebuilt. A release signed with a key that is not in `keys/thrift-release.gpg` fails to build until the key is added. Only the newest release is built; tags of older versions stay as they were last built.

The images of the current version are also rebuilt when the Ubuntu or Alpine base image changes and at least every 7 days, so distro security fixes reach the current tags.

## Building

```sh
docker build -f Dockerfile.ubuntu --target slim \
  --build-arg VERSION=<version> \
  --build-arg SOURCE_SHA256=<sha256 from package.yml> \
  -t thrift:local .
```

Use `Dockerfile.alpine` for the Alpine image. `--build-arg JOBS=<n>` limits the number of parallel compile jobs.

## Licenses

Apache Thrift is licensed under the Apache License 2.0, so the image's license label is `Apache-2.0`. Thrift's `LICENSE` and `NOTICE` are in `/usr/local/share/randomcontainers/thrift/licenses/`. The compiler also contains a SHA-256 implementation by Brad Conte, which is in the public domain, and the Bootstrap 2.0.3 style sheet, under the Apache License 2.0, that the HTML generator writes into `style.css`; Thrift's `LICENSE` lists both. The scanner and parser are generated during the build by the distro's flex and Bison, and Bison's license exception allows its output to be used under any terms. The Ubuntu and Alpine packages in the image keep their own licenses.

The build applies no patches. `/usr/local/share/randomcontainers/thrift/source` lists the upstream download URLs of the tarball and its signature. Those hosts keep only the two newest releases; older ones are in the Apache archive at `https://archive.apache.org/dist/thrift/<version>/`.

The files in this repository are available under the MIT license, see [LICENSE](LICENSE).

## Requesting a tool

To suggest another tool, use the [Request a tool](https://github.com/randomcontainers/.github/issues/new?template=tool-request.yml) form.
