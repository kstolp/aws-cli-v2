# Maintainer: Kevin Stolp <kevinstolp@gmail.com>
# Contributor: Kino <cybao292261@163.com>
# Contributor: Chih-Hsuan Yen <yan12125@archlinux.org>
# Contributor: Marcel Campello <marcel.campello@prafrentex.com.br>
# Contributor: David Birks <david@birks.dev>

pkgname=aws-cli-v2
# https://github.com/aws/aws-cli/raw/v2/CHANGELOG.rst
pkgver=2.27.50
pkgrel=1
pkgdesc='Unified command line interface for Amazon Web Services (version 2)'
arch=(any)
url='https://github.com/aws/aws-cli/tree/v2'
license=("Apache-2.0")
depends=(python python-awscrt python-certifi python-colorama python-dateutil
         python-distro python-docutils python-jmespath python-prompt_toolkit python-ruamel-yaml
         python-urllib3)
makedepends=(python-build python-wheel python-flit-core python-installer)
# Tests need the 'ps' binary
checkdepends=(python-pytest python-pytest-xdist python-jsonschema procps-ng)
provides=(aws-cli)
conflicts=(aws-cli)
install=aws-cli-v2.install
source=("https://awscli.amazonaws.com/awscli-$pkgver.tar.gz"{,.sig}
        build-ac.index-in-tmp.diff
        fix-env.diff
        "$pkgname-tz-fix.patch"
        "${pkgname}-ruamel-yaml-v4.patch"
        allow-egg-info.diff
        botocore-2990-rebased.patch
        botocore-2551.patch
        prompt_toolkit.patch)
sha256sums=('91ff7f7aa5c3f0a0dcbaba9807b0b294775d8b1925f34023434b122dfe9c3748'
            'SKIP'
            '444f0d2eb314f07474ce752e02932140bb9b9fec15c79bafe567f62a42398e93'
            '893d61d7e958c3c02bfa1e03bf58f6f6abd98849d248cc661f1c56423df9f312'
            'e4729ea33f9d5939618ce227a8f47afe27e3fd79528caaca752c499321fd1098'
            'e0cf52144da312228eee4de02d63ef4b8a0de4aca72d258783b7c826d1fd08a6'
            '1c16c82ffad39cf170e4673c8933b584162f206cb8d32c194795e5cf0e89cd69'
            '64d331c0b7e43834131c9fc549609a847faf96b0ae00c06f6e2d4ac7c8bb9089'
            '778c621885dae2218c840eec06a0e0294df7d1180dea12264b34a93994be7c0d'
            'f11edd5579a8de41431132b036ab4948ca749060ff63ee50d2a4cdd5c1adc087')
validpgpkeys=(
  'FB5DB77FD5C118B80511ADA8A6310ACC4672475C'  # the key mentioned on https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html
)

prepare() {
  cd awscli-$pkgver

  # Don't treat warnings as errors
  sed -i '/"error::/d' pyproject.toml

  # ac.index is an SQLite database, and building it on copy-on-write filesystems (ex: BTRFS) takes ages
  patch -Np1 -i ../build-ac.index-in-tmp.diff

  # Fix conflicts between tests/functional/test_clidriver.py::TestSession and tests/functional/botocore/leak/test_resource_leaks.py
  patch -Np1 -i ../fix-env.diff

  # Fix possible test failure with a non-UTC time zone (https://bugs.archlinux.org/task/77919)
  patch -Np1 -i ../$pkgname-tz-fix.patch

  # Fix tests with newer ruamel.yaml
  # https://github.com/aws/aws-cli/pull/8072 (unmerged)
  patch -Np1 -i ../${pkgname}-ruamel-yaml-v4.patch

  # tests/dependencies checks dependencies, and many Arch Linux packages are not using PEP 517 yet
  patch -Np1 -i ../allow-egg-info.diff

  # Workaround for issue with prompt_toolkit. Related issues:
  # https://github.com/aws/aws-cli/issues/9453
  # https://github.com/prompt-toolkit/python-prompt-toolkit/issues/1988
  patch -Np0 -i ../prompt_toolkit.patch

  pushd awscli

  # Backport fixes for urllib3 2.x to vendored botocore
  # [Move 100-continue behavior to use high-level request interface](https://github.com/boto/botocore/pull/2990)
  # Manually rebased due to conflicts from refactoring
  patch --no-backup-if-mismatch -Np1 -i ../../botocore-2990-rebased.patch

  # Backport an update to vendored six.py for compatibility with Python 3.12
  # [Update six from 1.10.0 to 1.16.0](https://github.com/boto/botocore/pull/2551)
  patch --no-backup-if-mismatch -Np1 -i ../../botocore-2551.patch

  popd
}

build() {
  cd awscli-$pkgver

  # flit-core adds runtime dependencies to reported build-time dependencies [1],
  # and upstream often lags behind the latest dependencies [2], thus --skip-dependency-check
  # [1] https://github.com/pypa/flit/issues/354
  # [2] https://github.com/aws/aws-cli/issues/5943
  python -m build --wheel --no-isolation --skip-dependency-check
  # Copy the built ac.index for tests
  cp -v build/unpacked_wheel/awscli/data/ac.index awscli/data/ac.index
}

#check() {
#  # Avoid intermittent test failures, see git commit messages
#  ulimit -S -n 4096
#
#  # Avoid the user's environment variable from being used in the tests
#  unset AWS_DEFAULT_PROFILE
#  unset AWS_PROFILE
#  unset AWS_REGION
#
#  cd awscli-$pkgver
#
#  export AWS_SECRET_ACCESS_KEY=fake_key
#  export AWS_ACCESS_KEY_ID=fake_id
#  export TZ=UTC
#
#  export PYTHONPATH="$PWD"
#
#  # Install a temporary copy to a virtual environment, as tests/dependencies checks global site-packages
#  python -m venv --system-site-packages "$PWD/venv"
#  "$PWD/venv/bin/python" -m installer dist/*.whl
#
#  # * Use --dist=loadfile following upstream. The default --dist=load may cause test failures and is not faster
#  # * Disable backend tests - those tests check if aws-cli can be installed or not, and are not compatible with all kinds of environments
#  # * Disable lockfile test since requirements-docs-lock.txt file is not present in downloaded archive
#  "$PWD/venv/bin/python" -m pytest tests -n 2 --dist loadfile --ignore=tests/backends --ignore=tests/integration --deselect='tests/dependencies/test_closure.py::TestDependencyClosure::test_lockfile_pins_within_runtime_dependencies_bounds[requirements-docs-lock.txt]'
#}

package() {
  cd awscli-$pkgver
  python -m installer --destdir="$pkgdir" dist/*.whl
  install -Dm 644 LICENSE.txt "$pkgdir/usr/share/licenses/$pkgname/LICENSE"
  install -Dm 644 bin/aws_bash_completer "$pkgdir/usr/share/bash-completion/completions/aws"
}
