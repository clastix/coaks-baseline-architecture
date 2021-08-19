# Capsule over AKS (CoAKS) Baseline Architecture - DRAFT
This reference implementation introduces the recommended starting (baseline) infrastructure architecture for implementing a multi-tenancy [Azure AKS](https://azure.microsoft.com/services/kubernetes-service) cluster using the [Clastix Capsule Operator](https://github.com/clastix/capsule). 

This implementation and document is meant to guide Azure users through the process of getting this secure baseline infrastructure deployed and understanding the components of it.

This architecture is infrastructure focused, more so than on applications. It concentrates on the AKS cluster itself, including concerns with user identity, cluster governance, policies management, data persistence and network topologies.

The implementation presented here is the minimum recommended baseline for most AKS clusters. This implementation integrates with other Azure services that will provide a network topology that will support multi-regional scenarios, and keep the in-cluster traffic secure as well. This architecture should be considered your starting point for pre-production and production stages.

The material here is relatively dense. We strongly encourage you to dedicate time to walk through these instructions, with a mind to learning. We do NOT provide any "one click" deployment here. However, once you've understood the components involved it is encouraged that you build suitable, auditable GitOps deployment processes around your final infrastructure.


