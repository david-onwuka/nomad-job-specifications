
# Introduction to HashiCorp Nomad Job Specifications

> A practical exploration of workload orchestration with HashiCorp Nomad, using Docker to deploy, manage, and verify an Nginx web server.

![Nomad](https://img.shields.io/badge/HashiCorp-Nomad-00CA8E?logo=nomad&logoColor=white)
![Docker](https://img.shields.io/badge/Container_Runtime-Docker-2496ED?logo=docker&logoColor=white)
![Nginx](https://img.shields.io/badge/Web_Server-Nginx-009639?logo=nginx&logoColor=white)
![HCL](https://img.shields.io/badge/Configuration-HCL-623CE4)
![Status](https://img.shields.io/badge/Lab-Completed-success)

---

## Overview

This project documents my practical introduction to HashiCorp Nomad job specifications and the process of deploying a containerized application through Nomad.

Rather than stopping at the syntax of a job specification, I worked through the deployment lifecycle of an actual workload. I defined an Nginx web server in HCL, configured its Docker driver, specified its networking and resource requirements, validated the configuration, and submitted the job to a local Nomad development agent.

I also explored how Nomad evaluates jobs, schedules allocations, manages tasks, and reports deployment health. During deployment, I encountered Docker Hub DNS resolution failures that delayed the container image pull. Investigating the allocation events helped me understand how infrastructure-level problems can affect workload scheduling and deployment deadlines.

The Nginx container eventually started successfully and returned an HTTP 200 response. However, Nomad retained a failed deployment status because the deployment exceeded its progress deadline. I have documented both outcomes because they demonstrate an important distinction between an application running successfully and an orchestrator reporting a healthy deployment.

## Project Objectives

The main objectives of this exercise were to:

- Understand the structure and purpose of Nomad job specifications.
- Write a workload definition using HashiCorp Configuration Language (HCL).
- Understand the relationship between jobs, task groups, and tasks.
- Configure the Docker driver to run a containerized application.
- Define networking and resource requirements.
- Validate and deploy a job through the Nomad CLI.
- Inspect job evaluations, deployments, allocations, and task events.
- Understand how Nomad handles task failures and restarts.
- Troubleshoot container image-pull problems.
- Verify application availability independently of deployment status.

## Technology Stack

| Technology | Role in the Project |
|---|---|
| HashiCorp Nomad | Workload scheduling and orchestration |
| Docker | Container runtime |
| Nginx | Web server deployed as the workload |
| HCL | Language used to define the Nomad job |
| Ubuntu 24.04 on WSL2 | Local development environment |
| Docker Desktop | Docker engine and container management |
| Git and GitHub | Version control and project publication |

## Repository Structure

```text
nomad-job-specifications/
├── nginx.nomad
├── README.md
└── evidence/
    ├── job-validation.txt
    ├── job-status.txt
    ├── allocation-status.txt
    └── http-verification.txt
```

## Understanding the Nomad Job Specification

Nomad uses declarative job specifications to describe the workloads it should run and the resources required to run them.

For this exercise, I created `nginx.nomad` with the following configuration:

### nginx.nomad

```hcl
job "nginx-web" {
  datacenters = ["dc1"]
  type        = "service"

  group "web" {
    count = 1

    network {
      port "http" {
        to = 80
      }
    }

    task "nginx" {
      driver = "docker"

      config {
        image = "nginx:alpine"
        ports = ["http"]
      }

      resources {
        cpu    = 100
        memory = 128
      }
    }
  }
}
```

### Breaking Down the Configuration

**1. Job Definition**

```hcl
job "nginx-web" {
  datacenters = ["dc1"]
  type        = "service"
}
```

The job block defines the workload submitted to Nomad.

I named the job `nginx-web` and specified `dc1` as its target datacenter. The `service` job type indicates that this workload is intended to run continuously rather than execute as a one-time batch operation.

**2. Task Group**

```hcl
group "web" {
  count = 1
}
```

A task group defines a collection of tasks that Nomad schedules together.

The `count` parameter specifies the desired number of instances of the task group. In this exercise, I requested one instance of the Nginx workload.

**3. Network Configuration**

```hcl
network {
  port "http" {
    to = 80
  }
}
```

This configuration defines a network port labelled `http` and maps it to port `80` inside the container.

I did not specify a fixed host port. This allows Nomad to allocate an available host port dynamically, reducing the possibility of port conflicts.

During deployment, Nomad allocated host port `31127`.

**4. Task and Docker Driver**

```hcl
task "nginx" {
  driver = "docker"

  config {
    image = "nginx:alpine"
    ports = ["http"]
  }
}
```

The task block describes the actual workload.

I selected the Docker driver and specified `nginx:alpine` as the container image. The `ports` configuration references the previously defined `http` network label.

This allows Nomad to manage the Nginx container through Docker.

**5. Resource Requirements**

```hcl
resources {
  cpu    = 100
  memory = 128
}
```

The resource block specifies the resources requested by the task.

For this workload, I allocated:

- CPU: 100 MHz.
- Memory: 128 MiB.

Defining resource requirements allows Nomad to consider the workload's resource needs when scheduling it on an available client node.

## Implementation and Deployment

I performed the exercise in Ubuntu running through WSL2, with Docker Desktop providing the container runtime.

### Step 1: Start the Nomad Development Agent

I started Nomad in development mode:

```bash
nomad agent -dev
```

This started a local Nomad environment with the server and client functionality required for the exercise.

The development agent exposed its HTTP API at:

```text
http://127.0.0.1:4646
```

I kept the agent running in a separate terminal while executing the remaining commands.

### Step 2: Verify Node Availability

Before submitting the job, I checked whether the Nomad client node was available:

```bash
nomad node status
```

The node appeared with a `ready` status.

This confirmed that the local Nomad client was available to receive and execute workloads.

### Step 3: Validate the Job Specification

I validated the HCL configuration:

```bash
nomad job validate nginx.nomad
```

Nomad returned:

```text
Job validation successful
```

This confirmed that the job specification passed validation.

Validation checks the job configuration, but it does not by itself guarantee that every external dependency, such as Docker image availability, will work during deployment.

### Step 4: Submit the Job

I submitted the configuration to Nomad:

```bash
nomad job run nginx.nomad
```

Nomad created the job and initiated an evaluation to determine how the workload should be scheduled.

The resulting deployment and allocation were assigned the following identifiers:

| Resource | Identifier |
|---|---|
| Job | `nginx-web` |
| Evaluation | `6411b585` |
| Deployment | `9b26de2f` |
| Allocation | `2db6b200-36e9-a1bf-455a-f09b0b3c9b2b` |

### Step 5: Inspect Job Status

I used the following command to inspect the job:

```bash
nomad job status nginx-web
```

The job eventually reported:

```text
Status = running
```

The task group summary showed one running instance:

```text
Task Group  Queued  Starting  Running  Failed
web         0       0         1        0
```

This indicated that the desired workload instance was running.

### Step 6: Inspect the Allocation

I inspected the individual allocation:

```bash
nomad alloc status 2db6b200
```

The allocation reported:

```text
Client Status      = running
Desired Status     = run
Task "nginx"       = running
```

The allocation also showed the dynamically assigned network address:

```text
127.0.0.1:31127 -> 80
```

This confirmed that the Nginx task was running and that Nomad had mapped host port `31127` to container port `80`.

## Deployment Verification

A running allocation is useful evidence, but I also wanted to verify that the application itself was responding to requests.

I used `curl` to send an HTTP HEAD request to the dynamically allocated host port:

```bash
curl -I http://127.0.0.1:31127
```

The server returned:

```http
HTTP/1.1 200 OK
Server: nginx/1.31.8
Content-Type: text/html
```

The HTTP 200 response confirmed that the Nginx web server was accessible and responding successfully.

### Observed Deployment Results

| Check | Result |
|---|---|
| Job specification validation | Successful |
| Nomad client node | Ready |
| Job status | Running |
| Allocation status | Running |
| Nginx task | Running |
| Docker container | Running |
| Dynamic host port | 31127 |
| Container port | 80 |
| HTTP response | 200 OK |
| Deployment health status | Failed |

## Troubleshooting: Docker Hub DNS Resolution

One of the most useful parts of this exercise was investigating a deployment problem rather than relying only on the initial job submission output.

During deployment, Nomad attempted to retrieve the `nginx:alpine` image from Docker Hub. The Docker driver encountered DNS resolution failures while trying to contact Docker Hub's authentication service.

The allocation events showed:

```text
lookup auth.docker.io: no such host
```

The failure occurred twice, and Nomad restarted the task as part of its retry behaviour.

I also encountered a Docker credential-helper issue when running `docker pull` directly. The Docker CLI configuration contained a `credsStore` entry referencing `desktop.exe`. I backed up the configuration and removed that credential-store setting for the local CLI configuration.

This credential-helper issue and the DNS resolution failure were separate problems. Removing the credential-store setting did not itself resolve DNS.

Eventually, Docker successfully downloaded all eight image layers, and the Nginx task started.

### Why Nomad Marked the Deployment as Failed

Although the container eventually started, Nomad had already marked the allocation unhealthy.

The allocation events reported:

```text
Task not running by healthy_deadline of 5m0s
```

The deployment subsequently reported:

```text
Status      = failed
Description = Failed due to progress deadline
```

The reason was that the workload did not start within the expected deployment progress period.

### Final State After Recovery

The final inspection showed:

```text
Job Status       = running
Allocation Status = running
Task Status      = running
```

The HTTP connectivity test also returned:

```text
HTTP/1.1 200 OK
```

This means the workload recovered and became accessible, even though the deployment remained recorded as failed.

### What I Learned from the Incident

This experience helped me distinguish three related but different concepts in Nomad:

- **Job status:** Describes the overall state of the submitted job.
- **Allocation status:** Describes the state of a particular scheduled instance of the workload.
- **Deployment health:** Describes whether the deployment has progressed according to Nomad's deployment requirements.

A workload can eventually become operational after a deployment has already been marked failed.

This is why checking only `nomad job status` or relying on a successful HTTP response is not enough when investigating deployment health. Allocation events and deployment information provide important additional context.

## Key Takeaways

By completing this exercise, I developed a better understanding of the following:

1. **Declarative workload configuration:** How to describe an application and its requirements using HCL instead of manually managing its execution.
2. **Nomad architecture:** How jobs, task groups, tasks, evaluations, deployments, and allocations relate to one another.
3. **Container orchestration:** How Nomad uses a task driver to execute containerized workloads.
4. **Resource management:** How CPU and memory requirements are specified for a workload.
5. **Networking:** How Nomad dynamically allocates host ports and maps them to container ports.
6. **Deployment lifecycle:** How a submitted job progresses through evaluation, scheduling, allocation, and execution.
7. **Troubleshooting:** How to investigate task restarts, Docker image-pull failures, and DNS resolution problems.
8. **Operational verification:** Why application-level testing and orchestrator-level deployment health should be examined separately.

## Limitations and Possible Improvements

This exercise was performed using a local Nomad development agent. It demonstrates the fundamentals of Nomad but does not represent a production deployment.

The current configuration uses one workload instance and does not include several features that would be important in a production environment.

Possible improvements include:

- Deploying across multiple Nomad client nodes.
- Adding explicit HTTP service health checks.
- Configuring service discovery.
- Exploring rolling updates and deployment strategies.
- Introducing persistent storage where required.
- Integrating secrets management.
- Adding monitoring, logging, and alerting.
- Testing workload recovery and rescheduling behaviour.

## Conclusion

This exercise gave me practical experience with HashiCorp Nomad beyond simply reading about its job specification syntax.

I wrote and validated an HCL job specification, deployed an Nginx container through the Docker driver, inspected the resulting allocation, and verified application connectivity.

I also encountered and investigated a real deployment issue involving Docker Hub DNS resolution and Nomad's deployment progress deadline.

The most important lesson was that application availability and deployment health are related but distinct operational concerns. Although the Nginx workload eventually recovered and returned HTTP 200 OK, Nomad retained the failed deployment status because the workload exceeded its progress deadline.

Overall, the exercise strengthened my understanding of workload orchestration, container scheduling, resource configuration, networking, deployment monitoring, and troubleshooting with HashiCorp Nomad.

---

## Project Information

| Field | Details |
|---|---|
| Project | Introduction to HashiCorp Nomad Job Specifications |
| Orchestrator | HashiCorp Nomad |
| Container Runtime | Docker |
| Workload | Nginx |
| Configuration Language | HCL |
| Environment | Ubuntu on WSL2 |
| Deployment Mode | Nomad Development Agent |
| Verification | HTTP 200 OK |

**Author:** David Onwuka

**Repository:** [nomad-job-specifications](https://github.com/david-onwuka/nomad-job-specifications)
