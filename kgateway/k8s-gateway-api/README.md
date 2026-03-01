# Kubernetes Gateway API

## Embracing the Future of Kubernetes Traffic Management

The Kubernetes Gateway API represents a significant evolution in how we manage ingress traffic in cloud-native environments. As a CNCF project that
graduated to beta status, Gateway API provides a more expressive, extensible and role-oriented approach to configuring network traffic compared to the traditional Ingress resource.

In this comprehensive guide, we'll walk through setting up Gateway API on Kubernetes using entirely open-source tools, demonstrating why this
is becoming the go-to solution for modern cloud-native architectures.

## Why Gateway API?

Before diving into the setup, let's understand what makes Gateway API special:

**Role-Oriented Design:** Gateway API separates concerns between cluster operators (who manage infrastructure) and application developers (who manage routes), enabling better collaboration and security boundaries.

**Enhanced Expressiveness:** Support for header-based routing, traffic weighting, request mirroring, and more advanced traffic management patterns out of the box.

**CNCF Standarization:** As a CNCF project, Gateway API ensures vendor neutrality and community-driven evolution, avoiding vendor lock-in.

**Protocol Support:** Native support for HTTP, HTTPS, TCP, UDP, and gRPC, making in truly multi-protocol.

## Prerequisites

Before we begin, ensure you have the following:

- A running Kubernetes cluster (v1.23 or later). You can use minikube, kind, or any managed Kubernetes service.
- kubectl configured to communicate with your cluster.
- Basic understanding of Kubernetes concepts like Pods, Services, and Deployments.
- Cluster admin permissions to install CRDs.

## Architecture Overview

Gateway API introduces several key resources that work together to manage traffic:

**GatewayClass:** Defines a class of Gateways that share common configuration, typically managed by cluster adminstrators. Think of it as the template
that defines which controller will handle the Gateway.

**Gateway:** Represents the actual load balancer or proxy instance. It listens on specific ports and protocols, providing the entry point for traffic into the cluster.

**HTTPRoute:** Defines HTTP-specific routing rules, connecting Gateways to backend Services. This is where application developers define their routing logic.

**ReferenceGrant:** Enables cross-namespace references, allowing routes in one namespace to reference services in another while maintaining security boundaries.

## Step-by-Step Setup Guide

### Step 1: Install Gateway API CRDs

Gateway API is implemented as Kubernetes Custom Resource Definitions. Let's install them first.

The Gateway API CRDs are version-controlled and maintained by the CNCF community. We'll install the stable channel which includes GA resources:

```bash
# Install Gateway API CRDs (v1.2.1 - latest stable as of writing)
kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.2.1/standard-install.yaml
```

This command installs the core CRDs including GatewayClass, Gateway, HTTPRoute, and ReferenceGrant.

Verify the installation:

```bash
kubectl get crd | grep gateway
```

### Step 2: Choose and Install a Gateway Controller

Gateway API is an interface specification. You need a controller implementation to actually handle the traffic. Several CNCF and open-source projects support Gateway API. For this tutorial, we'll use **Envoy Gateway**, a CNCF project that provides a batteries-included Gateway API implementation.

**Installing Envoy Gateway**

```bash
# Install Envoy Gateway using Helm
helm repo add envoy-gateway https://gateway.envoyproxy.io
helm repo update
# Create namespace for Envoy Gateway
kubectl create namespace envoy-gateway-system
# Install Envoy Gateway
helm install eg envoy-gateway/gateway-helm \
  --namespace envoy-gateway-system \
  --create-namespace
```