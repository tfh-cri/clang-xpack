# How to make a new release (maintainer info)

## Release schedule

The xPack LLVM clang release schedule generally follows the original LLVM
[releases](https://github.com/llvm/llvm-project/releases/), but aims to
get the latest release after a new major release is published.

## Prepare the build

Before starting the build, perform some checks and tweaks.

### Check Git

In the `xpack-dev-tools/clang-xpack` Git repo:

- switch to the `xpack-develop` branch
- if needed, merge the `xpack` branch

No need to add a tag here, it'll be added when the release is created.

### Check the latest upstream release

Check the LLVM GitHub [Releases](https://github.com/llvm/llvm-project/releases)
and compare the the xPack [Releases](https://github.com/xpack-dev-tools/clang-xpack/releases/).
Find the latest release that seems stable; usually skip X.Y.0 releases,
since they are usually followed by a X.Y.1 release in several month.

### Increase the version

Determine the version (like `14.0.6`) and update the `scripts/VERSION`
file; the format is `14.0.6-1`. The fourth number is the xPack release number
of this version. A fifth number will be added when publishing
the package on the `npm` server.

### Fix possible open issues

Check GitHub issues and pull requests:

- <https://github.com/xpack-dev-tools/clang-xpack/issues/>

and fix them; assign them to a milestone (like `14.0.6-1`).

### Check `README.md`

Normally `README.md` should not need changes, but better check.
Information related to the new version should not be included here,
but in the web release files.

### Update version in `README` files

- update version in `README-RELEASE.md`
- update version in `README-BUILD.md`
- update version in `README.md`

### Update `CHANGELOG.md`

- open the `CHANGELOG.md` file
- check if all previous fixed issues are in
- add a new entry like _- v14.0.6-1 prepared_
- commit with a message like _prepare v14.0.6-1_

Note: if you missed to update the `CHANGELOG.md` before starting the build,
edit the file and rerun the build, it should take only a few minutes to
recreate the archives with the correct file.

### Merge upstream repo & prepare patch

To keep the development repository fork in sync with the upstream LLVM
repository, in the `xpack-dev-tools/llvm-project` Git repo:

- fetch `upstream`
- checkout the `llvmorg-14.0.6` tag in detached state HEAD
- create a branch like `v14.0.6-xpack`
- cherry pick the commit to _clang: add /Library/... to headers search path_ from a previous release;
  enable commit immediately
- push branch to `origin`
- add a `v14.0.6-1-xpack` tag; enable push to origin
- select the commit with the patch
- save as patch
- move to `patches`
- rename `llvm-14.0.6.patch.diff`

Note: currently the patch is required to fix the CLT library path.

### Update the version specific code

- open the `common-versions-source.sh` file
- add a new `if` with the new version before the existing code

### Update helper

With a git client, go to the helper repo and update to the latest master commit.

## Build

### Development run the build scripts

Before the real build, run a test build on the development machine (`wksi`)
or the production machines (`xbbma`, `xbbmi`):

```sh
sudo rm -rf ~/Work/clang-*-*

caffeinate bash ${HOME}/Work/clang-xpack.git/scripts/helper/build.sh --develop --macos
```

Similarly on the Intel Linux (`xbbli`):

```sh
bash ${HOME}/Work/clang-xpack.git/scripts/helper/build.sh --develop --linux64

bash ${HOME}/Work/clang-xpack.git/scripts/helper/build.sh --develop --win64
```

... the Arm Linux 64-bit (`xbbla64`):

```sh
bash ${HOME}/Work/clang-xpack.git/scripts/helper/build.sh --develop --arm64
```

... and on the Arm Linux 32-bit (`xbbla32`):

```sh
bash ${HOME}/Work/clang-xpack.git/scripts/helper/build.sh --develop --arm32
```

Work on the scripts until all platforms pass the build.

## Push the build scripts

In this Git repo:

- push the `xpack-develop` branch to GitHub
- possibly push the helper project too

From here it'll be later cloned on the production machines.

### Run the build scripts

- on the macOS machine (`xbbmi`) open ssh sessions to the build
machines (`xbbma`, `xbbli`, `xbbla64` and `xbbla32`):

```sh
caffeinate ssh xbbma
caffeinate ssh xbbli
caffeinate ssh xbbla64
caffeinate ssh xbbla32
```

Start the runner on all machines:

```sh
~/actions-runners/xpack-dev-tools/run.sh &
```

Check that both the project Git and the submodule are pushed to GitHub.

To trigger the GitHub Actions build, use the xPack action:

- `trigger-workflow-build-xbbli`
- `trigger-workflow-build-xbbla64`
- `trigger-workflow-build-xbbla32`
- `trigger-workflow-build-xbbmi`
- `trigger-workflow-build-xbbma`

This is equivalent to:

```sh
bash ${HOME}/Work/clang-xpack.git/scripts/helper/trigger-workflow-build.sh --machine xbbli
bash ${HOME}/Work/clang-xpack.git/scripts/helper/trigger-workflow-build.sh --machine xbbla64
bash ${HOME}/Work/clang-xpack.git/scripts/helper/trigger-workflow-build.sh --machine xbbla32
bash ${HOME}/Work/clang-xpack.git/scripts/helper/trigger-workflow-build.sh --machine xbbmi
bash ${HOME}/Work/clang-xpack.git/scripts/helper/trigger-workflow-build.sh --machine xbbma
```

These scripts require the `GITHUB_API_DISPATCH_TOKEN` variable to be present
in the environment, and the organization `PUBLISH_TOKEN` to be visible in the
Settings → Action →
[Secrets](https://github.com/xpack-dev-tools/clang-xpack/settings/secrets/actions)
page.

This command uses the `xpack-develop` branch of this repo.

The builds take about 13 hours to complete:

- `xbbmi`: 1h25m (vm)
- `xbbma`: 31 min
- `xbbli`: 3h20m (Windows included)
- `xbbla64`: 9h37m (!)
- `xbbla32`: 8h

The workflow result and logs are available from the
[Actions](https://github.com/xpack-dev-tools/clang-xpack/actions/) page.

The resulting binaries are available for testing from
[pre-releases/test](https://github.com/xpack-dev-tools/pre-releases/releases/tag/test/).

## Testing

### CI tests

The automation is provided by GitHub Actions.

On the macOS machine (`xbbmi`) open a ssh sessions to the Arm/Linux
test machine `xbbla`:

```sh
caffeinate ssh xbbla
```

Start both runners (to allow the 32/64-bit tests to run in parallel):

```sh
~/actions-runners/xpack-dev-tools/1/run.sh &
~/actions-runners/xpack-dev-tools/2/run.sh &
```

To trigger the GitHub Actions tests, use the xPack actions:

- `trigger-workflow-test-prime`
- `trigger-workflow-test-docker-linux-intel`
- `trigger-workflow-test-docker-linux-arm`

These are equivalent to:

```sh
bash ${HOME}/Work/clang-xpack.git/scripts/helper/tests/trigger-workflow-test-prime.sh
bash ${HOME}/Work/clang-xpack.git/scripts/helper/tests/trigger-workflow-test-docker-linux-intel.sh
bash ${HOME}/Work/clang-xpack.git/scripts/helper/tests/trigger-workflow-test-docker-linux-arm.sh
```

These scripts require the `GITHUB_API_DISPATCH_TOKEN` variable to be present
in the environment.

These actions use the `xpack-develop` branch of this repo and the
[pre-releases/test](https://github.com/xpack-dev-tools/pre-releases/releases/tag/test/)
binaries.

The tests results are available from the
[Actions](https://github.com/xpack-dev-tools/clang-xpack/actions/) page.

Since GitHub Actions provides a single version of macOS, the
multi-version macOS tests run on Travis.

To trigger the Travis test, use the xPack action:

- `trigger-travis-macos`

This is equivalent to:

```sh
bash ${HOME}/Work/clang-xpack.git/scripts/helper/tests/trigger-travis-macos.sh
```

This script requires the `TRAVIS_COM_TOKEN` variable to be present
in the environment.

The test results are available from
[Travis CI](https://app.travis-ci.com/github/xpack-dev-tools/clang-xpack/builds/).

### Manual tests

Install the binaries on all platforms.

On GNU/Linux and macOS systems, use:

```sh
.../xpack-clang-14.0.6-1/bin/clang --version
xPack x86_64 clang version 14.0.6
```

On Windows use:

```dos
...\xpack-clang-14.0.6-1\bin\clang --version
xPack x86_64 clang version 14.0.6
```

## Create a new GitHub pre-release draft

- in `CHANGELOG.md`, add the release date and a message like _- v14.0.6-1 released_
- commit and push the `xpack-develop` branch
- run the xPack action `trigger-workflow-publish-release`

The workflow result and logs are available from the
[Actions](https://github.com/xpack-dev-tools/clang-xpack/actions/) page.

The result is a
[draft pre-release](https://github.com/xpack-dev-tools/clang-xpack/releases/)
tagged like **v14.0.6-1** (mind the dash in the middle!) and
named like **xPack LLVM clang v14.0.6-1** (mind the dash),
with all binaries attached.

- edit the draft and attach it to the `xpack-develop` branch (important!)
- save the draft (do **not** publish yet!)

## Prepare a new blog post

Run the xPack action `generate-jekyll-post`; this will leave a file
on the Desktop.

In the `xpack/web-jekyll` GitHub repo:

- select the `develop` branch
- copy the new file to `_posts/releases/clang`

If any, refer to closed
[issues](https://github.com/xpack-dev-tools/clang-xpack/issues/).

## Update the preview Web

- commit the `develop` branch of `xpack/web-jekyll` GitHub repo;
  use a message like **xPack LLVM clang v14.0.6-1 released**
- push to GitHub
- wait for the GitHub Pages build to complete
- the preview web is <https://xpack.github.io/web-preview/news/>

## Create the pre-release

- go to the GitHub [Releases](https://github.com/xpack-dev-tools/clang-xpack/releases/) page
- perform the final edits and check if everything is fine
- temporarily fill in the _Continue Reading »_ with the URL of the
  web-preview release
- **keep the pre-release button enabled**
- do not enable Discussions yet
- publish the release

Note: at this moment the system should send a notification to all clients
watching this project.

## Update the README-BUILD listings and examples

- check and possibly update the `ls -l` output
- check and possibly update the output of the `--version` runs
- check and possibly update the output of `tree -L 2`
- commit changes

## Check the list of links

- open the `package.json` file
- check if the links in the `bin` property cover the actual binaries
- if necessary, also check on Windows

## Update package.json binaries

- select the `xpack-develop` branch
- run the xPack action `update-package-binaries`
- open the `package.json` file
- check the `baseUrl:` it should match the file URLs (including the tag/version);
  no terminating `/` is required
- from the release, check the SHA & file names
- compare the SHA sums with those shown by `cat *.sha`
- check the executable names
- commit all changes, use a message like
  `package.json: update urls for 14.0.6-1.1 release` (without `v`)

## Publish on the npmjs.com server

- select the `xpack-develop` branch
- check the latest commits `npm run git-log`
- update `CHANGELOG.md`, add a line like _- v14.0.6-1.1 published on npmjs.com_
- commit with a message like _CHANGELOG: publish npm v14.0.6-1.1_
- `npm pack` and check the content of the archive, which should list
  only the `package.json`, the `README.md`, `LICENSE` and `CHANGELOG.md`;
  possibly adjust `.npmignore`
- `npm version 14.0.6-1.1`; the first 5 numbers are the same as the
  GitHub release; the sixth number is the npm specific version
- the commits and the tag should have been pushed by the `postversion` script;
  if not, push them with `git push origin --tags`
- `npm publish --tag next` (use `--access public` when publishing for
  the first time)

After a few moments the version will be visible at:

- <https://www.npmjs.com/package/@xpack-dev-tools/clang?activeTab=versions>

## Test if the binaries can be installed with xpm

Run the xPack action `trigger-workflow-test-xpm`, this
will install the package via `xpm install` on all supported platforms.

The tests results are available from the
[Actions](https://github.com/xpack-dev-tools/clang-xpack/actions/) page.

## Update the repo

- merge `xpack-develop` into `xpack`
- push to GitHub

## Tag the npm package as `latest`

When the release is considered stable, promote it as `latest`:

- `npm dist-tag ls @xpack-dev-tools/clang`
- `npm dist-tag add @xpack-dev-tools/clang@14.0.6-1.1 latest`
- `npm dist-tag ls @xpack-dev-tools/clang`

In case the previous version is not functional and needs to be unpublished:

- `npm unpublish @xpack-dev-tools/clang@14.0.6-1.X`

## Update the Web

- in the `master` branch, merge the `develop` branch
- wait for the GitHub Pages build to complete
- the result is in <https://xpack.github.io/news/>
- remember the post URL, since it must be updated in the release page

## Create the final GitHub release

- go to the GitHub [Releases](https://github.com/xpack-dev-tools/clang-xpack/releases/) page
- check the download counter, it should match the number of tests
- add a link to the Web page `[Continue reading »]()`; use an same blog URL
- remove the _tests only_ notice
- **disable** the **pre-release** button
- click the **Update Release** button

## Share on Twitter

- in a separate browser windows, open [TweetDeck](https://tweetdeck.twitter.com/)
- using the `@xpack_project` account
- paste the release name like **xPack LLVM clang v14.0.6-1 released**
- paste the link to the Web page
  [release](https://xpack.github.io/clang/releases/)
- click the **Tweet** button

## Remove pre-release binaries

- go to <https://github.com/xpack-dev-tools/pre-releases/releases/tag/test/>
- remove the test binaries

## Clean the work area

Run the xPack action `trigger-workflow-deep-clean`, this
will remove the build folders on all supported platforms.

The tests results are available from the
[Actions](https://github.com/xpack-dev-tools/clang-xpack/actions/) page.
