# Hyperswitch Project Overview

## Project Description
**Hyperswitch** is a composable, open-source payments infrastructure built in Rust. It's a modular payment processing system that enables businesses to build their own payment stack without vendor lock-in.

### Tagline
"Linux for Payments" - A well-architected reference for teams who want to own their payments stack.

## Key Features
1. **Modular Architecture**: Pick only the components you need
2. **Multi-Connector Support**: Integrate with multiple payment processors
3. **Global Payment Methods**: Cards, wallets, BNPL, UPI, Pay by Bank
4. **Intelligent Routing**: Route transactions based on predicted success rates
5. **Revenue Recovery**: Combat passive churn with retry strategies
6. **Cost Observability**: Audit and monitor payment costs
7. **Vault**: PCI-compliant payment method storage
8. **Reconciliation**: Automated 2-way and 3-way reconciliation
9. **Alternate Payment Methods**: PayPal, Apple Pay, Google Pay, Samsung Pay, Klarna

## Project Statistics
- **Primary Language**: Rust (Edition 2021, Rust 1.85.0+)
- **License**: Apache 2.0
- **Repository**: https://github.com/juspay/hyperswitch
- **Maintained By**: Juspay (powering 400+ enterprises)
- **Core Team**: 150+ engineers

## Repositories
1. **App Server** (this repo) - Core payments engine, payment unification, smart routing
2. **Web Client (SDK)** - https://github.com/juspay/hyperswitch-web - Frontend widgets and SDKs
3. **Control Center** - https://github.com/juspay/hyperswitch-control-center - Dashboard for analytics and operations

## Key Design Principles
- **Payment Diversity**: Enable choice across payment methods, processors, and flows
- **Open Source First**: Transparency drives trust and reusable software
- **Community-Driven**: Roadmap shaped by contributors and real-world use cases
- **Systems Engineering**: High standards for reliability, security, performance
- **Value Creation**: For developers, customers, and partners

## Deployment Options
- **Local Docker**: `scripts/setup.sh` - One-click setup
- **Hosted Sandbox**: https://app.hyperswitch.io (no setup required)
- **Cloud Deployment**: AWS, GCP, Azure via Helm or CDK

## Project Phases
- **Specification** - Requirements analysis
- **Pseudocode** - Algorithm design
- **Architecture** - System design
- **Refinement** - TDD implementation
- **Completion** - Integration
