# Self-assessment for Karmada

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
| Software | [A link to Karmada’s repository.](https://github.com/karmada-io/karmada)  |
| Security Provider | No. The primary function of the project is to run Kubernetes clusters across clouds. Security is not the primary objective. |
| Languages | Go, Shell and others |
| SBOM | Software bill of materials.  Link to the libraries, packages, versions used by the project, may also include direct dependencies. |
| | |

### Security links

Provide the list of links to existing security documentation for the project. You may
use the table below as an example:
| Doc | url |
| -- | -- |
| Security file | TBD - https://my.security.file |
| Default and optional configs | TBD - https://example.org/config |

## Overview
Karmada (Kubernetes Armada) 
Karmada is a Kubernetes clusters management system that enables you to run multiple Kubernetes clusters across clouds. It uses Kubernetes-native APIs and provides advanced scheduling capabilitie supports multi-cloud Kubernetes.

Karmada aims to provide turnkey automation for multi-cluster application management in multi-cloud and hybrid cloud scenarios, with key features such as centralized multi-cloud management, high availability, failure recovery, and traffic scheduling.

### Background

Provide information for reviewers who may not be familiar with your project's
domain or problem area.
When a development team utilizes Kubernetes for cluster deployment, they often encounter limitations on the number of pods that can be effectively deployed and managed. These limitations can be imposed by the cloud platform's constraints or available resources. Consequently, the development team is compelled to set up and manage multiple Kubernetes clusters. These clusters may serve specific purposes, such as catering to different regions, accommodating various application versions, or handling load balancing requirements. There exist numerous motivations for scaling instances both horizontally and vertically. However, this scaling endeavor brings with it a set of challenges and complexities that must be addressed.

Karmada addresses the challenge of managing clusters across diverse platforms. It offers centralized management of Kubernetes clusters deployed across different cloud platforms and regions, streamlining the management process.

### Actors
These are the individual parts of your system that interact to provide the 
desired functionality.  Actors only need to be separate, if they are isolated
in some way.  For example, if a service has a database and a front-end API, but
if a vulnerability in either one would compromise the other, then the distinction
between the database and front-end is not relevant.

The means by which actors are isolated should also be described, as this is often
what prevents an attacker from moving laterally after a compromise.

The following are the Actors found in the Karmada project:

1. Karmada Admin
2. Karmada CLI
3. Karmada API Server
4. Karmada Controller Manager
5. Karmada Agent
6. Karmada Scheduler
7. Member Cluster

#### Karmada Control Plane
The Karmada Control Plane consists of:
* Karmada API Server
* Karmada Controller Manager
* Karmada Scheduler
* etcd

#### Karmada API Server
The aggregate API server is an extended API server implemented using Kubernetes API Aggregation Layer technology. It offers Cluster API and related sub-resources, such as cluster/status and cluster/proxy, it implements advanced capabilities like Aggregated Kubernetes API which can be used to access member clusters through karmada-apiserver.

#### Karmada Controller Manager
The karmada-controller-manager runs various controller processes.
The controllers watch Karmada objects and then talk to the underlying clusters' API servers to create regular Kubernetes resources.
The controllers are listed at Karmada Controllers.

#### Karmada Agent
Karmada has two Cluster Registration Mode such as Push and Pull, karmada-agent shall be deployed on each Pull mode member cluster. It can register a specific cluster to the Karmada control plane and sync manifests from the Karmada control plane to the member cluster. In addition, it also syncs the status of member cluster and manifests to the Karmada control plane.

#### Karmada Scheduler
The karmada-scheduler is responsible for scheduling k8s native API resource objects (including CRD resources) to member clusters.
The scheduler determines which clusters are valid placements for each resource in the scheduling queue according to constraints and available resources. The scheduler then ranks each valid cluster and binds the resource to the most suitable cluster.

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

#### Push Mode
In Push mode, the Karmada control plane directly accesses the kube-apiserver of each member cluster to get status and deploy manifests. The karmada-agent is not required.

##### Actors
* User - Executes Karmada CLI to join/unregister clusters.
* Karmada CLI - CLI tool that calls Karmada control plane to join/unregister clusters.
* Karmada control plane - Creates/updates/deletes Cluster object representing the registered cluster.
* Member cluster - Its kube-apiserver is directly accessed by Karmada control plane (Karmada Controller Manager).

##### Workflow
1. User executes kubectl-karmada join to register a cluster.
2. The CLI calls Karmada control plane to create Cluster object.
3. Within Karmada control plane directly interacts with cluster's kube-apiserver for status and manifest deployment.
4. User executes kubectl-karmada unjoin to unregister the cluster.
5. The CLI calls Karmada control plane API to delete Cluster object.

In summary, Karmada control plane will access member cluster's kube-apiserver directly to get cluster status and deploy manifests.

#### Pull Mode
In Pull mode, the Karmada control plane (Karmada Controller Manager) does not directly access the member cluster. Instead, access is delegated to the Karmada Agent component deployed on each cluster.

##### Actors
* User - Executes Karmada CLI to join/unregister clusters.
* Karmada CLI - CLI tool that calls Karmada control plane API to register clusters. Also deploys karmada-agent.
* Karmada Controller Manager - Creates/deletes Cluster object representing registered cluster. Does not directly access cluster.
* Karmada Agent - Agent deployed on each cluster that handles communication with Karmada.
* Member Cluster - Its API server is accessed by the karmada-agent, not directly by Karmada.

##### Workflow
1. User executes karmadactl register to deploy karmada-agent and register a cluster.
2. CLI calls Karmada API to create Cluster object and deploy karmada-agent.
3. Agent creates ServiceAccount and RoleBinding for API access.
4. Agent watches karmada-es namespace and reports status to Karmada.
5. User executes karmadactl unregister to delete karmada-agent.
6. User manually deletes Cluster object from Karmada.

In summary, Karmada control plane will access Karmada Agent component deployed on each cluster to get cluster status and deploy manifests.

### Goals
The intended goals of the projects including the security guarantees the project
 is meant to provide (e.g., Flibble only allows parties with an authorization
key to change data it stores).

**General**

* Scalability: Support Horizontoal and vertical scaling of applications smoothly across clusters.
* Centralized Management: Provide a unified control plane to manage multiple Kubernetes clusters deployed across different platforms like on-prem, public clouds, edge, etc.
* Open and Neutral: Kubernetes native API compatible and integrates with mainstream cloud providers.

**Security**

* At deployment time, karmada data should be protected and robust against tampering.
* Authentication and authorization for API access to Karmada control plane components like the API server.
* Validating identity of member clusters joining Karmada control plane.
* Protect control plane from being compromised and it's identity should be checked by member clusters before processing on the request.

### Non-goals
Non-goals that a reasonable reader of the project’s literature could believe may
be in scope (e.g., Flibble does not intend to stop a party with a key from storing
an arbitrarily large amount of data, possibly incurring financial cost or overwhelming
 the servers)

**General**

* TBD

**Security**
* Stop a third-party with an API key from accessing Karmada Control Plane.
* Prevent a malicious actor using Kubernetes vulnerabilites to exploit the Karmada.

## Self-assessment use

This self-assessment is created by the Karmada team to perform an internal analysis of the
project's security.  It is not intended to provide a security audit of Karmada, or
function as an independent assessment or attestation of Karmada's security health.

This document serves to provide Karmada users with an initial understanding of
Karmada's security, where to find existing security documentation, Karmada plans for
security, and general overview of Karmada security practices, both for development of
Karmada as well as security of Karmada.

This document provides the CNCF TAG-Security with an initial understanding of Karmada
to assist in a joint-assessment, necessary for projects under incubation.  Taken
together, this document and the joint-assessment serve as a cornerstone for if and when
Karmada seeks graduation and is preparing for a security audit.

### Threat Modeling with STRIDE

#### Spoofing

##### Threat-01-S - Spoofing of Karmada Admin

The identity of the Karmada Admin with access to Karmada API-Sever can be spoofed
by other users due to stolen credentials or lack of authentication.

##### Threat-01-S Recommended mitigations

* Karmada API-Sever must authenticate Karmada Admin before processing request.
* If the authentication check fails then the request must be discarded and logged as a security event.

##### Threat-02-S - Spoofing of a Karmada API-Sever

 When a worker cluster is created, a user could somehow interfere with it and send 
 instructions to it as Karmada API-Sever, this can happen through Karmada Agent or 
 kube-apiserver (depending push or pull mode). If the user could gain access to the 
 Karmada worker cluster, it can allow the user to gain access to secrets they should 
 not be able to.

##### Threat-02-S Recommended Mitigations

* Karmada worker cluster must authenticate Karmada API-Sever before processing request.
* Karmada Agent must authenticate Karmada API-Sever before processing request.
* If the authentication check fails then the request must be discarded and logged as a security event.

#### Tampering

##### Threat-03-T - Tampering of Karmada

 Karmada components (API Server, Agent, Controller Manager and Scheduler) and configuration 
 files (ETCD) can be tampered during build time, installation or runtime.

##### Threat-03-T Recommended Mitigations

* At build time: Create SBoM of all the Karmada components and verify checksum as a post build action.
* During installation: Verify checksum of the downloaded components.
* During runtime: Alert and log on modification of Karmada components to detect for potential tampering.

##### Threat-04-T - Tampering of Communication to Karmada Worker Cluster

 The communication from Karmada Control Plane to Karmada Worker Cluster can be tampered 
 during transit. This can allow malicious actors to modify the commands sent by the Karmada 
 Control Plane to Karmada Agent or kube-apiserver (depending push or pull mode).

##### Threat-04-T Recommended Mitigations

* Karmada worker cluster must check the integrity of the commands recived from Karmada Control Plane before processing.
* Karmada Agent must check the integrity of the commands recived from Karmada Control Plane before processing.
* If the integrity check fails then the commands must be discarded and logged as a security event.


#### Repudiation

##### Threat-05-R - Repudiation of actions performed by Karmada Admin

Karmada Admin has highly privileged access to cloud and cluster resources. Misuse
of permissions via actions performed by Karmada Admin should be detectable and
logged for auditing purposes.

##### Threat-05-R Recommended Mitigations

* Implement auditing for Karmada control plane to log all actions performed by Karmada Admin.
* Implement centralized audit collection and alerting for any suspicious activity or security events.

#### Information Disclosure

##### Threat-06-I - Exposure of communication between Karmada Control Plane and Karmada Worker Cluster

A malicious actor can snoop the communication between Karmada Control Plane and Karmada Worker cluster.
This allows the malicious actor to get access to any sensitive or secret information passed between them.

##### Threat-06-I Recommended mitigations

* Sensitive information send between Karmada control plane and Karmada worker cluster (Karmada Agent and kube-apiserver) must be encrypted.

#### Denial of Service

##### Threat-07-D - Exhausting Cloud resources

A user within the network of KArmada environment can send incessant requests to Karmada worker cluster making it unavailable to Karmada Contorl Plane.


##### Threat-07-D Recommended Mitigations

* Isolate the network of Karmada environment.
* Isolate the Karmada worker cluster from users.
#### Elevation of Privilege

##### Threat-08-E - Using Karmada worker cluster to get access to Karmada Control Plane

If a Karmada worker cluster is colocated in the same system as a Karmada control plane.
A user can use the credentials of Karmada worker cluster to snapshot the disk 
of the Karmada control plane, create a new EC2 volume, mount it to a node in the workload 
cluster, and exfiltrate the etcd datastore. The same thing can happen if a Karmada worker cluster
is compromised or controlled by a malicious actor.

##### Threat-08-E Recommended mitigations

* Karmada control plane and Karmada worker cluster (Karmada Agent and kube-apiserver) must have different roles and access levels to implement separation of privileges.


##### Threat-09-E - Using create cluster permission to get cluster admin access

Someone with create cluster permissions can elevate their access by locating the
credentials that can give cluster-admin access to the attacker.

##### Threat-09-E Recommended Mitigation

* Limit who can create pods.
* Implement Cluster level Pod Security to prevent host access for someone with
  Create Pod access.

## Security functions and features

* Critical.  A listing critical security components of the project with a brief
description of their importance.  It is recommended these be used for threat modeling.
These are considered critical design elements that make the product itself secure and
are not configurable.  Projects are encouraged to track these as primary impact items
for changes to the project.
* Security Relevant.  A listing of security relevant components of the project with
  brief description.  These are considered important to enhance the overall security of
the project, such as deployment configurations, settings, etc.  These should also be
included in threat modeling.

## Project compliance

* Compliance.  List any security standards or sub-sections the project is
  already documented as meeting (PCI-DSS, COBIT, ISO, GDPR, etc.).

## Secure development practices

* Development Pipeline.  A description of the testing and assessment processes that
  the software undergoes as it is developed and built. Be sure to include specific
information such as if contributors are required to sign commits, if any container
images immutable and signed, how many reviewers before merging, any automated checks for
vulnerabilities, etc.
* Communication Channels. Reference where you document how to reach your team or
  describe in corresponding section.
  * Internal. How do team members communicate with each other?
  * Inbound. How do users or prospective users communicate with the team?
  * Outbound. How do you communicate with your users? (e.g. flibble-announce@
    mailing list)
* Ecosystem. How does your software fit into the cloud native ecosystem?  (e.g.
  Flibber is integrated with both Flocker and Noodles which covers
virtualization for 80% of cloud users. So, our small number of "users" actually
represents very wide usage across the ecosystem since every virtual instance uses
Flibber encryption by default.)

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
