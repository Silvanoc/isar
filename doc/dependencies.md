# Dependency management in ISAR

## Introduction

The typical goal of an ISAR build is to obtain:

1. a disk image containing one or more partition images (all optional),
1. containing one or more Debian root filesystems with
1. certain Debian packages installed.

Reaching this goal relies on the availability of the above listed elements on *runtime*, the specification of the **[runtime environments](#runtime-environment-specification)** helps the different build tools ensuring the availability of the mentioned elements on runtime.

Usually some of the runtime dependencies have to be built, and it happens in special environments. These **[buildtime environments](#buildtime-environment-specification)** help the different build tools preparing the environment for building the runtime dependencies.

The differentiation between buildtime and runtime environments is very important.
The fact that ISAR uses Debian both for buildtime and runtime environments might mislead some people.
In general the names of the variables being used to specify those dependencies aren't supporting this differentiation, but difficulting it.

Not only the build environments have to be specified, but also the **[build process](#build-process-specification)** that takes care of building and integrating both the runtime dependencies and the buildtime dependencies.

## Runtime environment specification

As mentioned above, ISAR can build [partitioned disk images](#disk-image) or only [root filesystems](#root-filesystem).

Let's first analyze how the content of the different runtime containers can be specified from bigger to smaller container.

```
+---------------------------------------------------------------------------------------------------+
|                                                                                                   |
|    DISK IMAGE                                                                                     |
|                                                                                                   |
|   +------------+   +-----------------------------------+   +------------------------------+       |
|   |            |   |                                   |   |                              ||      |
|   | Bootloader |   | Bootable partition                |   | Other partitions             |||     |
|   |            |   |                                   |   |                              ||||    |
|   |            |   | +------------------------------+  |   |                              ||||    |
|   |            |   | |                              |  |   |                              ||||    |
|   |            |   | |  ROOT FILESYSTEM             |  |   |                              ||||    |
|   +------------+   | |                              |  |   |                              ||||    |
|                    | |    +-------------------+     |  |   |                              ||||    |
|                    | |    |                   | |   |  |   |                              ||||    |
|                    | |    |  DEBIAN PACKAGES  | |   |  |   |                              ||||    |
|                    | |    |                   | |   |  |   |                              ||||    |
|                    | |    |                   | |   |  |   |                              ||||    |
|                    | |    |                   | |   |  |   |                              ||||    |
|                    | |    +-------------------+ |   |  |   |                              ||||    |
|                    | |     ---------------------+   |  |   |                              ||||    |
|                    | |                              |  |   +------------------------------+|||    |
|                    | +------------------------------+  |    -------------------------------+||    |
|                    |                                   |    --------------------------------+|    |
|                    +-----------------------------------+     --------------------------------+    |
|                                                                                                   |
+---------------------------------------------------------------------------------------------------+
```

### Disk image

A disk image depends on:

1. a bootloader (optional) and
1. one or more partitions, containing one of them a [root filesystem](#root-filesystem).

The ISAR way to **specify** them is through *WKS files* (OpenEmbedded KickStart files).
ISAR doesn't change it in any significant way, therefore the corresponding [Yocto documentation][wks] provides the best reference.

### Root filesystem

A root filesystem depends on:

1. a minimal set of files (the result of `debootstrap`) and
1. additional files and configurations obtained from *Debian packages*.

The ISAR way to **specify** them is quite complex and involves *ISAR variables* that can be found in configuration files or recipes and artifacts and metadata being provided by *Debian packages*.

See following sections for more information about how to specify the content of the root filesystem:

* [ISAR `IMAGE_INSTALL` variable](#isar-image_install-variable)
* [ISAR `IMAGE_PREINSTALL` variable](#isar-image_preinstall-variable)
* [ISAR `DEBIAN_DEPENDS` variable](#isar-debian_depends-variable)
* [Debian `Depends` control-file value](#debian-depends-control-file-value)

## Buildtime environment specification

The creation of the [above mentioned runtime units](#runtime-environment-specification) (disk images, bootloaders, root filesystems, packages, ...) takes place in different environments.

These environments have to be specified.

### Buildchroot

The *Debian packages* are built in the so-called *buildchroot* filesystem.

It's a Debian chroot that is used to build custom Debian packages required by the **[root filesystem](#root-filesystem)**.
This build environment depends on the presence of certain artifacts (compilers, configurations, linkers, ...).

The ISAR way to **specify** them is quite complex and involves *ISAR variables* that can be found in configuration files or recipes and artifacts and metadata being provided by *Debian packages*.

See following sections for more information about how to specify the content of the chroot:

* [ISAR `IMAGER_INSTALL` variable](#isar-image_install-variable)
* [ISAR `BUILDCHROOT_PREINSTALL` variable](#isar-buildchroot_preinstall-variable)
* [Debian `Build-Depends` control-file value](#debian-build-depends-control-file-value)

If a disk image is being created, the same environment is being used for building the image.
For this usage additional dependencies might appear. See the [imager](#imager) section for more information.

### Imager

The *disk images* are built in the so-called *imager*.

It's a Debian chroot that is used to build **[disk images](#disk-image)**.
This build environment depends on the presence of certain artifacts (compilers, configurations, linkers, ...).
Although conceptually the imager doesn't have anything to do with the buildchroot, in the implementation the same chroot is being used for both tasks.

The ISAR way to **specify** them involves *ISAR variables* that can be found in configuration files or recipes.

See following sections for more information about how to specify the content of the chroot:

* [ISAR `WIC_IMAGER_INSTALL` variable](#isar-wic_imager_install-variable)
* [ISAR `IMAGER_BUILD_DEPS` variable](#isar-imager_build_deps-variable)

### Target filesystem

It's a Debian root filesystem that results from creating a minimalistic Debian skeleton via `debootstrap` and installing Debian packages on it, resulting in the [root filesystem](#root-filesystem).

There's no need to **specify** this build environment, since it's the [root filesystem of the runtime environment](#root-filesystem).

## Build process specification

ISAR relies on a subset of Bitbake to take care of the build process.
Therefore the ISAR way to **specify** the different tasks involved in the process is Bitbake recipes with some *Bitbake variables* (well documented ) and some *ISAR variables* (some of them are only syntactic sugar).

See following sections for more information about how to specify inter-tasks dependencies:

* [Bitbake `DEPENDS` variable](#bitbake-depends-variable)
* [ISAR `IMAGE_INSTALL` variable](#isar-image_install-variable)
* [ISAR `IMAGER_BUILD_DEPS` variable](#isar-imager_build_deps-variable)

## Mapping: Variables <=> Build environment and Stages

| Variable | Environment | Stage |
|:-------- |:-----------------:|:-----:|
| [`DEPENDS`](#bitbake-depends-variable) | | All |
| [`IMAGE_INSTALL`](#isar-image_install-variable) | [Root filesystem](#root-filesystem) | [Root filesystem](#root-filesystem) preparation and build |
| [`IMAGE_PREINSTALL`](#isar-image_preinstall-variable) | [Root filesystem](#root-filesystem) | |
| [`DEBIAN_DEPENDS`](#isar-debian_depends-variable) | [Root filesystem](#root-filesystem) | |
| [`Depends` control-file](#debian-depends-control-file-value) | [Root filesystem](#root-filesystem) | |
| [`IMAGE_TRANSIENT_PACKAGES`](#isar-image_transient_packages-variable) | [Root filesystem](#root-filesystem) | [Root filesystem](#root-filesystem) preparation and build |
| [`BUILDCHROOT_PREINSTALL`](#isar-buildchroot_preinstall-variable) | [Buildchroot](#buildroot) | |
| [`Build-Depends` control-file](#debian-build-depends-control-file-value) | [Buildchroot](#buildroot) | |
| [`IMAGER_BUILD_DEPS`](#isar-imager_build_deps-variable) | | [Imager](#imager) preparation |
| [`IMAGER_INSTALL`](#isar-imager_install-variable) | [Imager](#imager) | |
| [`WIC_IMAGER_INSTALL`](#isar-wic_imager_install-variable) | [Imager](#imager) | |

## Dependencies specification variables

### Bitbake `DEPENDS` variable

This is a **Bitbake** variable that specifies **Bitbake inter-task dependencies**.

Since it specifies the recipes that another recipe depend upon, this variable can be specified in **package recipes** and **image recipes**.

### ISAR `IMAGE_INSTALL` variable

This is an **ISAR-only** variable that specifies at the same time **image custom Debian package dependencies** and **Bitbake inter-task dependencies**.

It basically specifies that a **[root filesystem](#root-filesystem)** (the prefix 'IMAGE' can be misleading here) requires a custom Debian package.
But since custom Debian packages are being built from package recipes, this variable also specifies that an image recipe depends on package recipes.

For this "magic" to work both the Debian package and the package recipe have to have the same name.
Something else than this 1:1 behaviour (e.g. a recipe creating multiple Debian binary packages out of a single Debian source package) requires some hacking out of the scope of this document.

This variable can be found in **configuration files** and **image recipes**.

### ISAR `IMAGE_PREINSTALL` variable

This is an **ISAR-only** variable that specifies **image upstream Debian package dependencies**.

It specifies that a **[root filesystem](#root-filesystem)** requires certain *upstream Debian packages*, being "upstream Debian packages" ready built packages that can be downloaded from a Debian package repository.

This variable can be found in **configuration files** and **image recipes**.

### ISAR `DEBIAN_DEPENDS` variable

This is an **ISAR-only** variable that specifies **Debian inter-package dependencies** and can be used only on package recipes inheriting from the **`dpkg-raw` class**.

It lists the Debian binary packages that a custom Debian package depends upon.
It can be used for both upstream and custom packages.
It results in an entry in the corresponding **[Debian `Depends` control-file value](#debian-depends-control-file-value)**.

A common usage is for creating meta-packages that only keep dependencies on further packages synchronized.

This variable can be found either in **package recipes**.

### ISAR `IMAGE_TRANSIENT_PACKAGES` variable

This is an **ISAR-only** variable that specifies **image custom Debian package dependencies** and **Bitbake inter-task dependencies**.

This is a variable that is rarely needed and only for a very specific use case!
That is when a package is needed in the **[root filesystem](#root-filesystem)** during the **root filesystem build stage**, but not in the final **[root filesystem](#root-filesystem)**.
Meaning that the installation is only transient, therefore its name.

This variable can be found in **configuration files** and **image recipes**.

### Debian `Depends` control-file value

This is an **Debian** variable that specifies **Debian inter-package dependencies**.

`Depends` specifies the Debian packages that are required in the [runtime environment](#runtime-environment-specification) by the package declaring the dependencies.

It relies on standard Debian mechanisms, therefore the corresponding [official Debian documentation][pkg-rels] provides the best reference.

### ISAR `BUILDCHROOT_PREINSTALL` variable

This is an **ISAR-only** variable that specifies **buildchroot Debian package dependencies**.

It specifies that **[buildchroot](#buildroot)** requires certain *Debian packages*.

This variable can be found in **buildchroot recipes**.

### Debian `Build-Depends` control-file value

This is an **Debian** variable that specifies **Debian inter-package dependencies**.

`Build-Depends` specifies the Debian packages that are required in the [buildtime environment](#buildtime-environment-specification) to build the package declaring the dependencies.

It relies on standard Debian mechanisms, therefore the corresponding [official Debian documentation][pkg-rels] provides the best reference.

### ISAR `IMAGER_BUILD_DEPS` variable

This is an **ISAR-only** variable that specifies **Bitbake inter-task dependencies**.

It has exactly the same effect as `DEPENDS`.
It simply provides syntactic sugar to separate dependencies on recipes for building the runtime environment from dependencies on recipes for building the **[imager](#imager)**.

This variable can be found in **configuration files**.

### ISAR `IMAGER_INSTALL` variable

This is an **ISAR-only** variable that specifies **imager Debian package dependencies**.

It specifies that certain *Debian packages* have to be installed on the **[imager](#imager)**.

This variable can be found typically in **configuration files**, although it could also be part of **image recipes**.

### ISAR `WIC_IMAGER_INSTALL` variable

This is an **ISAR-only** variable that specifies **imager Debian package dependencies**.

It specifies that an **[imager](#imager)** requires certain *Debian packages* so that WIC can build the disk images (e.g. `parted`).

This variable can be found typically in **configuration files**, although it could also be part of **image recipes**.

[wks]: https://www.yoctoproject.org/docs/current/ref-manual/ref-manual.html#ref-kickstart "OpenEmbedded Kickstart (.wks) Reference"
[pkg-rels]: https://www.debian.org/doc/debian-policy/ch-relationships.html "Debian policy: Declaring relationships between packages"
