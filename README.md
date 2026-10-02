# jpegoptim

Container images with [jpegoptim](https://github.com/tjko/jpegoptim), which makes JPEG files smaller, either losslessly or by recompressing them to a maximum quality or file size. `latest` also includes ImageMagick, for resizing or converting images before they are optimized. jpegoptim is compiled from the signed release tarball against the libjpeg-turbo of Ubuntu or Alpine. The images are rebuilt when jpegoptim publishes a release and when the base image changes, for `linux/amd64` and `linux/arm64`.

This is an unofficial build, not affiliated with or endorsed by the jpegoptim project. Report problems with the image in this repository and problems with jpegoptim itself [upstream](https://github.com/tjko/jpegoptim/issues).

## Quick start

Optimize every JPEG in the current directory without changing its pixels, and strip the metadata:

```sh
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" \
  ghcr.io/randomcontainers/jpegoptim --strip-all *.jpg
```

jpegoptim replaces each file, and only when the result is smaller. To keep the originals, write the results to another directory with `--dest`, which must exist:

```sh
mkdir -p optimized
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" \
  ghcr.io/randomcontainers/jpegoptim --strip-all --dest=optimized *.jpg
```

Recompress photos saved above quality 85, or aim for a file size, in kilobytes or as a percentage of the original:

```sh
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" \
  ghcr.io/randomcontainers/jpegoptim --max=85 --strip-all *.jpg

docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" \
  ghcr.io/randomcontainers/jpegoptim --size=200 photo.jpg
```

With `--stdin` and `--stdout` no mount is needed. The report then goes to stderr:

```sh
docker run --rm -i ghcr.io/randomcontainers/jpegoptim --stdin --stdout --strip-all < photo.jpg > photo-small.jpg
```

`--noaction` prints what each file would save without writing anything, and `--csv` prints the results as comma-separated values. A file that libjpeg can only partly decode, such as a truncated download, is rewritten too, with a warning; `--nofix` skips such files. Run the image without arguments to list every option.

`--strip-all` also removes the Exif orientation tag and ICC color profiles. A phone photo that depends on its orientation tag is then shown rotated, and a photo tagged with a wide-gamut profile such as Display P3 is shown with the wrong colors. Keep them by adding `--keep-exif` and `--keep-icc` after `--strip-all`. Options are read from left to right, and `--strip-all` cancels any `--keep-*` option given before it:

```sh
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" \
  ghcr.io/randomcontainers/jpegoptim --strip-all --keep-exif --keep-icc *.jpg
```

`--keep-exif` keeps all of the Exif data, including any GPS position. To drop it, use the default image instead: ImageMagick rotates the pixels, so the orientation tag is no longer needed, and only the color profile is kept:

```sh
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" --entrypoint sh \
  ghcr.io/randomcontainers/jpegoptim \
  -c 'magick photo.jpg -auto-orient -resize 1600x1600 web.jpg && jpegoptim --max=85 --strip-all --keep-icc web.jpg'
```

Data after the end of the image, such as the video part of a motion photo, is dropped unless you add `--save-extra`.

## What is in the image

| | slim | default |
|---|---|---|
| jpegoptim, linked against the distro's libjpeg-turbo | yes | yes |
| ImageMagick, to resize, rotate or convert images before optimizing them | no | yes |

ImageMagick is the `slim` build from [randomcontainers/imagemagick](https://github.com/randomcontainers/imagemagick). It has no Ghostscript, so it cannot read PDF or PostScript files. The entrypoint runs `jpegoptim` under `tini`; run ImageMagick with `--entrypoint magick`, or from a shell as in the example above.

Not included: arithmetic coding (`--all-arith`), which upstream marks as experimental. jpegoptim is linked against libjpeg-turbo, not mozjpeg, so the DC scan optimization it turns on with mozjpeg is not used. The configure line and the output of `jpegoptim --version` are in `/usr/local/share/randomcontainers/jpegoptim/buildinfo`.

## Default or slim

The default image (`latest`) also has ImageMagick, for the usual resize-then-optimize step and for converting other formats to JPEG. `slim` has only jpegoptim and the libraries it needs. Use it when you only optimize existing JPEG files, or as the base for your own image.

The default image is also published as `ghcr.io/randomcontainers/jpegoptim-imagemagick`, built in the [jpegoptim-imagemagick](https://github.com/randomcontainers/jpegoptim-imagemagick) repository with the same contents and a different digest.

## Tags

`<version>` is a jpegoptim release such as `1.5.6`. `<minor>` and `<major>` are its shorter forms, `1.5` and `1`, and follow the newest release in that series.

| Default (with ImageMagick) | Slim | Base |
|---|---|---|
| `latest`, `<version>`, `<minor>`, `<major>` | `slim`, `<version>-slim`, `<minor>-slim`, `<major>-slim` | Ubuntu |
| `ubuntu`, `<version>-ubuntu`, `<minor>-ubuntu`, `<major>-ubuntu` | `slim-ubuntu`, `<version>-slim-ubuntu`, `<minor>-slim-ubuntu`, `<major>-slim-ubuntu` | Ubuntu |
| `<version>-ubuntu26.04` | `<version>-slim-ubuntu26.04` | Ubuntu 26.04 |
| `alpine`, `<version>-alpine`, `<minor>-alpine`, `<major>-alpine` | `slim-alpine`, `<version>-slim-alpine`, `<minor>-slim-alpine`, `<major>-slim-alpine` | Alpine |
| `<version>-alpine3.24` | `<version>-slim-alpine3.24` | Alpine 3.24 |

The images are currently built on Ubuntu 26.04 and Alpine 3.24. Tags without a distro version move to the next distro release when the project does; tags ending in `ubuntu26.04` or `alpine3.24` stay on that release and are no longer rebuilt once the project moves to the next one. Every tag of the current jpegoptim version, including the exact version, is rebuilt in place (see [Updates](#updates)), so pin a digest when you need the same bytes every time.

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

jpegoptim writes each result to a temporary file in the same directory and renames it over the original, so the directory has to be writable; the file itself only has to be readable. The new file gets the original's permissions and group, and `--preserve` also keeps its modification time. With `--preserve-perms`, jpegoptim overwrites the original file in place instead, which needs write access to the file.

## Extending the slim image

Use a `slim` tag as the base for your own image. `slim`, `slim-ubuntu` and `slim-alpine` follow new jpegoptim releases and are rebuilt when the distro changes. The packages jpegoptim needs are listed in `/usr/local/share/randomcontainers/jpegoptim/runtime-deps`. Switch to root to install more, then back:

```dockerfile
FROM ghcr.io/randomcontainers/jpegoptim:slim-ubuntu@sha256:...
USER root
RUN apt-get update \
 && apt-get install -y --no-install-recommends optipng \
 && rm -rf /var/lib/apt/lists/*
USER 1000:1000
```

On Alpine, use `apk add --no-cache optipng`. The entrypoint is `["tini", "--", "jpegoptim"]`; set your own `ENTRYPOINT` if your image runs a script. To pick up new jpegoptim releases and base image fixes, let Dependabot or Renovate update the digest in your `FROM` line.

## Verifying

Each image has a build provenance attestation from this repository's GitHub Actions run, signed by the shared build workflow in `randomcontainers/ci`:

```sh
gh attestation verify oci://ghcr.io/randomcontainers/jpegoptim:latest \
  --repo randomcontainers/jpegoptim --signer-repo randomcontainers/ci
```

Images from `ghcr.io/randomcontainers/jpegoptim-imagemagick` are built in that repository, so verify them with `--repo randomcontainers/jpegoptim-imagemagick` and the same `--signer-repo`.

Each platform image also carries an SPDX SBOM that lists every distro package with its version:

```sh
docker buildx imagetools inspect ghcr.io/randomcontainers/jpegoptim:latest --format '{{ json .SBOM }}'
```

Before compiling, the build checks the tarball against the SHA-256 recorded in `package.yml` and its signature against the key of jpegoptim's author, Timo Kokkonen, in `keys/tjko.gpg` (fingerprint `9168 CD7A 2F0A AB06 79D0 9BBA 9D08 A80C ED90 8D6A`). That key is a 1024-bit DSA key and signs with SHA-1, so the SHA-256, which is checked against the digest GitHub records for the release asset, is the stronger of the two checks.

## Updates

The project checks the [tjko/jpegoptim](https://github.com/tjko/jpegoptim/releases) releases every 15 minutes. A release is picked up once it is 24 hours old and has the source tarball and its signature attached. The tarball is checked against the digest GitHub records for it, the new version and the tarball's SHA-256 are committed to `package.yml`, and the images are rebuilt. Only the newest release is built; tags of older versions stay as they were last built.

The images of the current version are also rebuilt when the Ubuntu or Alpine base image changes, the default ones when a new ImageMagick image is published, and all of them at least every 7 days, so distro security fixes reach the current tags.

## Building

```sh
docker build -f Dockerfile.ubuntu --target slim \
  --build-arg VERSION=<version> \
  --build-arg SOURCE_SHA256=<sha256 from package.yml> \
  -t jpegoptim:local .
```

Use `Dockerfile.alpine` for the Alpine image. `--build-arg JOBS=<n>` limits the number of parallel compile jobs. The default image is generated from the `combos` entry in `package.yml` by [randomcontainers/ci](https://github.com/randomcontainers/ci).

## Licenses

jpegoptim is licensed under the GNU General Public License, version 3 or later (GPL-3.0-or-later). Its `COPYRIGHT` and `LICENSE` files are in `/usr/local/share/randomcontainers/jpegoptim/licenses/`. It links libjpeg-turbo from Ubuntu or Alpine, which keeps its own license. This software is based in part on the work of the Independent JPEG Group.

The corresponding source for each image:

- jpegoptim: every version has a GitHub release in this repository, named `v<version>`, with the exact `jpegoptim-<version>.tar.gz` that was compiled and its signature. The build applies no patches. `/usr/local/share/randomcontainers/jpegoptim/source` lists that release and the upstream download URLs.
- Build scripts: this repository at the commit in the image's `org.opencontainers.image.revision` label. The Dockerfiles hold every configure flag.
- Ubuntu packages: the source packages on [Launchpad](https://launchpad.net/ubuntu) for the versions listed in the SBOM. `apt-get source <package>=<version>` fetches a version that is still in the Ubuntu archive.
- Alpine packages: Alpine has no source packages. For the versions listed in the SBOM, the source is the APKBUILD and patches in [aports](https://gitlab.alpinelinux.org/alpine/aports/-/tree/3.24-stable), branch `3.24-stable`, and the archives on [distfiles.alpinelinux.org](https://distfiles.alpinelinux.org/distfiles/v3.24/).

`latest` also contains ImageMagick, licensed under the ImageMagick License, with libheif and libde265, which are licensed under LGPL-3.0-or-later. Their license files and the source they are built from are described in the [imagemagick repository](https://github.com/randomcontainers/imagemagick#licenses).

The files in this repository are available under the MIT license, see [LICENSE](LICENSE).

## Requesting a tool

To suggest another tool, use the [Request a tool](https://github.com/randomcontainers/.github/issues/new?template=tool-request.yml) form.
