+++
title = "Homelab - IaC for Cloud Infrastructure"
date = "2022-03-14"
aliases = ["projects"]
[ author ]
  name = "Nikita Gulyayev"
+++

## Introduction
Homelab is a repository showcasing my Infrastructure as Code (IaC) practices for managing cloud-based environments. It's a collection of Terraform configurations and Ansible playbooks that automate the deployment and management of various cloud services and applications.

## Repository Structure
- **Ansible**: Contains playbooks and roles for configuring services like Cointracker, Grafana, Jenkins, Prometheus, and Redis.
- **AWS**: Hosts Terraform for AWS infrastructure, including EKS, general AWS infrastructure setups, and vanilla Kubernetes configurations.
- **Helm Charts**: Hosts the Helm chart for deploying Cointracker on K8s.
- **Hetzner**: Contains Terraform configurations for Hetzner cloud setups, including K3s cluster.
- **Examples**: A section with example Terraform files for EC2 autoscaling, Nginx setup, and managing remote state infrastructure.

## Key Highlights
- **Container Orchestration**: Helm charts and K8s configurations for container-based applications.
- **Automated Setup**: Simplifies complex setups like EKS clusters and vanilla K8s on AWS.
- **Monitoring and CI/CD**: Integrates tools like Grafana, Prometheus, and Jenkins for monitoring and continuous integration.

## Usage
The repository is a resource for anyone interested in IaC, providing real-world examples of cloud infrastructure management.

## Repository
To explore the code visit the [Homelab GitHub Repository](https://github.com/nickyfoster/homelab).
