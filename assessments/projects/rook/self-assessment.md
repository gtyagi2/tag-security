# Self-assessment
The Self-assessment is the initial document for projects to begin thinking about the
security of the project, determining gaps in their security, and preparing any security
documentation for their users. This document is ideal for projects currently in the
CNCF **sandbox** as well as projects that are looking to receive a joint assessment and
currently in CNCF **incubation**.

For a detailed guide with step-by-step discussion and examples, check out the free 
Express Learning course provided by Linux Foundation Training & Certification: 
[Security Assessments for Open Source Projects](https://training.linuxfoundation.org/express-learning/security-self-assessments-for-open-source-projects-lfel1005/).

# Self-assessment outline

## Table of contents

* [Metadata](#metadata)
  * [Security links](#security-links)
* [Overview](#overview)
  * [Actors](#actors)
  * [Actions](#actions)
  * [Background](#background)
  * [Goals](#goals)
  * [Non-goals](#non-goals)
* [Self-assessment use](#self-assessment-use)
* [Security functions and features](#security-functions-and-features)
* [Project compliance](#project-compliance)
* [Secure development practices](#secure-development-practices)
* [Security issue resolution](#security-issue-resolution)
* [Appendix](#appendix)

## Metadata

A table at the top for quick reference information, later used for indexing.

|   |  |
| -- | -- |
| Software |  [Rook](https://github.com/rook/rook)  |
| Security Provider | No  |
| Languages | Go, Python, C++ |
| SBOM | [prerequisites](https://github.com/rook/rook/blob/master/Documentation/Getting-Started/Prerequisites/prerequisites.md), [go.mod](https://github.com/rook/rook/blob/master/pkg/apis/go.mod), [API go.mod](https://github.com/rook/rook/blob/master/pkg/apis/go.mod), [Build Reqs](https://github.com/rook/rook/blob/master/INSTALL.md)|
| | |

### Security links

Provide the list of links to existing security documentation for the project. You may
use the table below as an example:
| Doc | url |
| -- | -- |
| Security file | https://github.com/rook/rook/blob/master/SECURITY.md |
| Default and optional configs | https://github.com/rook/rook/blob/release-1.12/design/ceph/ceph-config-updates.md |

## Overview

Rook turns distributed storage systems into self-managing, self-scaling, self-healing storage services. It automates the tasks of a storage administrator: deployment, bootstrapping, configuration, provisioning, scaling, upgrading, migration, disaster recovery, monitoring, and resource management.

### Background

Rook is an open source **cloud-native storage orchestrator** for Kubernetes, providing the platform, framework, and support for Ceph storage to natively integrate with Kubernetes.

[Ceph](https://ceph.com/) is a distributed storage system that provides file, block and object storage and is deployed in large scale production clusters.

Rook automates deployment and management of Ceph to provide self-managing, self-scaling, and self-healing storage services. The Rook operator does this by building on Kubernetes resources to deploy, configure, provision, scale, upgrade, and monitor Ceph.

The status of the Ceph storage provider is **Stable**. Features and improvements will be planned for many future versions. Upgrades between versions are provided to ensure backward compatibility between releases.

### Actors
These are the individual parts of your system that interact to provide the 
desired functionality.  Actors only need to be separate, if they are isolated
in some way.  For example, if a service has a database and a front-end API, but
if a vulnerability in either one would compromise the other, then the distinction
between the database and front-end is not relevant.

The means by which actors are isolated should also be described, as this is often
what prevents an attacker from moving laterally after a compromise.

![Rook Components on Kubernetes](Rook%20High-Level%20Architecture.png)

- `Rook Operator`
	- `Ceph CSI Driver`
	- `Ceph Daemons`

#### `Rook Operator`
The Rook operator is a simple container that has all that is needed to bootstrap and monitor the storage cluster. The operator will start and monitor Ceph monitor pods, the Ceph OSD daemons to provide RADOS storage, as well as start and manage other Ceph daemons. The operator manages CRDs for pools, object stores (S3/Swift), and filesystems by initializing the pods and other resources necessary to run the services.

The operator will monitor the storage daemons to ensure the cluster is healthy. Ceph mons will be started or failed over when necessary, and other adjustments are made as the cluster grows or shrinks. The operator will also watch for desired state changes specified in the Ceph custom resources (CRs) and apply the changes.

Rook automatically configures the Ceph-CSI driver to mount the storage to your pods. The rook/ceph image includes all necessary tools to manage the cluster. Rook is not in the Ceph data path. Many of the Ceph concepts like placement groups and crush maps are hidden so you don't have to worry about them. Instead, Rook creates a simplified user experience for admins that is in terms of physical resources, pools, volumes, filesystems, and buckets. Advanced configuration can be applied when needed with the Ceph tools.

#### `Ceph CSI Driver`
The Ceph-CSI driver provides the provisioning and mounting of volumes

#### `Ceph Daemons`
The Ceph daemons run the core storage architecture. See the [Glossary](https://github.com/rook/rook/blob/master/Documentation/Getting-Started/glossary.md#ceph) to learn more about each daemon.

### Actions
These are the steps that a project performs in order to provide some service
or functionality.  These steps are performed by different actors in the system.
Note, that an action need not be overly descriptive at the function call level.  
It is sufficient to focus on the security checks performed, use of sensitive 
data, and interactions between actors to perform an action.  

For example, the access server receives the client request, checks the format, 
validates that the request corresponds to a file the client is authorized to 
access, and then returns a token to the client.  The client then transmits that 
token to the file server, which, after confirming its validity, returns the file.

Rook can be used to automatically configure the Ceph CSI drivers to mount the storage to an application's pods. See image at top of [Actors](#actors) for reference

##### Actors

* Rook Operator
* Ceph CSI Drivers
* Ceph Daemons

#### Workflows

##### Block Storage Example

In the diagram [here](#actors), the flow to create an application with an RWO volume is:

1. The (blue) app creates a PVC to request storage
2. The PVC defines the Ceph RBD storage class (sc) for provisioning the storage
3. K8s calls the Ceph-CSI RBD provisioner to create the Ceph RBD image.
4. The kubelet calls the CSI RBD volume plugin to mount the volume in the app
5. The volume is now available for reads and writes.

A ReadWriteOnce volume can be mounted on one node at a time.

##### Shared File System Example

In the diagram [here](#actors), the flow to create a applications with a RWX volume is:

1. The (purple) app creates a PVC to request storage
2. The PVC defines the CephFS storage class (sc) for provisioning the storage
3. K8s calls the Ceph-CSI CephFS provisioner to create the CephFS subvolume
4. The kubelet calls the CSI CephFS volume plugin to mount the volume in the app
5. The volume is now available for reads and writes.

A ReadWriteMany volume can be mounted on multiple nodes for your application to use.

##### Block Storage Example

In the diagram [here](#actors), the flow to create an application with access to an S3 bucket is:

- The (orange) app creates an ObjectBucketClaim (OBC) to request a bucket
- The Rook operator creates a Ceph RGW bucket (via the lib-bucket-provisioner)
- The Rook operator creates a secret with the credentials for accessing the bucket and a configmap with bucket information
- The app retrieves the credentials from the secret
- The app can now read and write to the bucket with an S3 client

A S3 compatible client can use the S3 bucket right away using the credentials (`Secret`) and bucket info (`ConfigMap`).


### Goals

####
Rook automates deployment and management of Ceph to provide self-managing, self-scaling, and self-healing storage services. The Rook operator does this by building on Kubernetes resources to deploy, configure, provision, scale, upgrade, and monitor Ceph.

#### Security Goals

* All access to rook operator should be authenticated and authorized
* Secrets created by Rook operator should maintain confidentiality
* Rook operator should obey the principle of least privilege (see [here](https://github.com/rook/rook/blob/release-1.12/design/ceph/security-model.md) for planned changes)
* Ceph storage elements should maintain integrity and availability while scaling

### Non-goals

* Ceph- and K8s-specific vulnerabilities
* Vulnerabilities caused by user error
* Address security issues of extensions or tools used with Rook

## Self-assessment use

This self-assessment is created by the Rook team to perform an internal analysis of the
project's security.  It is not intended to provide a security audit of Rook, or
function as an independent assessment or attestation of Rook's security health.

This document serves to provide Rook users with an initial understanding of
Rook's security, where to find existing security documentation, Rook plans for
security, and general overview of Rook security practices, both for development of
Rook as well as security of Rook.

This document provides the CNCF TAG-Security with an initial understanding of Rook
to assist in a joint-assessment, necessary for projects under incubation.  Taken
together, this document and the joint-assessment serve as a cornerstone for if and when
Rook seeks graduation and is preparing for a security audit.

## Security functions and features

### Critical
#### User ID Mapping through Rook for CephNFS Clusters
User ID mapping is a critical security component facilitated by Rook for CephNFS clusters. CephNFS allows access to objects stored in Ceph clusters through the Network File System (NFS). Rook ensures secure user domain association, linking user authentication and authorization. By enforcing authentication mechanisms, Rook guarantees that only authorized users with valid credentials can access CephNFS clusters. This measure is crucial in preventing unauthorized data leakage or modifications.
#### User Authentication between Rook CephNFS Servers and NFS Clients
Rook leverages Kerberos for user authentication between CephNFS servers and NFS clients. Through the use of configuration files and keytab files, Kerberos establishes a secure connection between the NFS server and the Kerberos server, ensuring authenticated and controlled access. This safeguards against unauthorized access, providing a secure communication channel.
#### Object Storage Daemon Encryption Capability
Rook enhances security by providing encryption capabilities for Object Storage Daemons (OSDs). OSDs can be encrypted with keys stored in a Kubernetes Secret or managed by a Key Management System. Rook supports authentication of Key Management Systems through token-based or Vault Kubernetes native authentication, adding an extra layer of security to OSD encryption.

### Security Relevant
#### Server Side Encryption in Ceph RADOS Gateway
Ceph RADOS Gateway (RGW), a pivotal component of Ceph storage, offers object storage services with a RESTful API. It supports Server Side Encryption with the flexibility to manage encryption keys either within RGW or through a Key Management System. Users have the autonomy to configure their preferences, empowering them to align encryption practices with their security policies.

#### NFS Cluster Security Specification
The security configuration of NFS clusters involves a range of customizable settings, including principal name, domain name, Kerberos configuration files, Kerberos keytab file, and System Security Services Daemon (SSSD) settings. Users can fine-tune security parameters such as sidecar image, configuration file, volume source, additional files, debug level for SSSD, and Kubernetes resource requests. This customization capability empowers users to tailor security settings for NFS clusters based on their specific requirements.

#### preservePoolsOnDelete for Object Stores in Pools
The setting "preservePoolsOnDelete" plays a critical role in determining the fate of pools used to store objects when the objects are deleted. Pools, being repositories of settings and data, are safeguarded from accidental loss. This security measure prevents users from unintentionally losing critical information, enhancing the overall security of the system.

#### preserveFilesystemOnDelete for File Systems in Ceph
The "preserveFilesystemOnDelete" setting governs whether the underlying filesystem remains intact or is deleted when a Ceph File System (CephFS) is deleted. This security setting acts as a protective measure, ensuring that data is not accidentally or unintentionally lost during filesystem deletion operations. It adds an additional layer of security to prevent data loss incidents.

## Project compliance

* Compliance.  List any security standards or sub-sections the project is
  already documented as meeting (PCI-DSS, COBIT, ISO, GDPR, etc.).

## Secure development practices

* Development Pipeline.  
Rook's development pipeline is designed to ensure that the software is robust, reliable, and secure. It involves several stages of testing and assessment as the software is developed and built.

#### Contributor Requirements

Contributors to Rook are required to sign their commits, adhering to the Developer Certificate of Origin (DCO). This practice ensures the integrity of the code by verifying that the changes are made by the person who claims to have made them. Contributors use the Signed-off-by line in commit messages to signify their adherence to these requirements. Git has a -s command-line option to append this automatically to commit messages
Rook leverages a DCO bot to enforce the DCO on each pull request and branch commits. This bot helps ensure that all contributions are properly signed off.
Contributors can get started by forking the repository on GitHub, reading the installation document for build and test instructions, and playing with the project 

#### Container Images

The container images used in Rook are immutable, which means they cannot be changed after they are created. This practice enhances the security of the software by preventing unauthorized modifications.

#### Reviewers

Before a commit is merged, it is reviewed by multiple reviewers. This practice helps catch potential security issues early in the development process. The exact number of reviewers required before merging is not specified in the documentation.
Rook empowers contributors to approve and merge code changes autonomously. The maintainer team does not have sufficient resources to fully review and approve all proposed code changes, so trusted members of the community are given these abilities. The goal of this process is to increase the code velocity of all storage providers and streamline their day-to-day operations, such as pull request approval and merging

#### Automated Checks

Rook includes automated checks for vulnerabilities. These checks are part of Rook's continuous integration (CI) process and automatically run against every pull request. The results of these tests, along with code reviews and other criteria, determine whether a pull request will be accepted into the Rook repository

#### Integration Tests

Rook's upstream continuous integration (CI) tests will run integration tests against your changes automatically. You do not need to run these tests locally, but you may if you like.


* Communication Channels.
  Rook Communication Channels

#### Internal Communication
- Slack: Join our [slack channel](https://slack.rook.io)
- GitHub: Start a [discussion](https://github.com/rook/rook/discussions) or open an [issue](https://github.com/rook/rook/issues)
- Security topics: [cncf-rook-security@lists.cncf.io](#reporting-security-vulnerabilities)

#### Inbound Communication

Users or prospective users communicate with the Rook team through GitHub issues and pull requests. GitHub is a platform that hosts the Rook project's codebase and provides features for tracking changes, managing versions, and collaborating on code. Users can report issues, propose changes, or contribute to the project by submitting pull requests.
- GitHub: Start a [discussion](https://github.com/rook/rook/discussions) or open an [issue](https://github.com/rook/rook/issues)

#### Outbound Communication
- Twitter: [@rook_io](https://twitter.com/rook_io)
Mailing lists
* [cncf-rook-security@lists.cncf.io](mailto:cncf-rook-security@lists.cncf.io): for any security concerns. Received by Product Security Team members, and used by this Team to discuss security issues and fixes.
* [cncf-rook-distributors-announce@lists.cncf.io](mailto:cncf-rook-distributors-announce@lists.cncf.io): for
  early private information on Security patch releases.
  
Community Meeting

A regular community meeting takes place every other [Tuesday at 9:00 AM PT (Pacific Time)](https://zoom.us/j/392602367?pwd=NU1laFZhTWF4MFd6cnRoYzVwbUlSUT09).
Convert to your [local timezone](http://www.thetimezoneconverter.com/?t=9:00&tz=PT%20%28Pacific%20Time%29).

Any changes to the meeting schedule will be added to the [agenda doc](https://docs.google.com/document/d/1exd8_IG6DkdvyA0eiTtL2z5K2Ra-y68VByUUgwP7I9A/edit?usp=sharing) and posted to [Slack #announcements](https://rook-io.slack.com/messages/C76LLCEE7/).

Anyone who wants to discuss the direction of the project, design and implementation reviews, or general questions with the broader community is welcome and encouraged to join.

- Meeting link: <https://zoom.us/j/392602367?pwd=NU1laFZhTWF4MFd6cnRoYzVwbUlSUT09>
- [Current agenda and past meeting notes](https://docs.google.com/document/d/1exd8_IG6DkdvyA0eiTtL2z5K2Ra-y68VByUUgwP7I9A/edit?usp=sharing)
- [Past meeting recordings](https://www.youtube.com/playlist?list=PLP0uDo-ZFnQP6NAgJWAtR9jaRcgqyQKVy)


## Security issue resolution

* Responsible Disclosures Process. A outline of the project's responsible
  disclosures process should suspected security issues, incidents, or
vulnerabilities be discovered both external and internal to the project. The
outline should discuss communication methods/strategies.
  * Vulnerability Response Process. Who is responsible for responding to a
    report. What is the reporting process? How would you respond?
* Incident Response. A description of the defined procedures for triage,
  confirmation, notification of vulnerability or security incident, and
patching/update availability.

## Appendix

* Known Issues Over Time. List or summarize statistics of past vulnerabilities
  with links. If none have been reported, provide data, if any, about your track
record in catching issues in code review or automated testing.
* [CII Best Practices](https://www.coreinfrastructure.org/programs/best-practices-program/).
  Best Practices. A brief discussion of where the project is at
  with respect to CII best practices and what it would need to
  achieve the badge.
* Case Studies. Provide context for reviewers by detailing 2-3 scenarios of
  real-world use cases.
* Related Projects / Vendors. Reflect on times prospective users have asked
  about the differences between your project and projectX. Reviewers will have
the same question.
