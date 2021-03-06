lsst-build, a builder and continuous integration tool for LSST
==============================================================

[![Build Status](https://travis-ci.org/lsst/lsst_build.svg?branch=master)](https://travis-ci.org/lsst/lsst_build)

Provides the following capabilities:

* Given one or more top-level packages, intelligently clone their git
  repositories and check out the requested branches into a build directory:

  ```bash
  lsst-build prepare
     [--repository-pattern=format_pattern_for_repo_URLs]
     [--exclusion-map=exclusions.txt]
     [--version-git-repo=versiondbdir]
     [--ref=branch1 [--ref=branch2 [...]]]
     <builddir> <product1> [product2 [product3 [...]]]
  ```

  Run `lsst-build prepare -h` to see the full list of options.

* Given that build directory, and an EUPS stack, intelligently build and
  install those packages:

  ```bash
  lsst-build build <builddir>
  ```

  Run `lsst-build build -h` to see the full list of options.

Example:

```bash
git clone git@github.com:lsst/versiondb.git
export REPOSITORY_PATTERN="git://git.lsstcorp.org/LSST/DMS/%(product)s.git|git://git.lsstcorp.org/LSST/DMS/devenv/%(product)s.git|git://git.lsstcorp.org/LSST/DMS/testdata/%(product)s.git|git://git.lsstcorp.org/LSST/external/%(product)s.git"

cat > exclusions.txt <<-EOF
    # Exclusion map. Format:
    # dependency regex      product regex
    cuda_toolkit            .*
    cuda_sdk                .*
    scipy                   .*
    healpy                  .*
    condor                  .*
EOF

mkdir build

lsst-build prepare \
    --exclusion-map=exclusions.txt
    --version-git-repo=versiondb
    build lsst_distrib

lsst-build build build
```

`lsst-build prepare`
------------------

`lsst-build prepare` clones the given top level product(s) into `<builddir>`,
discovers its dependencies by reading its `ups/$PRODUCT.table` file, and
recursively clones them until all dependencies are exhausted.  Following the
clone, it writes out `<builddir>/manifest.txt` file, which lists all cloned
products, their versions and dependency relationships.

It maps product names to git repository URLs via a repository pattern, a
`|`-separated string of python string format patterns passed by
`--repository-pattern` option, or existing in a `$REPOSITORY_PATTERN`
environment variable. Each format pattern is evaluated in order with
`%(product)` replaced by the product name to construct the URL, and a git-clone
is attempted until a clone is successful.

`lsst-build` skips any dependencies matching a rule in an exclusion map given
`--via exclusion-map` option.  The exclusion map is a text file with two
entries per line: the dependency regex and the product regex. Any product that
matches the dependency regex, while being considered for cloning as a
dependency of a product that matches product regex, will be skipped.

Once cloned, a branch/tag/commit given by `--ref` is checked out. If multiple
`--ref` options are given, each ref is tested for existence until one that
exists is found and checked out. If no requested refs exist, `master` is
checked out. This is commonly used to prepare and test the build of a ticket
branch, falling back to master if that branch doesn't exist in some
repositories (i.e., `lsst-build prepare --ref ticket/1234 --ref master ...`).

Upon completing the clone and ref checkouts of all packages in the product
tree, `lsst-build prepare` writes out a "build manifest" in
`<builddir>/manifest.txt`.  This is a topologically sorted whitespace-separated
list of (product, sha1, version, dependencies), with one line per cloned
product.  Given the topological sorting, it is possible to build the packages
in order of appearance in the file. The 'dependencies' entry is a
comma-separated list of dependency product names (first level only).

In addition to these tuples, the manifest may also contain variable assignments
of the form `VARNAME=VARVALUE`.  Current implementation of `lsst-build` defines
only one, `BUILD`, which is a locally unique identifier identifying this
particular set of packages (i.e., a "build number").

The construction of version string in the build manifest depends on whether
`--version-git-repo` option has been used. If it is not, the version string is
constructed as `<pkgautoversion>+<deps_sha1>`, where pkgautoversion is the
output of EUPS' pkgautoversion tool when run in the product's git repository,
and `<deps_sha1>` is the abbreviated SHA1 of a sorted, space-separated list of
(product, version) tuples of its dependencies. This guarantees that two
disconnected runs of `lsst-build` with exactly the same input repositories will
compute the same versions.

If `--version-git-repo=<versiondb>` is given, then the versions are of the form
`<pkgautoversion>+<N>`, where N is a monotonically increasing integer.
Nevertheless, N is guaranteed to be the same for the same set of dependencies.
To achieve this, the files in `<versiondb>` directory record the mapping from
deps_sha1, computed as described in the previous paragraph, and the integer N.
Given the same source repositories, and the same versiondb repository, two
disconnected `lsst-build prepare` runs are guaranteed to generate the same
versions.  Note that `<versiondb>` must be a specially formatted git repository
(see the 'VersionDB Repository' section for more).

The construction of the build ID (`BUILD=xxx` entry in the manifest) also
depends on whether `--version-git-repo` option is in effect. If no, then this
ID is computed by looking for the highest EUPS tag matching `b*` and
incrementing it by one (e.g., if EUPS tags `b1`, `b2`, and `b3` already exist,
`BUILD=b4` would be written into the manifest). If `--version-git-repo` is
used, the same algorithm is applied but on *git* tags of the versiondb
repository.  Also, if `--version-git-repo` is used the manifest will be stored
in versiondb repository, the repository will be git-committed and tagged with
the build ID.

If `--version-git-repo` is used, it is advisable to git-push the repository
upstream upon a successful `lsst-build prepare` (esp. before `lsst-build build` is
run). This ensures that no versions can exist in the installed stack that are
not present in the versiondb repo on the central server.

`lsst-build build`
----------------

`lsst-build build` builds, installs and EUPS-declares, each product found in
the list in `<builddir>/manifest.txt` unless the product with the same name and
version is already EUPS-declared in the stack. The build is done using the
eupspkg mechanism; products not using eupspkg cannot be built with `lsst-build
build`. Each successfully installed product is EUPS-tagged with the build
identifier (the `BUILD=xxxx` entry in `manifest.txt`).

For each product found in `<builddir>/manifest.txt`, `lsst-build` build first
checks if it already exists on the stack in EUPS_PATH. If not, it enters its
directory, sets up its dependencies (roughly, using `setup -r .`"), and runs
eupspkg {prep, config, build, install} sequence. If the build is successful, it
declares the product to EUPS. Either way, the product is tagged with the `BUILD`
value given in manifest.txt.

At the end of a successful run, all products listed in `<builddir>/manfest.txt`
will have been build, declared and installed into the active EUPS stack (the
first entry on `$EUPS_PATH`), and tagged with the value of `BUILD` (the "build
number").

Environment Variables
---------------------

`lsst-build` may use the following environment variables:

* `REPOSITORY_PATTERN`

  A `|`-separated string of python string format patterns used to map product
  names to URLs of their git repositories.  Each format pattern is evaluated
  with `%(product)` replaced by the product name to construct the URL.  A
  git-clone is attempted on each one (in order), until a clone is successful.
  Setting this variable is equivalent to specifying the patter via
  `--repository-pattern` option.

* `EUPS_PATH`

  The colon-separated path to EUPS-managed software stacks. `lsst-build` will
  use these (via EUPS) to determine whether products need to be built or have
  already been built, and will install new products into this location (using
  eupspkg).

VersionDB Repository
--------------------

VersionDB is a specially-formatted git-managed directory that serves as a `+N`
dependency number and build ID number database for `lsst-build` with
`--version-git-repo` option (see above).

It consists of three subdirectories:

* `ver_db/`

  A directory of text files, one per product, with space-separated
  (product_version, N, deps_sha1) pairs.  Used by `lsst-build` to always assign
  the same `+N` suffix to same set of dependencies (both products and their
  versions).

* `dep_db/`

  A directory of text files, one per product, with space-separated list of
  (product_version, N, dep_name, dep_version) tuples.  May be used by the user
  to quickly look up a set of dependencies corresponding to some `+N` suffix.
  Note that this information is also available in the product's expanded table
  file.

* `manifests/`

  A directory where build manifests are stored. Each manifest is named
  `$BUILD.txt`, where `$BUILD` is the unique build ID.  Every time `lsst-build`
  is run with `--version-git-repo`, it will add the resulting manifest to this
  directory, git-commit it, and tag it with the build ID, unless a manifest
  with matching content already exists. In the latter case, that build ID will
  be reused and no new commits will be added to VersionDB.

Note that it is not necessary to understand this internal format to use this
repository; in fact, one should *not* depend on its internal format, as it may
change as `lsst-build` itself is improved.
