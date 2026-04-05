---
layout: page
title: About
permalink: /about/
---

## About Me

Senior Software Engineer with 14+ years of experience in distributed systems, microservices architecture, and event-driven platforms across banking and public infrastructure. Now operating at the forefront of AI-native engineering — shifting from manual coding to designing, orchestrating, and validating AI-generated solutions using tools like Claude Code and GitHub Copilot.

Deep expertise in highly regulated, large-scale environments including Open Banking, secure API ecosystems, and real-time logistics optimisation. Passionate about evolving backend engineering into a discipline centred on system design, AI orchestration, and architectural rigour rather than low-level implementation.

## Skills

**AI & Tooling**: Claude Code, GitHub Copilot, Google AI Tools, AI-augmented development workflows

**Cloud & Azure**: App Service, Azure Functions, Service Bus, Application Insights, KQL

**Backend**: Java, C#, TypeScript, Spring Boot, Microservices, Event-Driven Architecture, Kafka, REST/GraphQL APIs

**Security & Compliance**: OAuth2, OIDC, mTLS, API Gateway patterns, Defence-in-Depth, Open Banking standards

**DevOps & Practices**: GitHub Actions, CI/CD, Doc-as-Code, Infrastructure as Code

**Data & Storage**: HBase, Redis, MySQL, Protobuf, KQL, CDC, Transactional Outbox Pattern

**Architecture Patterns**: CQRS, Saga, Circuit Breaker, Chain of Responsibility, Finite State Machine, Decorator

**Optimisation**: Vehicle Routing Problem (VRP), dynamic dispatching, real-time decision-making

## Experience

### Software Engineer — WiseTech Global

Melbourne, VIC

Working in an AI-augmented engineering model within the Australian logistics domain, contributing to Transport Management System (TMS) optimisation and cloud-native platform development.

- Adopted an AI-native development approach using Claude Code and GitHub Copilot, acting as a technical control layer responsible for problem decomposition, AI orchestration, and ensuring production-grade quality of generated solutions
- Applied Vehicle Routing Problem (VRP) techniques to improve route efficiency, dynamic dispatching, and real-time decision-making in a logistics TMS
- Designed and maintained Azure-based event-driven services using App Service, Azure Functions, and Service Bus to build scalable, reliable architectures
- Implemented system observability and diagnostics with Azure Application Insights and KQL, enabling proactive monitoring and faster incident resolution
- Built and maintained GitHub Actions CI/CD pipelines to streamline multi-environment deployments
- Championed a Doc-as-Code approach, version-controlling technical documentation alongside source code to improve team alignment and knowledge retention
- Practised Test-Driven Development (TDD) in C# to ensure production-grade reliability, maintainability, and confidence in AI-generated code outputs

### Senior Software Engineer — Previous Role

Led end-to-end architecture and implementation of an Open Banking Platform comprising six core systems: Edge Gateway, API Gateway, Authorization Server, Resource Server, Developer Portal, and Control Center.

- Designed a layered security model (Defence-in-Depth) with robust API security mechanisms including OAuth2/OIDC, mTLS, IP whitelisting, ACLs, and anti-replay controls; introduced a plugin-based architecture with Redis-cached policy tables using the Chain of Responsibility pattern for dynamic gateway extensibility
- Designed a configuration propagation system using CDC and the transactional outbox pattern to decouple modules, ensuring eventual consistency and full audit compliance; modelled third-party app lifecycles with a finite state machine enforcing controlled, auditable state transitions
- Collaborated with 13 national financial institutions to co-author API Security Best Practices for Open Banking, and built a Developer Portal enabling thousands of third-party partners to onboard via app registration, sandbox testing, and certificate generation

### Software Engineer — Previous Role

Designed and delivered scalable backend systems for a public transportation platform handling real-time GPS data from 1,000+ concurrent bus devices.

- Migrated GPS data storage from MySQL to HBase with Protobuf encoding, reducing daily storage by 60% and query latency by over 80% for time-range reports; built a hierarchical reporting engine using post-order tree traversal and the Decorator pattern to modularise 60+ mileage reports
- Refactored high-throughput data ingestion with Netty, replacing a legacy C++ UDP server to achieve a 35% throughput gain for 1,000+ concurrent bus connections — earning internal recognition for technical innovation
- Reverse-engineered a misdocumented third-party GPS device protocol through binary pattern analysis, identifying a parity-check checksum misrepresented as CRC — unblocking a critical integration

## Side Projects

**[NanoBotTS](https://github.com/Harry-Zhao-AU/NanoBotTS)** — Lightweight AI Agent Framework (TypeScript)

- Designed and built a full agent loop from scratch — LLM reasoning, tool use, result feedback, and parallel safe-tool execution — gaining deep hands-on understanding of how production agent systems work
- Architected a decoupled MessageBus separating I/O channels (CLI, Telegram) from the reasoning core, with a two-tier memory system (LLM-consolidated long-term facts + session JSONL history) and token-aware context management
- Built a ProviderRegistry supporting Azure OpenAI, OpenAI, Groq, and Ollama, plus background autonomous services (scheduling, heartbeat monitoring, sub-agent spawning) — demonstrating end-to-end agent system design

## Get in Touch

- [Email](mailto:harry.zhao19@gmail.com)
- [GitHub](https://github.com/Harry-Zhao-AU)
- [LinkedIn](https://www.linkedin.com/in/harryzhao19)
