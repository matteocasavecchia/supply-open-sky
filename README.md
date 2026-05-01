# Supply Open Sky

*Autonomous drone delivery network for communities without access to basic services*

---

## Executive Summary

Supply Open Sky is an integrated autonomous drone delivery system designed
to bring potable water, medicines, mail, and essential goods to communities that
lack reliable access to basic services. It is built for geographically isolated
regions where conventional logistics are ineffective or impossible — areas
without dependable road infrastructure or established distribution networks.

The architecture is grounded in three principles: full operational autonomy with
no human intervention required along the network, resilience to communication
outages, and modular scalability that allows the network to grow without
redesigning existing infrastructure.

The network is organized as a tree topology branching out from a central Hub.
Drones depart loaded with payload, deliver along scheduled routes or to
on-demand GPS coordinates, and return to be reloaded. Each Node is
solar-powered, fully autonomous, and serves a dual purpose: it is relay
infrastructure for drone operations, and at the same time a permanent service
point for the surrounding community — providing public water access and an
emergency communication link via a LoRa Mesh backbone.

Supply Open Sky is not just a delivery system. It is logistical infrastructure designed to
exist where no other infrastructure does.

## What this repository contains

This repository hosts the public architectural documentation of the Supply Open
Sky project. The implementation code — drone firmware, scheduling engine, hub
software, communication stack, simulator — is maintained in a private
repository during the pilot deployment phase and is not published here.

- **[BLUEPRINT.md](./BLUEPRINT.md)** — full architectural specification of the
  system: operational concept, network topology, mission types, control
  architecture, communication stack, and design considerations.
- **[NOMENCLATURE.md](./NOMENCLATURE.md)** — reference for the technical
  terminology used across the project documentation and source code.

## Current Status

Hardware deployment is complete; pilot operations are in preparation. A
first end-to-end network is installed on physical hardware, with the full
software stack (mission scheduler, hub services, communication backbone,
drone and node firmware) running on dedicated single-board computers.
End-to-end mission execution has been validated in test against both
simulated and real flight scenarios.

The current phase focuses on operational hardening, telemetry
instrumentation, and validation of the scheduling logic on real flight
data.

*This section is updated periodically as the project advances.*

## How to follow this project

- **LinkedIn** — [Matteo Casavecchia](https://www.linkedin.com/in/casavecchia/)
- **Watch this repository** for documentation updates and future releases.

## License

The documentation in this repository is released under the
[Creative Commons Attribution 4.0 International License (CC BY 4.0)](./LICENSE).
You are free to share and adapt the material with appropriate attribution.
