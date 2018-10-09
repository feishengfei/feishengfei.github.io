---
title: "Introduction to YOCTO project"
date: "2018-10-09 20:28:10 +0800"
toc: true
category: work
tags: [yocto, bitbake]
classes: wide
---
## Reference

* [1 yocto_official_overview](https://www.yoctoproject.org/software-overview/)
* [2 slide of intro_on YOCTO](http://opensourceday.org/slides/OSD2014-Introduction_to_Yocto_Light.pdf)
* [3 slide of yocto](https://bootlin.com/doc/training/yocto/yocto-slides.pdf)
  [Bootlin](https://git.bootlin.com/training-materials/)

## What is the Yocto Project

* An open source project hosted at The Linux Foundation
* A collection of:
	custom Linux-based embedded OS
	tools for automated building & testing
	processes for board support & license compliance
* A reference embedded distribution(called Poky)

**“Is not an Embedded Linux Distribution - It creates a custom one for You!”**

## Goals

* Help to build a Linux distribution for embedded systems
* Imporving the software development process for embedded Linux distributions

## Features

* Widely Adopted Across the Industry
* Architecture Agnostic
* Collaboration of thousands of developers worldwide
* Images & Code Transfer Easily
* Flexibility
* Ideal for Constrained Embedded and IoT devices
* Comprehensive Toolchain Capabilities
* Mechanism Rules Over Policy
* Uses a Layer Model
* Supports Partial Builds
* Releases According to a Strict Schedule
* Rich Ecosystem of Individuals and Organizations
* Binary Reproducibility
* License Manifest

## Core components

* BitBake, the *build engine*. It is a task scheduler, like `make`. It interprets configuration files and recipes (also called *metadata*) to perform a set of tasks, to download, configure and build specified applications and filesystem images.
* OpenEmbedded-Core, a set of base *layers*. It is a set of recipes, layers and classes which are shared between all OpenEmbedded based systems.
* Poky, the *reference system*. It is a collection of projects and tools, used to bootstrap a new distribution based on the Yocto Project.

## Build System:Poky

Poky consist of:
* Bitbake

	execute and manage all the build steps

* Metadata

	task definitions
	1. Configuration(.conf): global definition of variables
	2. Classes(.bbclass): define the build logic, the packaging...
	3. Recipes(.bb): define the individual piece of software/image to be build

## Recipes

Contains the following metadata:

* Repository or Path of the source code of the packages to build
* Patches to apply
* Dependencies from other recipes or from libraries
* Configuration and compilation options
* Define the packages to create and what files goes into the packages.

## Recipes build process

Bitbake build a recipe folloing this steps:

* fetch and unpack:
	1. can get the source files form tarballs, git, svn, etc.
	2. the source files are extracted into the work directory.
	3. and packed into download directory for future builds.
* patch:
	the extracted source files are then patched
* configure and install
	1. many standard build rules are available as autotools, cmake, gettext
	2. put the build into the staging area
* package generation
	1. create packages for dev, docs, locales
	2. support the formats ipk, Debian, RPM

## WORKFLOW

![workflow](/assets/images/yocto/yp-how-it-works-new-diagram.png)

## Layers

1. **Metadata (.bb + Patches):** Software layers containing user-supplied recipe files, patches, and append files.
2. **Machine BSP Configuration:** Board Support Package (BSP) layers (i.e. "BSP Layer" in the following figure) providing machine-specific configurations.
3. **Policy Configuration:** Distribution Layers (i.e. "Distro Layer" in the following figure) providing top-level or general policies for the images or SDKs being built for a particular distribution. 

![layer input](/assets/images/yocto/layer-input.png)

## Poky reference system overview

![poky ref overview](/assets/images/yocto/user-configuration.png)

**bitbake/** Holds all scripts used by the BitBake command.
	Usually matches the stable release of the BitBake project.

**documentation/** All documentation sources for the Yocto Project documentation. Can be used to generate nice PDFs.

**meta/** Contains the OpenEmbedded-Core metadata.

**meta-skeleton/** Contains template recipes for BSP and kernel development.

**meta/conf** Core set of configuration files

**meta/classes** Contains the *.bbclass files that are used to abstract common code so it can be reused by multiple packages.

**meta/recipes-\*** Core recipes.

**meta-poky/** Holds the configuration for the Poky reference distribution.

**meta-selftest/** Used to verify the behavior of the build system.

**meta-skeleton/** Template recipes for BSP and kernel development.

**meta-yocto-bsp/** Configuration for the Yocto Project reference hardware board support package.

**LICENSE** The license under which Poky is distributed (a mix of GPLv2 and MIT).

**oe-init-build-env** Script to set up the OpenEmbedded build environment. It will create the build directory. It takes an optional parameter which is the build directory name. By default, this is build. This script has to be sourced because it changes environment variables.

**scripts** Contains scripts used to set up the environment, development tools, and tools to flash the generated images on the target.

## TERMS FOR REFERENCE
**Configuration Files**: Files which hold global definitions of variables, user defined variables and hardware configuration information. They tell the build system what to build and put into the image to support a particular platform.

**Recipe**: The most common form of metadata. A recipe will contain a list of settings and tasks (instructions) for building packages which are then used to build the binary image. A recipe describes where you get source code and which patches to apply. Recipes describe dependencies for libraries or for other recipes, as well as configuration and compilation options. They are stored in layers.

**Layer**: A collection of related recipes. Layers allow you to consolidate related metadata to customize your build, and isolate information for multiple architecture builds. Layers are hierarchical in their ability to override previous specifications. You can include any number of available layers from the Yocto Project and customize the build by adding your layers after them. The Layer Index is searchable for layers within Yocto Project.

**Metadata**: A key element of the Yocto Project is the meta-data which is used to construct a Linux distribution, contained in the files that the build system parses when building an image. In general, Metadata includes recipes, configuration files and other information refering to the build instructions themselves, as well as the data used to control what things get built and to affect how they are built. The meta-data also includes commands and data used to indicate what versions of software are used, and where they are obtained from, as well as changes or additions to the software itself (patches or auxiliary files) which are used to fix bugs or customize the software for use in a particular situation. OpenEmbedded Core is an important set of validated metadata.

**OpenEmbedded-Core**: oe-core is meta-data comprised of foundation recipes, classes and associated files that are meant to be common among many different OpenEmbedded-derived systems, including the Yocto Project. It is a curated subset of an original repository developed by the OpenEmbedded community which has been pared down into a smaller, core set of continuously validated recipes resulting in a tightly controlled and an quality-assured core set of recipes.

**Poky**: A reference embedded distribution and a reference test configuration created to 

1. provide a base level functional distro which can be used to illustrate how to customize a distribution, 
2. to test the Yocto Project components, Poky is used to validate Yocto Project, and
3. as a vehicle for users to download Yocto Project. Poky is not a product level distro, but a good starting point for customization. Poky is an integration layer on top of oe-core.

**Build System - "Bitbake"**: a scheduler and execution engine which parses instructions (recipes) and configuration data. It then creates a dependency tree to order the compilation, schedules the compilation of the included code, and finally, executes the building of the specified, custom Linux image (distribution).

BitBake is a make-like build tool. BitBake recipes specify how a particular package is built. They include all the package dependencies, source code locations, configuration, compilation, build, install and remove instructions. Recipes also store the metadata for the package in standard variables. Related recipes are consolidated into a layer. During the build process dependencies are tracked and native or cross-compilation of the package is performed. As a first step in a cross-build setup, the framework will attempt to create a cross-compiler toolchain (Extensible SDK) suited for the target platform.

**Packages**: The output of the build system used to create your final image.

**Extensible Software Development Kit (ESDK)**: A custom SDK for application developers that allows them to incorporate their library and programming changes back into the image to make their code available to other apps developers.

**Image**: A binary form of a Linux distribution (operating system) intended to be loaded onto a device.


