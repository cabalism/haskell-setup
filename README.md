# haskell-actions/setup

[![GitHub Actions status](https://github.com/haskell-actions/setup/actions/workflows/workflow.yml/badge.svg)](https://github.com/haskell-actions/setup/actions/workflows/workflow.yml)

This action sets up a Haskell environment for use in actions by:

- if requested, installing a version of [ghc](https://downloads.haskell.org/~ghc/latest/docs/html/users_guide/) and [cabal](https://www.haskell.org/cabal/) and adding them to `PATH`,
- if requested, installing a version of [Stack](https://haskellstack.org) and adding it to the `PATH`,
- outputting of `ghc-version/exe/path`, `cabal-version/exe/path`, `stack-version/exe/path`, `stack-root` and `cabal-store` (for the requested components).

The GitHub runners come with [pre-installed versions of GHC and Cabal](https://github.com/actions/runner-images).
Those will be used whenever possible.
For all other versions, this action utilizes
[`ppa:hvr/ghc`](https://launchpad.net/~hvr/+archive/ubuntu/ghc),
[`ghcup`](https://github.com/haskell/ghcup-hs), and
[`chocolatey`](https://chocolatey.org/packages/ghc).

## Usage

See [action.yml](action.yml)

### Minimal

```yaml
on: [push]
name: build
jobs:
  runhaskell:
    name: Hello World
    runs-on: ubuntu-latest # or macOS-latest, or windows-latest
    steps:
      - uses: actions/checkout@v4
      - uses: haskell-actions/setup@v2
      - run: runhaskell Hello.hs
```

### Basic

```yaml
on: [push]
name: build
jobs:
  runhaskell:
    name: Hello World
    runs-on: ubuntu-latest # or macOS-latest, or windows-latest
    steps:
      - uses: actions/checkout@v4
      - uses: haskell-actions/setup@v2
        with:
          ghc-version: '8.8' # Resolves to the latest point release of GHC 8.8
          cabal-version: '3.0.0.0' # Exact version of Cabal
      - run: runhaskell Hello.hs
```

### Basic with Stack

```yaml
on: [push]
name: build
jobs:
  runhaskell:
    name: Hello World
    runs-on: ubuntu-latest # or macOS-latest, or windows-latest
    steps:
      - uses: actions/checkout@v4
      - uses: haskell-actions/setup@v2
        with:
          ghc-version: '8.8.4' # Exact version of ghc to use
          # cabal-version: 'latest'. Omitted, but defaults to 'latest'
          enable-stack: true
          stack-version: 'latest'
      - run: runhaskell Hello.hs
```

### Matrix Testing

```yaml
on: [push]
name: build
jobs:
  build:
    runs-on: ${{ matrix.os }}
    strategy:
      matrix:
        ghc: ['8.6.5', '8.8.4']
        cabal: ['2.4.1.0', '3.0.0.0']
        os: [ubuntu-latest, macOS-latest, windows-latest]
        exclude:
          # GHC 8.8+ only works with cabal v3+
          - ghc: 8.8.4
            cabal: 2.4.1.0
    name: Haskell GHC ${{ matrix.ghc }} sample
    steps:
      - uses: actions/checkout@v4
      - name: Setup Haskell
        uses: haskell-actions/setup@v2
        with:
          ghc-version: ${{ matrix.ghc }}
          cabal-version: ${{ matrix.cabal }}
      - run: runhaskell Hello.hs
```

### Model cabal workflow with caching

```yaml
name: build
on:
  push:
    branches: [main, master]
  pull_request:
    branches: [main, master]

# INFO: The following configuration block ensures that only one build runs per branch,
# which may be desirable for projects with a costly build process.
# Remove this block from the CI workflow to let each CI job run to completion.
concurrency:
  group: build-${{ github.ref }}
  cancel-in-progress: true

jobs:
  build:
    name: GHC ${{ matrix.ghc-version }} on ${{ matrix.os }}
    runs-on: ${{ matrix.os }}
    strategy:
      fail-fast: false
      matrix:
        os: [ubuntu-latest]
        ghc-version: ['9.8', '9.6', '9.4', '9.2', '9.0']

        include:
          - os: windows-latest
            ghc-version: '9.8'
          - os: macos-latest
            ghc-version: '9.8'

    steps:
      - uses: actions/checkout@v4

      - name: Set up GHC ${{ matrix.ghc-version }}
        uses: haskell-actions/setup@v2
        id: setup
        with:
          ghc-version: ${{ matrix.ghc-version }}
          # Defaults, added for clarity:
          cabal-version: 'latest'
          cabal-update: true

      - name: Configure the build
        run: |
          cabal configure --enable-tests --enable-benchmarks --disable-documentation
          cabal build all --dry-run
        # The last step generates dist-newstyle/cache/plan.json for the cache key.

      - name: Restore cached dependencies
        uses: actions/cache/restore@v4
        id: cache
        env:
          key: ${{ runner.os }}-ghc-${{ steps.setup.outputs.ghc-version }}-cabal-${{ steps.setup.outputs.cabal-version }}
        with:
          path: ${{ steps.setup.outputs.cabal-store }}
          key: ${{ env.key }}-plan-${{ hashFiles('**/plan.json') }}
          restore-keys: ${{ env.key }}-

      - name: Install dependencies
        # If we had an exact cache hit, the dependencies will be up to date.
        if: steps.cache.outputs.cache-hit != 'true'
        run: cabal build all --only-dependencies

      # Cache dependencies already here, so that we do not have to rebuild them should the subsequent steps fail.
      - name: Save cached dependencies
        uses: actions/cache/save@v4
        # If we had an exact cache hit, trying to save the cache would error because of key clash.
        if: steps.cache.outputs.cache-hit != 'true'
        with:
          path: ${{ steps.setup.outputs.cabal-store }}
          key: ${{ steps.cache.outputs.cache-primary-key }}

      - name: Build
        run: cabal build all

      - name: Run tests
        run: cabal test all

      - name: Check cabal file
        run: cabal check

      - name: Build documentation
        run: cabal haddock all
```

## Inputs

| Name                    | Description                                                                                                                                 | Type      | Default     |
| ----------------------- | ------------------------------------------------------------------------------------------------------------------------------------------- | --------- | ----------- |
| `ghc-version`           | GHC version to use, e.g. `9.2` or `9.2.5`.                                                                                                  | `string`  | `latest`    |
| `cabal-version`         | Cabal version to use, e.g. `3.6`.                                                                                                           | `string`  | `latest`    |
| `stack-version`         | Stack version to use, e.g. `latest`. Stack will only be installed if `enable-stack` is set.                                                 | `string`  | `latest`    |
| `enable-stack`          | If set, will setup Stack.                                                                                                                   | "boolean" | false/unset |
| `stack-no-global`       | If set, `enable-stack` must be set. Prevents installing GHC and Cabal globally.                                                             | "boolean" | false/unset |
| `stack-setup-ghc`       | If set, `enable-stack` must be set. Runs stack setup to install the specified GHC. (Note: setting this does _not_ imply `stack-no-global`.) | "boolean" | false/unset |
| `disable-matcher`       | If set, disables match messages from GHC as GitHub CI annotations.                                                                          | "boolean" | false/unset |
| `cabal-update`          | If set to `false`, skip `cabal update` step.                                                                                                | `boolean` | `true`      |
| `ghcup-release-channel` | If set, add a [release channel](https://www.haskell.org/ghcup/guide/#metadata) to ghcup.                                                    | `URL`     | none        |

Note: "boolean" types are set/unset, not true/false.
That is, setting any "boolean" to a value other than the empty string (`""`) will be considered true/set.
However, to avoid confusion and for forward compatibility, it is still recommended to **only use value `true` to set a "boolean" flag.**

In contrast, a proper `boolean` input like `cabal-update` only accepts values `true` and `false`.

## Outputs

The action outputs parameters for the components it installed.
E.g. if `ghc-version: 8.10` is requested, the action will output `ghc-version: 8.10.7` if installation succeeded,
and `ghc-exe` and `ghc-path` will be set accordingly.
(Details on version resolution see next section.)

| Name            | Description                                                                                                                | Type   |
| --------------- | -------------------------------------------------------------------------------------------------------------------------- | ------ |
| `ghc-version`   | The resolved version of `ghc`                                                                                              | string |
| `cabal-version` | The resolved version of `cabal`                                                                                            | string |
| `stack-version` | The resolved version of `stack`                                                                                            | string |
| `ghc-exe`       | The path of the `ghc` _executable_                                                                                         | string |
| `cabal-exe`     | The path of the `cabal` _executable_                                                                                       | string |
| `stack-exe`     | The path of the `stack` _executable_                                                                                       | string |
| `ghc-path`      | The path of the `ghc` executable _directory_                                                                               | string |
| `cabal-path`    | The path of the `cabal` executable _directory_                                                                             | string |
| `stack-path`    | The path of the `stack` executable _directory_                                                                             | string |
| `cabal-store`   | The path to the cabal store                                                                                                | string |
| `stack-root`    | The path to the stack root (equal to the `STACK_ROOT` environment variable if it is set; otherwise an OS-specific default) | string |

## Version Support

This action is conscious about the tool versions specified in [`versions.json`](src/versions.json).
This list is replicated (hopefully correctly) below.

Versions specified by the inputs, e.g. `ghc-version`, are resolved against this list,
by taking the first entry from the list if `latest` is requested,
or the first entry that matches exactly,
or otherwise the first entry that is a (string-)extension of the requested version extended by a `.`.
E.g., `8.10` will be resolved to `8.10.7`, and so will `8`.

**GHC:**

- `latest-nightly` (requires the resp. `ghcup-release-channel`, e.g. `https://ghc.gitlab.haskell.org/ghcup-metadata/ghcup-nightlies-0.0.7.yaml`)
- `latest` (default)
- `9.10.1` `9.10`
- `9.8.2` `9.8`
- `9.8.1`
- `9.6.5` `9.6`
- `9.6.4`
- `9.6.3`
- `9.6.2`
- `9.6.1`
- `9.4.8` `9.4`
- `9.4.7`
- `9.4.6`
- `9.4.5`
- `9.4.4`
- `9.4.3`
- `9.4.2`
- `9.4.1`
- `9.2.8` `9.2`
- `9.2.7`
- `9.2.6`
- `9.2.5`
- `9.2.4`
- `9.2.3`
- `9.2.2`
- `9.2.1`
- `9.0.2` `9.0`
- `9.0.1`
- `8.10.7` `8.10`
- `8.10.6`
- `8.10.5`
- `8.10.4`
- `8.10.3`
- `8.10.2`
- `8.10.1`
- `8.8.4` `8.8`
- `8.8.3`
- `8.8.2`
- `8.8.1`
- `8.6.5` `8.6`
- `8.6.4`
- `8.6.3`
- `8.6.2`
- `8.6.1`
- `8.4.4` `8.4`
- `8.4.3`
- `8.4.2`
- `8.4.1`
- `8.2.2` `8.2`
- `8.0.2` `8.0`
- `7.10.3` `7.10` (deprecated, not on `ubuntu-22.04` or up)

Suggestion: Try to support at least the three latest major versions of GHC.

**Cabal:**

- `head` (the [cabal-head](https://github.com/haskell/cabal/releases/tag/cabal-head) release of the most recent build of the `master` branch)
- `latest` (default, recommended)
- `3.10.3.0` `3.10`
- `3.10.2.1`
- `3.10.2.0`
- `3.10.1.0`
- `3.8.1.0` `3.8`
- `3.6.2.0` `3.6`
- `3.6.0.0`
- `3.4.1.0` `3.4`
- `3.4.0.0`
- `3.2.0.0` `3.2`
- `3.0.0.0` `3.0`
- `2.4.1.0` `2.4`

Recommendation: Use the latest available version if possible.

**Stack:** (with `enable-stack: true`)

- `latest` (default, recommended)
- `2.15.7` `2.15`
- `2.15.5`
- `2.15.3`
- `2.15.1`
- `2.13.1` `2.13`
- `2.11.1` `2.11`
- `2.9.3` `2.9`
- `2.9.1`
- `2.7.5` `2.7`
- `2.7.3`
- `2.7.1`
- `2.5.1` `2.5`
- `2.3.3` `2.3`
- `2.3.1`
- `2.1.3` `2.1`
- `2.1.1`
- `1.9.3.1` `1.9`
- `1.9.1.1`
- `1.7.1` `1.7`
- `1.6.5` `1.6`
- `1.6.3.1`
- `1.6.1.1`
- `1.5.1` `1.5`
- `1.5.0`
- `1.4.0` `1.4`
- `1.3.2` `1.3`
- `1.3.0`
- `1.2.0` `1.2`

Recommendation: Use the latest available version if possible.

Beyond the officially supported listed versions above, you can request any precise version of
[GHC](https://www.haskell.org/ghc/download.html),
[Cabal](https://www.haskell.org/cabal/download.html), and
[Stack](https://github.com/commercialhaskell/stack/tags).
The action will forward the request to the install methods (apt, ghcup, choco), and installation might succeed.

Note however that Chocolatey's version numbers might differ from the official ones,
please consult their pages for
[GHC](https://chocolatey.org/packages/ghc#versionhistory) and
[Cabal](https://chocolatey.org/packages/cabal#versionhistory).

## License

The scripts and documentation in this project are released under the [MIT License](LICENSE).

## Contributions

Contributions are welcome! See the [Contributor's Guide](docs/contributors.md).
