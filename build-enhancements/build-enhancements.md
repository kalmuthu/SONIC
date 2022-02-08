

# Build Improvements HLD

#### Rev 0.2

# Table of Contents

- [List of Tables](#list-of-tables)
- [Revision](#revision)
- [Definition/Abbreviation](#definitionabbreviation)
- [About This Manual](#about-this-manual)
- [1 Introduction and Scope](#1-introduction-and-scope)
  - [1.1 Current build infrastructure](#11-existingtools-limitation)
  - [1.2 Benefits of this feature](#12-benefits-of-this-feature)
- [2 Feature Requirements](#2-feature-requirements)
  - [2.1 Functional Requirements](#21-functional-requirements)
  - [2.2 Configuration and Management Requirements](#22-configuration-and-management-requirements)
  - [2.3 Scalability Requirements](#23-scalability-requirements)
  - [2.4 Warm Boot Requirements](#24-warm-boot-requirements)
- [3 Feature Description](#3-feature-description)
- [4 Feature Design](#4-feature-design)
  - [4.1 Overview](#41-design-overview)
  - [4.2 Docker-in-Docker build](#42-db-changes)
  - [4.3 SONIC version cache build](#42-db-changes)
  - [4.4 DPKG cache for image build](#42-db-changes)
- [5 CLI](#5-cli)
  - [5.1 Output Format](#51-cli-output-format)
- [6 Serviceability and Debug](#6-serviceability-and-debug)
- [7 Warm reboot Support](#7-warm-reboot-support)
- [8 Unit Test Cases ](#8-unit-test-cases)
- [9 References ](#9-references)

# List of Tables

[Table 1: Abbreviations](#table-1-abbreviations)

# Revision
| Rev  |    Date    |       Author        | Change Description                                           |
|:--:|:--------:|:-----------------:|:------------------------------------------------------------:|
| 0.1  |  | Kalimuthu Velappan      | Initial version                                              |


# Definition/Abbreviation

### Table 1: Abbreviations

| **Term** | **Meaning**                               |
| -------- | ----------------------------------------- |
| DPKG     | Debian Package                            |


# About this Manual

This document provides general information about the build improvements in SONiC. 


# 1 Introduction and Scope

This document describes the Functionality and High level design of the build improvement in SONiC.

The current SONiC build infrastructure generats all the SONiC build artifacts inside the docker container environment. When docker is
isolated from the host CPU, the docker resource usage and filesystem access are restricted from its full capacity. Docker isolation is more
essential for application containers, but for the build containers, the more essentail requirement is the build performace rather
than adopting a higher security model. It provides the better build performance when the build containers are run in native mode. 

This feature provides improvements three essential areas.
 - Build container creation using docker native mode.
 - Package cache support for source code componets.
 - Package cache support for binary image componets. 


## 1.1 Limitation of Existing tools:
 - Monit tool is a poll based approach which monitors the configured services for every 1 minute.
 - Monit tools feature of critical process monitoring was deprecated as supervisor does the job. Hence system-health tool which depends on monit does not work.
 - Container_checker in monit checks only for running status of expected containers.
 - Monits custom script execution can only run a logic to take some action but it is yet again a poll based approach.


## 1.2 Benefits of this feature:
 - Event based model - which does not hog cpu unlike poll based approach.
 - Know the overall system status through syslog and as well through CLIs
 - CLIs just connect to the backend daemon to fetch the details. No redundant codes.
 - Combatibility with application extension framework.
    SONiC package installation process will register new feature in CONFIG DB.
    Third party dockers(signature verified) gets integrated into sonic os and runs similar to the existing dockers accessing db etc.
    Now, once the feature is enabled, it becomes part of either sonic.target or multi-user.target and when it starts, it automatically comes under the system monitor framework watchlist.
    However, app ready status for those dockers cant be tracked unless they comply with the framework logic.
    Hence any third party docker needs to follow the framework logic by including "check_up_status" field while registering itself in CONFIG_DB and also make use of the provision given to docker apps to mark its closest up status in STATE_DB.

    


# 2 Feature Requirements

## 2.1 Functional Requirements

Following requirements are addressed by the design presented in this document:

1. Add a feature in the build infra to support the build container in both the modes
	- Docker-In-Docker
    - Native docker mode
2. Add caching support for all the SONiC version components.
3. Build optimizatoin for binary image generation.
4. Add caching support for binary image.
5. Add support for build time dependency over overlayFS support.


## 2.2 Configuration and Management Requirements

NA

## 2.3 Scalability Requirements

NA

## 2.4 Warm Boot Requirements

NA 


# 3 Feature Description

This feature provides build improvements in SONIC.

# 4 Feature Design
## 4.1 Design Overview

- SONiC build using Docker-In-Docker(DinD) mode.
    - The DinD mode uses dind docker solution
	- In DinD mode, major concern is about LSM (Linux Security Modules) like AppArmor and SELinux: when starting a container, the "inner
	Docker" might try to apply security profiles that will conflict or confuse the "outer Docker." This was actually the hardest problem to
	solve when trying to merge the original implementation of the -privileged flag. 
	- The second issue is linked to storage drivers. When you run Docker in Docker, the outer Docker runs on top of a normal filesystem
	(EXT4, BTRFS, etc) but the inner Docker runs on top of a copy-on-write system (AUFS, BTRFS, Device Mapper, etc., depending on
			what the outer Docker is setup to use). There are many combinations that won't work. For instance, it cannot run AUFS on top of
	AUFS and on the nested subvolumes, removing the parent subvolume will fail. 
	- Device Mapper is not namespaced, so if multiple instances of Docker use it on the same machine, they will all be able to see (and
			affect) each other's image and container backing devices.
	- There are workarounds for many of those issues; for instance, if you want to use AUFS in the inner Docker, just promote /var/lib/docker

	- There are workarounds for many of those issues; for instance, if you want to use AUFS in the inner Docker, just promote
	/var/lib/docker

	- In DinD mode, apart from the security aspect, lots of performace panaliteis are involved.
	- It uses the UnionFS/OverlayFS, which degrades the performace when number of lower layers are more. 
	- Docker 
	- There are workarounds for many of those issues; for instance, if you want to use AUFS in the inner Docker, just promote
	/var/lib/docker

- SONiC build using native docker mode.
	- The native docker mode mode uses the DoD.
    - The DoD mode uses socket file(/var/run/docker.sock) to communitcate with host dockerd daemon.
	- The SONiC uses the shared socket file between HOST and the container to run the build container.
	-   Eg: docker run -v /var/run/docker.sock:/var/run/docker.sock ...
	- When docker build or docker compose will runs in parallel with build container which is sibiliing to build container. 
	- This feature addresses the sonic target container creation in parallel that gives better performanace. 
	- In order will runs in parallel with build container which is sibiliing to build container. 
	- When docker builder/composer runs in the host dockerd, the multilevel UNIONFS/OverlayFS is not needed which gives better performance.


Currently, the build dockers are created as a user dockers(docker-base-stretch-, etc) that are
specific to each user. But the sonic dockers (docker-database, docker-swss, etc) are
created with a fixed docker name and common to all the users.

docker-database:latest
docker-swss:latest

When multiple builds are triggered on the same build server that creates parallel building issue because
all the build jobs are trying to create the same docker with latest tag.
This happens only when sonic dockers are built using native host dockerd for sonic docker image creation.

This patch creates all sonic dockers as user sonic dockers and then, while
saving and loading the user sonic dockers, it rename the user sonic
dockers into correct sonic dockers with tag as latest.

The user sonic docker names are derived from '_LOAD_DOCKER' make
variable and using Jinja template, it replaces the FROM docker name with correct user sonic docker name for
loading and saving the docker image.

The template processing covers only for common dockers, Broadcom and VS platform dockers. For other vendor specific dockers, respective vendors need to add the support.





            ----------------------------------                         ------------------------------------------- 
		   |    rules/docker-base-<dist>.mk   |---------------------->| dockers/docker-base-<dist>/Dockerfile.j2  |
            ----------------------------------                        |                                           |
                         |                                            | FROM debian:buster                        |
                         |                                             -------------------------------------------
						 V
            -----------------------------------------                  -----------------------------------------------   
		   |    rules/docker-config-<engine>.mk      |                | dockers/docker-config-<engine>/Dockerfile.j2  |
		   |                                         |--------------->|                                               |
		   |    <engine>_LOAD_DOCKERS = <dist>       |                | FROM {{ docker_config_engine_load_image }}    |      
		   | export docker_config_engine_load_image= |                |                                               |
		   |        docker-base-<dist>               |                |                                               |
            -----------------------------------------                  -----------------------------------------------  
                         |
                         |
						 V
            ---------------------------------------                   -----------------------------------------------   
		   |    rules/docker-<database>.mk         |                 | dockers/docker-config-<engine>/Dockerfile.j2  |
		   |                                       |---------------->|                                               |
		   |    <database>_LOAD_DOCKERS = <engine> |                 | FROM {{ docker_database_load_image }}         |      
		   | export docker_database_load_image=    |                 |                                               |
		   |        docker-base-<dist>             |                 |                                               |
            ---------------------------------------                   -----------------------------------------------  






## 4.2 Version cache support

Caching support added on top off verson componets.

### 4.2.1 Version components

1. Debian version cache
   - During pakcage installation, the downloaded debian packages are stored on the /var/lib/dpkg/cache folder before installation. 
   - The version cache saves all the debian package componets into version cache folder and gets restored up subsequent installation.
   - Any change in the version control files like diffent files are stored as
        1. Base docker
		2. Soinc docker
		3. Debain bootstrap packages
		4. Debian rootfs packages

	- The version control files are Dockerfile.j2 and common makefiles, version files.

2. Python version cache
   - Python cache files are stored on the pip cache folder.
   - Version control files are setup.py and common files.

3. Git clones
   - Git clones are bundled and stored into the version cache.
   - Version controls files are Makefile and clone urls, commit hash and common files.
   - Stored as bundle format.

4. Docker images
   - During sonic build, dockers are pulled from external hub and saved into version cache.
   - Version control files Dockerfile.j2 and common files.
   - Docker load and save commands are used for loading and saving the files into cache as .gz format.
   - Show command communicates with sysmonitor through this socket to get the current system status information from the dictionary maintained.
   - Sonic dockers are also cached.

5. Wget packages
   - Package that are downloaded through wget are also saved into the version cache.
   - URL and package hash are used  as version control files.
   - Stored as .gz format.

6. Go modules
   - Go modules are also cached into verson control cache.
   - mod.go and sum.go files are used as version control files.
   - Stored as cache directory and controlled through flock.

7. Bootstrap and debian package flies
   - Rootfs and bootstrap files are version cached.
   - Bulid_debian.sh and sonic_vendor_extension.sh files are version control files.

# Cache cleanup 
   - Touch is used to update the package to latest.
   - Files that are not recent, that can be cleaned up.

## 4.3 Binary Optimization

1. Binary image has two major components:
   - rootfs
   - sonic packages

1.1 Rootfs
    - Mostly packages are downoaded from the WEB.
    - Constant most of the time.
    - The rootfs the file system is created as image file system and cached as part of version cache.
    - Version control files are build_debin.sh and sonic_build_extention.

1.2 Parallen build option
	- Stage build provildes two stage build.
	- Phase 1 - Rootfs generation as part of other package generation.
	- Phase 2 - Docker generation in parallel.



## 6 Serviceability and Debug

- The system logging mechanisms explained in section 4.6 shall be used.
- The show commands can be used as debug aids.
- Techsupport:
  In generate dump tool, show system status all --detail CLI is included to be saved to the dump in the name of systemstatus.all.detail



## 8 Unit Test Cases

1. Check show system status all
2. Check show system status all --brief
3. Check show system status all --detail
4. Check show system status core
5. Make any of the docker apps down and check failed apps details are shown
6. Make any of the host apps down and check failed apps details are shown
7. Check top command for sysmonitor CPU and memory usage
8. Check syslogs for any service state change.
9. Check syslog for overall system status change.

## 9 References
NA
