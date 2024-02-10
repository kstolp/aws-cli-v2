# Maintainer: Chih-Hsuan Yen <yan12125@archlinux.org>
# Contributor: Marcel Campello <marcel.campello@prafrentex.com.br>
# Contributor: David Birks <david@birks.dev>

pkgname=aws-cli-v2
# https://github.com/aws/aws-cli/raw/v2/CHANGELOG.rst
pkgver=2.15.19
pkgrel=1
pkgdesc='Unified command line interface for Amazon Web Services (version 2)'
arch=(any)
url='https://github.com/aws/aws-cli/tree/v2'
license=(Apache)
depends=(python python-awscrt python-certifi python-colorama python-cryptography python-dateutil
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
        botocore-2922.patch
        botocore-2924.patch
        botocore-2990-rebased.patch)
sha256sums=('4acf1cbb9bd7f158d13276f18467d9f8fd8dde504e537141fe8899b1273f9d5c'
            'SKIP'
            '0267e41561ab2c46a97ebfb024f0b047aabc9e6b9866f204b2c1a84ee5810d63'
            '893d61d7e958c3c02bfa1e03bf58f6f6abd98849d248cc661f1c56423df9f312'
            '4fc614b8550d7363bb2d578c6b49326c9255203eb2f933fd0551f96ed5fb1f30'
            '20a9fcd5235bf606e86a6ec06ca30307ebbcfd36063d2ac561c1f9eff7243046'
            '6768df8667fe7fd827e6eef1c4cdb3eae25aba5806bbc725270200a585f62152'
            '62be6cad0f9039ae682abffd167181abbd4a690e2680867418c5542893d74b36'
            'aad8b863d9f9107c56401e71d76b71f526efd9f8efac31e2a007b9071f85b5b6'
            'a43c3e9aba8974fc09f1780a37b6a94108b15dbbbcecdf6d9e7e224ca135816b')
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

  # Backport fixes for urllib3 2.x to vendored botocore
  pushd awscli
  # [Defer to system defaults for cipher suites with urllib3 2.0+](https://github.com/boto/botocore/pull/2922)
  patch --no-backup-if-mismatch -Np1 -i ../../botocore-2922.patch
  # [Do not set_ciphers(DEFAULT_CIPHERS) if DEFAULT_CIPHERS is None](https://github.com/boto/botocore/pull/2924)
  patch --no-backup-if-mismatch -Np1 -i ../../botocore-2924.patch
  # [Move 100-continue behavior to use high-level request interface](https://github.com/boto/botocore/pull/2990)
  # Manually rebased due to conflicts from refactoring
  patch --no-backup-if-mismatch -Np1 -i ../../botocore-2990-rebased.patch
  popd

  # use unittest.mock
  # https://src.fedoraproject.org/rpms/awscli2/blob/rawhide/f/awscli2.spec
  find -type f -name '*.py' -exec sed \
      -e 's/^\( *\)import mock$/\1from unittest import mock/' \
      -e 's/^\( *\)from mock import mock/\1from unittest import mock/' \
      -e 's/^\( *\)from mock import/\1from unittest.mock import/' \
      -i '{}' +
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

check() {
  cd awscli-$pkgver

  export AWS_SECRET_ACCESS_KEY=fake_key
  export AWS_ACCESS_KEY_ID=fake_id

  export PYTHONPATH="$PWD"

  # Install a temporary copy to a virtual environment, as tests/dependencies checks global site-packages
  python -m venv --system-site-packages "$PWD/venv"
  "$PWD/venv/bin/python" -m installer dist/*.whl

  # * Use --dist=loadfile following upstream. The default --dist=load may cause test failures and is not faster
  # * Disable backend tests - those tests check if aws-cli can be installed or not, and are not compatible with all kinds of environments
  "$PWD/venv/bin/python" -m pytest tests -n auto --dist loadfile --ignore=tests/backends --ignore=tests/integration
}

package() {
  cd awscli-$pkgver
  python -m installer --destdir="$pkgdir" dist/*.whl
  install -Dm 644 LICENSE.txt "$pkgdir/usr/share/licenses/$pkgname/LICENSE"
  install -Dm 644 bin/aws_bash_completer "$pkgdir/usr/share/bash-completion/completions/aws"
}
