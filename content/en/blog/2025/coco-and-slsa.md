---
date: 2025-02-17
title: Confidential Containers(CoCo) and Supply-chain Levels for Software Artifacts (SLSA)
linkTitle: Confidential Containers(CoCo) and Supply-chain Levels for Software Artifacts (SLSA)
description: >
   How CoCo project has started generating and verifying the provenance of its release artifacts according to the SLSA standard?
author: Magnus Kulke ([@mkulke](https://github.com/mkulke)) & Pradipta Banerjee ([@bpradipt](https://github.com/bpradipt))
---

A few of us in the [CNCF Confidential Containers (CoCo)](https://confidentialcontainers.org/) community met some time ago to [discuss provenance management](https://github.com/confidential-containers/confidential-containers/issues/219) in the project.

One of our focus items was to introduce support for generating SLSA provenance and SBOM for CoCo artifacts. We are happy to say that the CoCo project has started generating and verifying the provenance of its release artifacts according to the SLSA standard.

This post is a summary of the progress made to date and the work that is remaining.

## About SLSA

SLSA is an incrementally adoptable set of guidelines for supply chain integrity. Integrity means protection against tampering or unauthorized modification at any stage of the software lifecycle.
Its specifications are helpful for both software producers and consumers: producers can follow the SLSA guidelines to make their software supply chain more secure, and consumers can use SLSA to decide whether to trust a software package. 

The following diagram depicts several known points of attacks.

{{< figure src="/img/supply-chain-threats.svg" alt="Known points of attacks in software supply chain" >}}

**Src: https://slsa.dev**

Note that SLSA does not currently address all of the threats presented here. For example, SLSA v1.0 does not address source threats.
SLSA **addresses build threats**. SLSA also mitigates dependency threats when you verify your dependencies’ SLSA provenance.

### SLSA Levels

SLSA structures itself into a series of levels that provide progressively stronger security guarantees for the supply chain.

These levels are divided into tracks, each focusing on a specific aspect of supply chain security. The tracks allow for the evolution of the SLSA specifications, enabling the introduction of new tracks without affecting the validity of existing levels.

### SLSA Requirements for Build Levels

The build requirements are listed below.

| **Track/Level** | **Requirements**                                        | **Focus**                  |
|-----------------|---------------------------------------------------------|----------------------------|
| Build L0        | (none)                                                  | (n/a)                      |
| Build L1        | Provenance showing how the package was built            | Mistakes, documentation    |
| Build L2        | Signed provenance, generated by a hosted build platform | Tampering after the build  |
| Build L3        | Hardened build platform                                 | Tampering during the build |

**Src: https://slsa.dev/spec/v1.0/levels**

For additional details on SLSA we recommend you to go through the official website: [https://slsa.dev](https://slsa.dev)

## What does SLSA do for CoCo?

Confidential containers get deployed into a Trusted Execution Environment (TEE) driven by a small set of critical infrastructure components like `kata-agent` and `attestation-agent` running on top of a custom-tailored Linux system. The sum of the software in such a TEE we consider to be a Trusted Computing Base (TCB).

We have to trust the TCB not to perform any nefarious or undesired actions that would compromise the confidentiality of the data entrusted to the TEE. This has two immediate implications:

- **We want to keep the software footprint in TCB as small as possible**

*Less* is not just *more* in this context. It's imperative because we cannot reasonably establish trust in a large body of software. Each addition to the software surface opens a vector for potential attacks on confidentiality, and we need to scrutinize it for such problems.

- **We have to be conscious about how we derive our trust**

To trust a TCB, we need to know how it has been assembled, specifically from which components and source code and in which environments it has been built. For an opaque component of questionable and untraceable provenance, it's hard to make a claim about its trustworthiness. A maximalist argument could be made for not providing any binaries but only auditable source code and build instructions.

This is a completely valid strategy for a particular class of consumers; however, for many users, it's not a practical option to test or adopt CoCo for their use cases. In some deployment options, the CoCo project trusts specific upstream dependencies (like the Fedora Project) to provide legitimate artifacts. It also reuses artifacts built in specialized repositories and shared across the project. For the latter scenario, we employed SLSA attestation to establish trust in the components of our TCB.

## CoCo Build Level

**Currently the CoCo project is at Build L2.**

The CoCo project involves different sub-projects/components. It uses the [attest-build-provenance](https://github.com/actions/attest-build-provenance) GitHub action to generate [provenance](https://slsa.dev/spec/v1.0/provenance). It generates signed SLSA provenance in the [in-toto](https://github.com/in-toto/attestation/tree/main/spec/v1) format. You can verify the provenance using the [attestation command in the GitHub CLI](https://cli.github.com/manual/gh_attestation_verify).

It's essential to understand **Attestation** and **Provenance** in the context of SLSA.
**Attestation** refers to an authenticated statement (metadata) about a software artefact or collection of software artifacts.
**Provenance** is the **Attestation** (metadata) describing how the outputs were produced, including identifying the platform and external parameters.

Following is the current state of provenance generation of the CoCo projects:

| **Component**     | **Provenance Generation**                                                                                                                           | **Remarks**                                 |
|-------------------|-----------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------|
| kata-containers   | [GHA workflow](https://github.com/kata-containers/kata-containers/blob/main/.github/workflows/build-kata-static-tarball-amd64.yaml#L124)                            |                                             |
| guest-components  | [GHA workflow](https://github.com/confidential-containers/guest-components/blob/main/.github/workflows/publish-artifacts.yml#L77)                                   |                                             |
| cloud-api-adaptor | [GHA workflow](https://github.com/confidential-containers/cloud-api-adaptor/blob/7e98574207fd6e06ffdd7586c0da90f352d05b04/.github/workflows/podvm_mkosi.yaml#L189)  | Provenance is for the generated qcow2 image |
| enclave-cc        | TBD                                                                                                                                                 |                                             |
| coco operator     | TBD                                                                                                                                                 |                                             |
| trustee           | TBD                                                                                                                                                 |                                             |
| trustee-operator  | TBD                                                                                                                                                 |                                             |

### Artifacts in OCI registries

Formerly exclusively a place to host layers for container images, OCI registries today can serve a multitude of use cases, such as Helm Charts or Attestation data. A registry is a content-addressable store, which means that it is named after a digest of its content.

```sh
$ DIGEST="84ec2a70279219a45d327ec1f2f112d019bc9dcdd0e19f1ba7689b646c2de0c2"
$ oras manifest fetch "quay.io/curl/curl@sha256:${DIGEST}" | sha256sum
84ec2a70279219a45d327ec1f2f112d019bc9dcdd0e19f1ba7689b646c2de0c2  -
```

We also use OCI registries to distribute and cache artifacts in the CoCo project.
There is a convention of specifying upstream dependencies in a `versions.yaml` file this:

```sh
oci:
  ...
  kata-containers:
	registry: ghcr.io/kata-containers/cached-artefacts
	reference: 3.13.0
  guest-components:
	registry: ghcr.io/confidential-containers/guest-components
	reference: 3df6c412059f29127715c3fdbac9fa41f56cfce4
```

Note that the `reference` in this case is a tag, sometimes a version, and sometimes a reference to the digest of a given git commit, not the digest of the OCI artefact. What do we express from this specification, and what do we want to verify?
We might want to resolve a tag to a git digest first, so tag 3.13.0 resolves to the digest `2777b13db748f9ba785c7d2be4fcb6ac9c9af265`. Knowing the git digest, we now want to verify that official repository runners built the artefact from the main branch using the source code and build workflows from that above digest.  
This is what the SLSA attestation that we created during the artefact's build can substantiate.
It wouldn't make sense to attest against an artifact using an OCI tag alias (those are not immutable, and one can move it to point to something else); the attestations are tied to an OCI artifact referenced by its digest and conveniently stored alongside this in the same repo. We can find it manually if we search for referrers of our OCI artifact.

```sh
$ GIT_DGST="2777b13db748f9ba785c7d2be4fcb6ac9c9af265"
$ oras resolve "ghcr.io/kata-containers/cached-artefacts/agent:${GIT_DGST}-x86_64"
sha256:c127db93af2fcefddebbe98013e359a7c30b9130317a96aab50093af0dbe8464
$ OCI_DGST=sha256:c127db93af2fcefddebbe98013e359a7c30b9130317a96aab50093af0dbe8464
$ oras discover "ghcr.io/kata-containers/cached-artefacts/agent@OCI_DGST"
...
└── application/vnd.dev.sigstore.bundle.v0.3+json
	└── sha256:f93cc5a59ec0b9d23482831e1ce52bb3298c168da5c2d333a33118280f1f6d5b
```

### Provenance Verification

As mentioned previously, Github provides a command line option to verify the [provenance](https://cli.github.com/manual/gh_attestation_verify).

The following snippet from the cloud-api-adaptor project's [Makefile](https://github.com/confidential-containers/cloud-api-adaptor/blob/main/src/cloud-api-adaptor/podvm/Makefile.inc) shows an example usage:

```sh
define pull_agent_artifact
	$(eval $(call generate_tag,tag,$(KATA_REF),$(ARCH)))
	$(eval OCI_IMAGE := $(KATA_REGISTRY)/agent)
	$(eval OCI_DIGEST := $(shell oras resolve $(OCI_IMAGE):${tag}))
	$(eval OCI_REF := $(OCI_IMAGE)@$(OCI_DIGEST))
	$(if $(filter yes,$(VERIFY_PROVENANCE)),$(ROOT_DIR)hack/verify-provenance.sh \
		-a $(OCI_REF) \
		-r $(KATA_REPO) \
		-d $(KATA_COMMIT))
	oras pull $(OCI_REF)
endef
```

The `verify-provenance.sh` script is a utility helper for the `gh attestation verify` command. It encodes the requirements we made about a valid build above.
The source is available [here](https://github.com/confidential-containers/cloud-api-adaptor/blob/main/src/cloud-api-adaptor/hack/verify-provenance.sh).

## Next Steps

To further improve the security posture of the CoCo project, we aim to integrate provenance generation and verification processes for other sub-projects, such as the CoCo operator and trustee.

We welcome your feedback and contributions—join our [community](https://github.com/confidential-containers/#join-the-community) and be part of the conversation!
