---
title: "Cluster API the Missing Manual"
date: 2023-02-22T12:20:14-08:00
draft: true
---

Gate 1: InfraObject must be owned by CoreObject. Once this is complete, the infra controller can proceed.



Infrastructure Provider sets a Ready status to true
This becomes a capi.Cluster's Status.InfrastructureReady field.

The bootstrap provider sets a Ready status to true
This becomes a Cluster's Status.ControlPlaneReady field.

A Machine's Ready status means that there is bootstrap config data available for use
