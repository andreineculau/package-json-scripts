# package-json-scripts

A NodeJS package to test how `package.json` scripts are being scheduled by
different package managers (e.g. npm, yarn, pnpm).

[See results!](#results)


## Why?

Documentation is rather scarse, and hard to follow.
e.g. npm-scripts https://github.com/npm/npm/blob/87629880a71baec352c1b5345bc29268d6212467/doc/misc/npm-scripts.md

If package managers want to piggy back on npm's `package.json` then they should follow it ad-literam,
or build consensus over a `package.json` standard i.e. make it npm-abstract.

I am primarily interested in how the package managers treat git dependencies;
see the [raison d'être](https://github.com/andreineculau/npm-publish-git#raison-dêtre)
of https://github.com/andreineculau/npm-publish-git .


## How?

I first started to document which scripts are run by each command,
by investigating the codebase, but soon realized that this was not a solution.

Instead I created this repo which basically registers a callback (=script) for each script name
that a package manager has a convention about.

A callback logs a timestamp (and environment variables for debugging purposes) on the file system.
At the end of the command execution, we print a log of the callbacks that were called.


## Installation

```shell
git clone git://github.com/andreineculau/package-json-scripts.git
```


## Usage

```shell
cd package-json-scripts
./test npm install
```

will end the execution by printing something like

```
/tmp/package-json-scripts/npm-5.3.0/1500753621857633000.npm_install.package-json-scripts
└── package-json-scripts
    ├── 1500753623507057000.preinstall.preinstall
    ├── 1500753624339557000.install.install
    ├── 1500753625133615000.postinstall.postinstall
    ├── 1500753626005398000.prepublish.prepublish
    ├── 1500753626918195000.prepare.prepare
    ├── 1500753627691279000.preshrinkwrap.preshrinkwrap
    ├── 1500753628482127000.shrinkwrap.shrinkwrap
    └── 1500753629289806000.postshrinkwrap.postshrinkwrap
```

which reads like

- npm version `5.3.0`
- executing `npm_install`
- in the `package-json-scripts` package
- ran the following scripts for the `package-json-scripts` package
- scripts ordered by timestamp, script name, npm_lifecycle_event (should be the same with the script name)

`./test path/to/npm@2/node_modules/.bin/npm install` would produce a similar file structure in
e.g. `/tmp/package-json-scripts/npm-3.10.10/1500753629289806000.npm_install.package-json-scripts`
(see updated npm version).

`./test yarn install` would produce a similar file structure in
e.g. `/tmp/package-json-scripts/yarn-0.27.5/1500753629289806000.yarn_install.package-json-scripts`
(see updated package manager name and version).


## Results

**I have recorded my results for npm@3 npm@4 npm@5, yarn@0.27 and pnpm@1.7.1 in
https://docs.google.com/spreadsheets/d/1XH8iDyOmNf-ZH3A2SMV5YTDcT2YPXmhFcaUPsIyYEu8/edit?usp=sharing .
Anyone should be able to add comments.**

For atomicity, a PDF (might not out of date with the link above) is checked in as [results.pdf](results.pdf).


### Observation: general unpredictability

Your package will be installed, uninstalled, packed, published, etc
in different ways based on the package manager that you, or your user is using.


### Observation: `npm install <git dep>` is unpredictable

**Example 1.** a package from the npm@4 times has a `prepare` script for `npm pack` or `npm publish`.
Now npm@5 will trigger that on `npm install`. From npm's perspective this may sound like a safe change,
but what if a build system with npm@4 in mind is designed to run the `prepare` script itself?
At best, the second run is a noop, but the initial state is different nonetheless.

**Example 2.** a package from the npm@5 times has a `prepare` script for `npm install`, but is installed
via npm@4 thus the `prepare` script is not run, resulting in a incomplete installation.

**Example 3.** a package from the npm@3 times has a `prepare` script for `npm pack` or `npm publish`,
maybe a simple `make` which was fetching dependencies, among them npm dependencies by running `npm install`.
Now npm@4 starts to run the `prepare` script as part of `npm install (no args)` -> recursive loop:
install calls prepare which calls install which calls...

**NOTE** `package.json` actually used to have a `engineStrict` boolean flag for those that remember.
Now it's only a flag for the `npm` command, in the hands of the user.

**WORKAROUND** In order to fail the installation on the grounds of incompatible package manager,
one should have a `preinstall` script and parse the `npm_config_user_agent` environment variable
e.g. `npm/5.3.0 node/v8.2.1 darwin x64`.

**WOKRAROUND** Alternatively, rather than failing the install,
in order to make your package installable as a git dependency via npm@3 npm@4 npm@5,
you could maybe write a `postinstall` script to check the npm version (see NOTE above),
and run the `prepare` script for npm@3 for example...


### Observation: yarn, yarn, yarn, yarn... yawn, yawn, yawn, yawn...

I cannot even find docs of how yarn intends to call the scripts,
nor with which npm version they want to provide compatibility with, if they do at all
(`yarn publish` is deliberately not aligned with `npm publish`).

For that reason, it is virtually impossible to say whether it's an error or not
the fact that

1. `yarn publish` first runs `prepublishOnly` and then `prepare`
1. `yarn publish` has a bump-version step,
   which may wrap the `npm publish` flow with `preversion` and `version+postversion`.
   **NOTE** coupling the two was regarded as progress ?! I guess á la `svn commit`.
1. `yarn` doesn't know (yet?) `prepack` and `postpack`
1. `yarn pack` doesn't call **any** scripts - no `prepublish`, no `prepare`, no nothing

**NOTE** further discrepancies were found when comparing the environment variables of a
yarn command and an npm command, but this was not my goal so far. See [diff_npm_vars.txt](diff_npm_vars.txt).


## Refs

* https://github.com/yarnpkg/yarn/issues/3992
* https://github.com/yarnpkg/yarn/issues/3209
* https://github.com/yarnpkg/rfcs/pull/70
* https://github.com/npm/npm/issues/17893
* https://github.com/pnpm/pnpm/issues/854
* https://github.com/pnpm/pnpm/issues/855

## License

[UNLICENSE](UNLICENSE)
