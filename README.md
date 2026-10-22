# PNK Azure DevOps Kubernetes Agent

> A lightweight, secure, and event-driven Azure DevOps agent runner designed for Kubernetes, orchestrated via KEDA, and operating unprivileged containers with an isolated Docker-in-Docker (DinD) sidecar architecture.

---

## 🚀 Overview

**PNK** is a custom solution for running Azure DevOps (AzDO) self-hosted agents inside Kubernetes clusters. It addresses the common challenge of running CI/CD pipelines that require Docker capabilities while strictly adhering to security best practices (running container workloads without root/privileged access).

By decoupling the AzDO agent from the Docker daemon using a dedicated sidecar (`DinD`), PNK ensures secure execution, dynamic resource management, and zero orphan agents left behind in Azure.

---

## 🏛️ Architecture & How It Works

PNK utilizes a modular sidecar pattern managed through Kubernetes ConfigMaps and Helm templates:

1. **Unprivileged Agent Container:** The main agent runs without privileged security contexts.
2. **Docker-in-Docker (DinD) Sidecar:** Runs alongside the agent to build and run container images requested by pipelines via a shared Unix socket (`/var/run/docker.sock`).
3. **KEDA Integration:** Automatically scales agents based on Azure DevOps queue length (supporting both `Job` and `Object` deployment models).
4. **Lifecycle Control & Step Monitoring:** Custom wrapper scripts (`start_agent.sh`, `monitor_job.sh`, `azdo.sh`) track individual pipeline steps, handle graceful shutdowns, manage container states (`READY`, `BUSY`, `DRAINING`), and guarantee proper cleanup.

---

## 🛠️ Key Features

* **Zero Privileges:** Main agent containers run securely without elevated permissions.
* **DinD Integration:** Securely shares `/var/run/docker.sock` with a dedicated Docker daemon sidecar to handle user image builds.
* **Advanced Step & State Monitoring:** Real-time tracking of pipeline status to handle graceful pod termination (`SIGTERM`) without cutting off active jobs mid-execution.
* **Automatic Registration & Cleanup:** Dynamically registers with Azure DevOps on startup and automatically unregisters/cleans up upon completion or failure to prevent orphan agents.
* **Flexible Scaling Models:** Supports KEDA `ScaledJob` (ephemeral single-run agents) or deployment objects with schedule/cron triggers.

---

## 📂 Script Structure

The core logic resides in shell scripts mounted via ConfigMaps:

* `azdo.sh`: Handles REST API interactions with Azure DevOps, agent configuration, and health checks.
* `common.sh`: Shared environment variables, logging utilities, path definitions, and permission fixers.
* `start_docker.sh`: Manages the lifecycle of the Docker daemon sidecar and cleans up runtime signals.
* `start_agent.sh`: Main entrypoint; synchronizes dependencies, registers the agent, and coordinates the execution loop.
* `monitor_job.sh`: Periodically inspects worker processes to report granular states (`BUSY` during pipeline steps, `DRAINING` when termination is requested).

---

## ⚙️ Configuration

Configured via Helm values (`values.yaml`). Example snippet:

```yaml
pnk:
  agent_name: "pnk-azdo-agent"

keda:
  scaledtype: "job" # Options: "job" or "object"
  
docker:
  storage_driver: "overlay2"
