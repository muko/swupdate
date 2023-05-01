.. SPDX-FileCopyrightText: 2013-2021 Stefano Babic <sbabic@denx.de>
.. SPDX-License-Identifier: GPL-2.0-only

..
        ==================================
        meta-swupdate: building with Yocto
        ==================================
==================================
meta-swupdate: Yocto でビルド
==================================


..
        Overview
概要
========

..
        The Yocto-Project_ is a community project under the umbrella of the Linux
        Foundation that provides tools and template to create the own custom Linux-based
        software for embedded systems.
Yocto-Project_ は Linux Foundation の傘下にあるコミュニティ プロジェクトで、組み込みシステム用の独自のカスタム Linux ベース ソフトウェアを作成するためのツールとテンプレートを提供します。

.. _Yocto-Project: http://www.yoctoproject.org
.. _meta-SWUpdate:  https://github.com/sbabic/meta-swupdate.git

..
        Add-on features can be added using *layers*. meta-swupdate_ is the layer to
        cross-compile the SWUpdate application and to generate the compound SWU images
        containing the release of the product.  It contains the required changes
        for mtd-utils and for generating Lua. Using meta-SWUpdate is a
        straightforward process. As described in Yocto's documentation
        about layers, you should include it in your *bblayers.conf* file to use it.
アドオン機能は、 *レイヤー* を使用して追加できます。
meta-swupdate_ は、SWUpdate アプリケーションをクロスコンパイルし、製品のリリースを含む複合 SWU イメージを生成する層です。
これには、mtd-utils と Lua の生成に必要な変更が含まれています。
meta-SWUpdate の使用は簡単なプロセスです。
レイヤーに関する Yocto のドキュメントで説明されているように、これを *bblayers.conf* ファイルに含めて使用する必要があります。

..
        Add meta-SWUpdate as usual to your bblayers.conf. You have also
        to add meta-oe to the list.
通常どおり、meta-SWUpdate を bblayers.conf に追加します。
また、meta-oe をリストに追加する必要があります。 

..
        In meta-SWUpdate there is a recipe to generate an initrd with a
        rescue system with SWUpdate. Use:
meta-SWUpdate には、SWUpdate を使用したレスキュー システムで initrd を生成するためのレシピがあります。
以下のように使います：

::

	MACHINE=<your machine> bitbake swupdate-image

..
        You will find the result in your tmp/deploy/<your machine> directory.
        How to install and start an initrd is very target specific - please
        check in the documentation of your bootloader.
結果は tmp/deploy/<your machine> ディレクトリにあります。
initrd をインストールして開始する方法は、ターゲット固有です。
ブートローダーのドキュメントを確認してください。

..
        What about libubootenv ?
libubootenv はどうですか？
========================

..
        This is a common issue when SWUpdate is built. SWUpdate depends on this library,
        that is generated from the U-Boot's sources. This library allows one to safe modify
        the U-Boot environment. It is not required if U-Boot is not used as bootloader.
        If SWUpdate cannot be linked, you are using an old version of U-Boot (you need
        at least 2016.05). If this is the case, you can add your own recipe for
        the package u-boot-fw-utils, adding the code for the library.
これは、SWUpdate のビルド時によく発生する問題です。
SWUpdate は、U-Boot のソースから生成されるこのライブラリに依存します。
このライブラリを使用すると、U-Boot 環境を安全に変更できます。
U-Boot をブートローダとして使用しない場合は必要ありません。
SWUpdate をリンクできない場合は、古いバージョンの U-Boot を使用しています (少なくとも 2016.05 が必要です)。
この場合、ライブラリのコードを追加して、パッケージ u-boot-fw-utils の独自のレシピを追加できます。

..
        It is important that the package u-boot-fw-utils is built with the same
        sources of the bootloader and for the same machine. In fact, the target
        can have a default environment linked together with U-Boot's code,
        and it is not (yet) stored into a storage. SWUpdate should be aware of
        it, because it cannot read it: the default environment must be linked
        as well to SWUpdate's code. This is done inside the libubootenv.
パッケージ u-boot-fw-utils が、同じマシン用の同じソースのブートローダーでビルドされていることが重要です。
実際、ターゲットは U-Boot のコードと一緒にリンクされたデフォルト環境を持つことができますが、それは (まだ) ストレージに格納されていません。
SWUpdate はそれを読み取ることができないため、これを認識する必要があります。
デフォルト環境も SWUpdate のコードにリンクする必要があります。
これは libubootenv 内で行われます。

..
        If you build for a different machine, SWUpdate will destroy the
        environment when it tries to change it the first time. In fact,
        a wrong default environment is taken, and your board won't boot again.
別のマシン用にビルドすると、SWUpdate は最初に環境を変更しようとしたときに環境を破壊します。
実際、間違ったデフォルト環境が使用され、ボードが再起動しなくなります。 

..
        To avoid possible mismatch, a new library was developed to be hardware independent.
        A strict match with the bootloader is not required anymore. The meta-swupdate layer
        contains recipes to build the new library (`libubootenv`) and adjust SWUpdate to be linked
        against it. To use it as replacement for u-boot-fw-utils:
不一致の可能性を回避するために、ハードウェアに依存しない新しいライブラリが開発されました。
ブートローダーとの厳密な一致はもはや必要ありません。
meta-swupdate 層には、新しいライブラリ (`libubootenv`) を構築し、それにリンクされるように SWUpdate を調整するためのレシピが含まれています。
u-boot-fw-utils の代わりとして使用するには:

        - set PREFERRED_PROVIDER_u-boot-fw-utils = "libubootenv"
        - add to SWUpdate config:

::

                CONFIG_UBOOT=y

..
        With this library, you can simply pass the default environment as file (u-boot-initial-env).
        It is recommended for new project to switch to the new library to become independent from
        the bootloader.
このライブラリを使用すると、デフォルトの環境をファイル (u-boot-initial-env) として簡単に渡すことができます。
新しいプロジェクトを新しいライブラリに切り替えて、ブートローダーから独立させることをお勧めします。

..
        The swupdate class
swupdateクラス
==================

..
        meta-swupdate contains a class specific for SWUpdate. It helps to generate the
        SWU image starting from images built inside the Yocto. It requires that all
        components, that means the artifacts that are part of the SWU image, are present
        in the Yocto's deploy directory.  This class should be inherited by recipes
        generating the SWU. The class defines new variables, all of them have the prefix
        *SWUPDATE_* in the name.
meta-swupdate には、SWUpdate に固有のクラスが含まれています。
Yocto 内に構築されたイメージから SWU イメージを生成するのに役立ちます。
すべてのコンポーネント、つまり SWU イメージの一部であるアーティファクトが Yocto の deploy ディレクトリに存在する必要があります。
このクラスは、SWU を生成するレシピによって継承される必要があります。
このクラスは新しい変数を定義します。それらはすべて、名前にプレフィックス *SWUPDATE_* が含まれています。

..
        - **SWUPDATE_IMAGES** : this is a list of the artifacts to be packaged together.
        The list contains the name of images without any extension for MACHINE or
        filetype, that are added automatically.
        Example :
- **SWUPDATE_IMAGES** : これは一緒にパッケージ化されるアーティファクトのリストです。
  このリストには、MACHINE またはファイルタイプの拡張子なしで、自動的に追加されるイメージの名前が含まれています。
  例  :


::

        SWUPDATE_IMAGES = "core-image-full-cmdline uImage"

..
        - **SWUPDATE_IMAGES_FSTYPES** : extension of the artifact. Each artifact can
        have multiple extension according to the IMAGE_FSTYPES variable.
        For example, an image can be generated as tarball and as UBIFS for target.
        Setting the variable for each artifact tells the class which file must
        be packed into the SWU image.
- **SWUPDATE_IMAGES_FSTYPES** : アーティファクトの拡張。
  各成果物は、IMAGE_FSTYPES 変数に従って複数の拡張子を持つことができます。
  たとえば、イメージはターゲットの tarball および UBIFS として生成できます。
  各アーティファクトの変数を設定すると、どのファイルを SWU イメージにパックする必要があるかがクラスに通知されます。


::

        SWUPDATE_IMAGES_FSTYPES[core-image-full-cmdline] = ".ubifs"

..
        - **SWUPDATE_IMAGES_NOAPPEND_MACHINE** : flag to use drop the machine name from the
        artifact file. Most images in *deploy* have the name of the Yocto's machine in the
        filename. The class adds automatically the name of the MACHINE to the file, but some
        artifacts can be deployed without it.
- **SWUPDATE_IMAGES_NOAPPEND_MACHINE** : アーティファクト ファイルからマシン名を削除するために使用するフラグ。
  *deploy* のほとんどのイメージには、ファイル名に Yocto のマシンの名前が含まれています。
  このクラスは MACHINE の名前をファイルに自動的に追加しますが、一部のアーティファクトは MACHINE なしでデプロイできます。


::

        SWUPDATE_IMAGES_NOAPPEND_MACHINE[my-image] = "1"

- **SWUPDATE_SIGNING** : if set, the SWU is signed. There are 3 allowed values:
  RSA, CMS, CUSTOM. This value determines used signing mechanism.
- **SWUPDATE_SIGN_TOOL** : instead of using openssl, use SWUPDATE_SIGN_TOOL to sign
  the image. A typical use case is together with a hardware key. It is
  available if SWUPDATE_SIGNING is set to CUSTOM
- **SWUPDATE_PRIVATE_KEY** : this is the file with the private key used to sign the
  image using RSA mechanism. Is available if SWUPDATE_SIGNING is set to RSA.
- **SWUPDATE_PASSWORD_FILE** : an optional file containing the password for the private
  key. It is available if SWUPDATE_SIGNING is set to RSA.
- **SWUPDATE_CMS_KEY** : this is the file with the private key used in signing
  process using CMS mechanism. It is available if SWUPDATE_SIGNING is set to
  CMS.
- **SWUPDATE_CMS_CERT** : this is the file with the certificate used in signing
  process using CMS method. It is available if SWUPDATE_SIGNING is
  set to CMS.

- **SWUPDATE_AES_FILE** : this is the file with the AES password to encrypt artifact. A new `fstype` is
  supported by the class (type: `enc`). SWUPDATE_AES_FILE is generated as output from openssl to create
  a new key with

  ::

                openssl enc -aes-256-cbc -k <PASSPHRASE> -P -md sha1 -nosalt > $SWUPDATE_AES_FILE

  To use it, it is enough to add IMAGE_FSTYPES += "enc" to the  artifact. SWUpdate supports decryption of
  compressed artifact, such as

  ::

        IMAGE_FSTYPES += ".ext4.gz.enc"


..
        Automatic sha256 in sw-description
sw-description の自動 sha256
----------------------------------

..
        The swupdate class takes care of computing and inserting sha256 hashes in the
        sw-description file. The attribute *sha256* **must** be set in case the image
        is signed. Each artifact must have the attribute:
swupdate クラスは、sha256 ハッシュの計算と sw-description ファイルへの挿入を処理します。
イメージが署名されている場合は、属性 *sha256* を設定 **しなければなりません**。
各アーティファクトには次の属性が必要です。

::

        sha256 = "$swupdate_get_sha256(artifact-file-name)"

..
        For example, to add sha256 to the standard Yocto core-image-full-cmdline:
たとえば、標準の Yocto core-image-full-cmdline に sha256 を追加するには、次のようにします。

::

        sha256 = "$swupdate_get_sha256(core-image-full-cmdline-machine.ubifs)";


..
        The name of the file must be the same as in deploy directory.
ファイルの名前は、デプロイ ディレクトリと同じにする必要があります。

BitBake variable expansion in sw-description
--------------------------------------------

To insert the value of a BitBake variable into the update file, pre- and
postfix the variable name with "@@".
For example, to automatically set the version tag:

::

        version = "@@DISTRO_VERSION@@";

Automatic versions in sw-description
------------------------------------

By setting the version tag in the update file to `@SWU_AUTO_VERSION` it is
automatically replaced with `PV` from BitBake's package-data-file for the package
matching the name of the provided filename tag.
For example, to set the version tag to `PV` of package `u-boot`:

::

        filename = "u-boot";
        ...
        version = "@SWU_AUTO_VERSION";

Since the filename can differ from package name (deployed with another name or
the file is a container for the real package) you can append the correct package
name to the tag: `@SWU_AUTO_VERSION:<package-name>`.
For example, to set the version tag of the file `packed-bootloader` to `PV` of
package `u-boot`:

::

        filename = "packed-bootloader";
        ...
        version = "@SWU_AUTO_VERSION:u-boot";

To automatically insert the value of a variable from BitBake's package-data-file
different to `PV` (e.g. `PKGV`) you can append the variable name to the tag:
`@SWU_AUTO_VERSION@<package-data-variable>`.
For example, to set the version tag to `PKGV` of package `u-boot`:

::

        filename = "u-boot";
        ...
        version = "@SWU_AUTO_VERSION@PKGV";

Or combined with a different package name:

::

        filename = "packed-bootloader";
        ...
        version = "@SWU_AUTO_VERSION:u-boot@PKGV";

Using checksum for version
--------------------------

It is possible to use the hash of an artifact as the version in order to use
"install-if-different".  This allows versionless artifacts to be skipped if the
artifact in the update matches the currently installed artifact.

In order to use the hash as the version, the sha256 hash file placeholder
described above in Automatic sha256 in sw-description must be used for version.

Each artifact must have the attribute:

::

        version = "@artifact-file-name"

The name of the file must be the same as in deploy directory.

Template for recipe using the class
-----------------------------------

::

        DESCRIPTION = "Example recipe generating SWU image"
        SECTION = ""

        LICENSE = ""

        # Add all local files to be added to the SWU
        # sw-description must always be in the list.
        # You can extend with scripts or wahtever you need
        SRC_URI = " \
            file://sw-description \
            "

        # images to build before building swupdate image
        IMAGE_DEPENDS = "core-image-full-cmdline virtual/kernel"

        # images and files that will be included in the .swu image
        SWUPDATE_IMAGES = "core-image-full-cmdline uImage"

        # a deployable image can have multiple format, choose one
        SWUPDATE_IMAGES_FSTYPES[core-image-full-cmdline] = ".ubifs"
        SWUPDATE_IMAGES_FSTYPES[uImage] = ".bin"

        inherit swupdate

..
        Simplified version for just image
イメージだけの簡易版
---------------------------------

..
        In many cases there is a single image in the SWU. This is for example when
        just rootfs is updated. The generic case described above required an additional
        recipe that must be written and maintained. For this reason, a simplified version
        of the class is introduced that allowed to build the SWU from the image recipe.
多くの場合、SWU には単一のイメージがあります。
これは、たとえば rootfs だけが更新された場合です。
上記の一般的なケースでは、追加のレシピを作成して維持する必要がありました。
このため、イメージ レシピから SWU を構築できるクラスの簡略化されたバージョンが導入されています。 

..
        Users just need to import the `swupdate-image` class. This already sets some variables.
        A sw-description must still be added into a `files` directory, that is automatically searched by the class.
        User still needs to set SWUPDATE_IMAGE_FSTYPES[`your image`] to the fstype that should be packed
        into the SWU - an error is raised if the flag is not set.
ユーザーは `swupdate-image` クラスをインポートするだけです。
これにより、すでにいくつかの変数が設定されています。
sw-description は、クラスによって自動的に検索される `files` ディレクトリに追加する必要があります。
ユーザーは SWUPDATE_IMAGE_FSTYPES[`your image`] を SWU にパックする必要がある fstype に設定する必要があります。
フラグが設定されていない場合はエラーが発生します。

..
        In the simple way, your recipe looks like
簡単な方法で、あなたのレシピは次のようになります

::
        <your original recipe code>

        SWUPDATE_IMAGES_FSTYPES[<name of your image>] = <fstype to be put into SWU>
        inherit swupdate-image
