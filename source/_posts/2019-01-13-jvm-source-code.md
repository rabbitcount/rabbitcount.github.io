---
title: 2019-01-13-jvm-source-code
date: 2019-01-13 11:38:58
tags:
- jvm
---


jdk源码结构
```
.
├── ASSEMBLY_EXCEPTION
├── LICENSE
├── Makefile
├── README
├── README-builds.html
├── THIRD_PARTY_README
├── common
│   ├── autoconf
│   │   ├── Makefile.in
│   │   ├── autogen.sh
│   │   ├── basics.m4
│   │   ├── basics_windows.m4
│   │   ├── boot-jdk.m4
│   │   ├── bootcycle-spec.gmk.in
│   │   ├── build-aux
│   │   │   ├── autoconf-config.guess
│   │   │   ├── config.guess
│   │   │   ├── config.sub
│   │   │   ├── install.sh
│   │   │   └── pkg.m4
│   │   ├── build-performance.m4
│   │   ├── builddeps.conf.example
│   │   ├── builddeps.conf.nfs.example
│   │   ├── builddeps.m4
│   │   ├── compare.sh.in
│   │   ├── config.h.in
│   │   ├── configure
│   │   ├── configure.ac
│   │   ├── generated-configure.sh
│   │   ├── help.m4
│   │   ├── hotspot-spec.gmk.in
│   │   ├── jdk-options.m4
│   │   ├── libraries.m4
│   │   ├── platform.m4
│   │   ├── source-dirs.m4
│   │   ├── spec.gmk.in
│   │   ├── spec.sh.in
│   │   ├── toolchain.m4
│   │   ├── toolchain_windows.m4
│   │   └── version-numbers
│   ├── bin
│   │   ├── boot_cycle.sh
│   │   ├── compare-objects.sh
│   │   ├── compare.sh
│   │   ├── compare_exceptions.sh.incl
│   │   ├── hgforest.sh
│   │   ├── hide_important_warnings_from_javac.sh
│   │   ├── logger.sh
│   │   ├── shell-tracer.sh
│   │   └── test_builds.sh
│   ├── nb_native
│   │   └── nbproject
│   │       ├── configurations.xml
│   │       └── project.xml
│   └── src
│       └── fixpath.c
├── configure
├── get_source.sh
├── make
│   ├── HotspotWrapper.gmk
│   ├── Javadoc.gmk
│   ├── Jprt.gmk
│   ├── Main.gmk
│   ├── MakeHelpers.gmk
│   ├── common
│   │   ├── CORE_PKGS.gmk
│   │   ├── IdlCompilation.gmk
│   │   ├── JavaCompilation.gmk
│   │   ├── MakeBase.gmk
│   │   ├── NON_CORE_PKGS.gmk
│   │   ├── NativeCompilation.gmk
│   │   ├── RMICompilation.gmk
│   │   └── support
│   │       ├── ListPathsSafely-post-compress.incl
│   │       ├── ListPathsSafely-pre-compress.incl
│   │       ├── ListPathsSafely-uncompress.sed
│   │       └── unicode2x.sed
│   ├── devkit
│   │   ├── Makefile
│   │   └── Tools.gmk
│   ├── jprt.properties
│   ├── scripts
│   │   ├── hgforest.sh
│   │   ├── lic_check.sh
│   │   ├── normalizer.pl
│   │   ├── update_copyright_year.sh
│   │   └── webrev.ksh
│   └── templates
│       ├── bsd-header
│       ├── gpl-cp-header
│       └── gpl-header
└── test
    └── Makefile
```