# How to make a new release (maintainer info)

## Release schedule

The xPack clang release schedule generally follows the original GNU
[releases](https://github.com/llvm/llvm-project/releases), but with a
several weeks filter, which means that releases that are shortly
overwritten are skipped. Also initial x.y.0 releases are skipped.

## Prepare the build

Before starting the build, perform some checks and tweaks.

### Check Git

- switch to the `xpack-develop` branch
- if needed, merge the `xpack` branch

### Increase the version

Determine the version (like `12.0.1`) and update the `scripts/VERSION`
file; the format is `12.0.1-1`. The fourth number is the xPack release number
of this version. A fifth number will be added when publishing
the package on the `npm` server.

### Fix possible open issues

Check GitHub issues and pull requests:

- <https://github.com/xpack-dev-tools/clang-xpack/issues/>

and fix them; assign them to a milestone (like `12.0.1-1`).

### Check `README.md`

Normally `README.md` should not need changes, but better check.
Information related to the new version should not be included here,
but in the web release files.

### Update version in `README` files

- update version in `README-RELEASE.md`
- update version in `README-BUILD.md`
- update version in `README.md`

## Update `CHANGELOG.md`

- open the `CHANGELOG.md` file
- check if all previous fixed issues are in
- add a new entry like _v12.0.1-1 prepared_
- commit with a message like _prepare v12.0.1-1_

Note: if you missed to update the `CHANGELOG.md` before starting the build,
edit the file and rerun the build, it should take only a few minutes to
recreate the archives with the correct file.

### Update the version specific code

- open the `common-versions-source.sh` file
- add a new `if` with the new version before the existing code

### Update helper

With Sourcetree, go to the helper repo and update to the latest master commit.

## Build

### Development run the build scripts

Before the real build, run a test build on the development machine (`wks`):

```sh
sudo rm -rf ~/Work/clang-*

caffeinate bash ~/Downloads/clang-xpack.git/scripts/build.sh --develop --without-pdf --without-html --disable-tests --osx
```

Similarly on the Intel Linux:

```sh
bash ~/Downloads/clang-xpack.git/scripts/build.sh --develop --without-pdf --without-html --disable-tests --linux64
bash ~/Downloads/clang-xpack.git/scripts/build.sh --develop --without-pdf --without-html --disable-tests --linux32

bash ~/Downloads/clang-xpack.git/scripts/build.sh --develop --without-pdf --without-html --disable-tests --win64
bash ~/Downloads/clang-xpack.git/scripts/build.sh --develop --without-pdf --without-html --disable-tests --win32
```

And on the Arm Linux:

```sh
bash ~/Downloads/clang-xpack.git/scripts/build.sh --develop --without-pdf --without-html --disable-tests --arm64
bash ~/Downloads/clang-xpack.git/scripts/build.sh --develop --without-pdf --without-html --disable-tests --arm32
```

Work on the scripts until all 4 platforms pass the build.

## Push the build script

In this Git repo:

- push the `xpack-develop` branch to GitHub
- possibly push the helper project too

From here it'll be later cloned on the production machines.

### Run the build scripts

On the macOS machine (`xbbm13`) open ssh sessions to both Linux machines
(`xbbi` and `xbba`):

```sh
caffeinate ssh xbbi

caffeinate ssh xbba
```

Note: If this is a virtual machine, be sure the host will not go to sleep
(run `caffeinate sh` in a terminal).

On all machines, clone the `xpack-develop` branch and remove previous builds

```sh
rm -rf ~/Downloads/clang-xpack.git; \
git clone \
  --recurse-submodules \
  --branch xpack-develop \
  https://github.com/xpack-dev-tools/clang-xpack.git \
  ~/Downloads/clang-xpack.git

sudo rm -rf ~/Work/clang-*
```

Empty trash.

On the macOS 10.13 machine (`xbbm13`):

```sh
caffeinate bash ~/Downloads/clang-xpack.git/scripts/build.sh --osx
```

A typical run takes about 70 minutes.

On `xbbi`:

```sh
bash ~/Downloads/clang-xpack.git/scripts/build.sh --all

bash ~/Downloads/clang-xpack.git/scripts/build.sh --linux64
bash ~/Downloads/clang-xpack.git/scripts/build.sh --win64
bash ~/Downloads/clang-xpack.git/scripts/build.sh --linux32
bash ~/Downloads/clang-xpack.git/scripts/build.sh --win32
```

A typical run on the Intel machine takes about 350 minutes
(almost 4 hours).

On `xbba`:

```sh
bash ~/Downloads/clang-xpack.git/scripts/build.sh --all

bash ~/Downloads/clang-xpack.git/scripts/build.sh --arm64
bash ~/Downloads/clang-xpack.git/scripts/build.sh --arm32
```

A typical run on the Arm machine takes about 755 minutes
(almost 13 hours).

### Clean the destination folder

On the development machine (`wks`) clear the folder where binaries from all
build machines will be collected.

```sh
rm -f ~/Downloads/xpack-binaries/clang/*
```

### Copy the binaries to the development machine

On all three machines:

```sh
(cd ~/Work/clang-*/deploy; scp * ilg@wks:Downloads/xpack-binaries/clang)
```

## Run the pre-release native tests

Publish the archives on the
[pre-release](https://github.com/xpack-dev-tools/pre-releases/releases/tag/test)
project, and run the native tests on all platforms:

```sh
rm -rf ~/Downloads/clang-xpack.git; \
git clone \
  --recurse-submodules \
  --branch xpack-develop \
  https://github.com/xpack-dev-tools/clang-xpack.git  \
  ~/Downloads/clang-xpack.git

rm -rf ~/Work/cache/xpack-clang-*

bash ~/Downloads/clang-xpack.git/tests/scripts/native-test.sh \
  "https://github.com/xpack-dev-tools/pre-releases/releases/download/test/"
```

For early experimental releases, use:

```sh
bash ~/Downloads/clang-xpack.git/tests/scripts/native-test.sh \
  "https://github.com/xpack-dev-tools/pre-releases/releases/download/experimental/"
```

## Create a new GitHub pre-release

- in `CHANGELOG.md`, add release date
- commit and push the `xpack-develop` branch
- go to the GitHub [releases](https://github.com/xpack-dev-tools/clang-xpack/releases/) page
- click **Draft a new release**, in the `xpack-develop` branch
- name the tag like **v12.0.1-1** (mind the dash in the middle!)
- name the release like **xPack clang v12.0.1-1**
(mind the dash)
- as description, use:

```markdown
![Github Releases (by Release)](https://img.shields.io/github/downloads/xpack-dev-tools/clang-xpack/v12.0.1-1/total.svg)

Version v12.0.1-1 is a new release of the **xPack clang** package, following the GCC release.

_At this moment these binaries are provided for tests only!_
```

- **attach binaries** and SHA (drag and drop from the
  `~/Downloads/xpack-binaries/*` folder will do it)
- **enable** the **pre-release** button
- click the **Publish Release** button

Note: at this moment the system should send a notification to all clients
watching this project.

## Run the native tests

Run the native tests on all platforms:

```sh
rm -rf ~/Downloads/clang-xpack.git; \
git clone --recurse-submodules -b xpack-develop \
  https://github.com/xpack-dev-tools/clang-xpack.git  \
  ~/Downloads/clang-xpack.git

rm -rf ~/Work/cache/xpack-clang-*

bash ~/Downloads/clang-xpack.git/tests/scripts/native-test.sh \
  "https://github.com/xpack-dev-tools/clang-xpack/releases/download/v12.0.1-1/"
```

## Run the release CI tests

Using the scripts in `tests/scripts/`, start:

TODO:

The test results are available from:

- TODO

For more details, see `tests/scripts/README.md`.

## Prepare a new blog post

In the `xpack/web-jekyll` GitHub repo:

- select the `develop` branch
- add a new file to `_posts/clang/releases`
- name the file like `2021-07-12-clang-v12-0-1-1-released.md`
- name the post like: **xPack clang v12.0.1-1 released**
- as `download_url` use the tagged URL like `https://github.com/xpack-dev-tools/clang-xpack/releases/tag/v12.0.1-1/`
- update the `date:` field with the current date
- update the Travis URLs using the actual test pages
- update the SHA sums via copy/paste from the original build machines
(it is very important to use the originals!)

If any, refer to closed
[issues](https://github.com/xpack-dev-tools/clang-xpack/issues/)
as:

- **[Issue:\[#1\]\(...\)]**.

### Update the SHA sums

On the development machine (`wks`):

```sh
cat ~/Downloads/xpack-binaries/clang/*.sha
```

Copy/paste the build report at the end of the post as:

```console
## Checksums
The SHA-256 hashes for the files are:

0a2a2550ec99b908c92811f8dbfde200956a22ab3d9af1c92ce9926bf8feddf9
xpack-clang-12.0.1-1-darwin-x64.tar.gz

254588cbcd685748598dd7bbfaf89280ab719bfcd4dabeb0269fdb97a52b9d7a
xpack-clang-12.0.1-1-linux-arm.tar.gz

10e30128d626f9640c0d585e6b65ac943de59fbdce5550386add015bcce408fa
xpack-clang-12.0.1-1-linux-arm64.tar.gz

50f2e399382c29f8cdc9c77948e1382dfd5db20c2cb25c5980cb29774962483f
xpack-clang-12.0.1-1-linux-ia32.tar.gz

9b147443780b7f825eec333857ac7ff9e9e9151fd17c8b7ce2a1ecb6e3767fd6
xpack-clang-12.0.1-1-linux-x64.tar.gz

501366492cd73b06fca98b8283f65b53833622995c6e44760eda8f4483648525
xpack-clang-12.0.1-1-win32-ia32.zip

dffc858d64be5539410aa6d3f3515c6de751cd295c99217091f5ccec79cabf39
xpack-clang-12.0.1-1-win32-x64.zip
```

## Update the preview Web

- commit the `develop` branch of `xpack/web-jekyll` GitHub repo;
  use a message like **xPack clang v12.0.1-1 released**
- push
- wait for the GitHub Pages build to complete
- the preview web is <https://xpack.github.io/web-preview/news/>

## Update package.json binaries

- select the `xpack-develop` branch
- run `xpm-dev binaries-update`

```sh
xpm-dev binaries-update \
  -C "${HOME}/Downloads/clang-xpack.git" \
  '12.0.1-1' \
  "${HOME}/Downloads/xpack-binaries/clang"
```

- open the GitHub [releases](https://github.com/xpack-dev-tools/clang-xpack/releases/)
  page and select the latest release
- check the download counter, it should match the number of tests
- open the `package.json` file
- check the `baseUrl:` it should match the file URLs (including the tag/version);
  no terminating `/` is required
- from the release, check the SHA & file names
- compare the SHA sums with those shown by `cat *.sha`
- check the executable names
- commit all changes, use a message like
  `package.json: update urls for 12.0.1-1.1 release` (without `v`)

## Publish on the npmjs.com server

- select the `xpack-develop` branch
- check the latest commits `npm run git-log`
- update `CHANGELOG.md`; commit with a message like
  _CHANGELOG: publish npm v12.0.1-1.1_
- `npm pack` and check the content of the archive, which should list
  only the `package.json`, the `README.md`, `LICENSE` and `CHANGELOG.md`;
  possibly adjust `.npmignore`
- `npm version 12.0.1-1.1`; the first 5 numbers are the same as the
  GitHub release; the sixth number is the npm specific version
- push the `xpack-develop` branch to GitHub
- push tags with `git push origin --tags`
- `npm publish --tag next` (use `--access public` when publishing for
  the first time); for updates use `npm publish --tag update`

After a few moments the version will be visible at:

- <https://www.npmjs.com/package/@xpack-dev-tools/clang?activeTab=versions>

## Test if the npm binaries can be installed with xpm

Run the `tests/scripts/trigger-travis-xpm-install.sh` script, this
will install the package on Intel Linux 64-bit, macOS and Windows 64-bit.

The test results are available from:

- <https://travis-ci.com/github/xpack-dev-tools/clang-xpack/>

For 32-bit Windows, 32-bit Intel GNU/Linux and 32-bit Arm, install manually.

```sh
xpm install --global @xpack-dev-tools/clang@next
```

## Test the npm binaries

Install the binaries on all platforms.

```sh
xpm install --global @xpack-dev-tools/clang@next
```

On GNU/Linux systems, including Raspberry Pi, use the following commands:

```sh
~/.local/xPacks/@xpack-dev-tools/clang/12.0.1-1.1/.content/bin/clang --version

clang (xPack clang 64-bit) 12.0.1
```

On macOS, use:

```sh
~/Library/xPacks/@xpack-dev-tools/clang/12.0.1-1.1/.content/bin/clang --version

clang (xPack clang 64-bit) 12.0.1
```

On Windows use:

```doscon
%USERPROFILE%\AppData\Roaming\xPacks\@xpack-dev-tools\clang\12.0.1-1.1\.content\bin\clang --version

clang.exe (xPack MinGW-w64 GCC 64-bit) 12.0.1
```

## Update the repo

- merge `xpack-develop` into `xpack`
- push

## Tag the npm package as `latest`

When the release is considered stable, promote it as `latest`:

- `npm dist-tag ls @xpack-dev-tools/clang`
- `npm dist-tag add @xpack-dev-tools/clang@12.0.1-1.1 latest`
- `npm dist-tag ls @xpack-dev-tools/clang`

## Update the Web

- in the `master` branch, merge the `develop` branch
- wait for the GitHub Pages build to complete
- the result is in <https://xpack.github.io/news/>
- remember the post URL, since it must be updated in the release page

## Create the final GitHub release

- go to the GitHub [releases](https://github.com/xpack-dev-tools/clang-xpack/releases/) page
- check the download counter, it should match the number of tests
- add a link to the Web page `[Continue reading »]()`; use an same blog URL
- **disable** the **pre-release** button
- click the **Update Release** button

## Share on Twitter

- in a separate browser windows, open [TweetDeck](https://tweetdeck.twitter.com/)
- using the `@xpack_project` account
- paste the release name like **xPack clang v12.0.1-1 released**
- paste the link to the Web page
  [release](https://xpack.github.io/clang/releases/)
- click the **Tweet** button

## Remove pre-release binaries

- got to <https://github.com/xpack-dev-tools/pre-releases/releases/tag/test>
- remove the test binaries
