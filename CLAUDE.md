# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Overview

This is a technical documentation repository containing learning materials for cloud infrastructure concepts, written in Chinese. There is no code, build system, or tests.

## Content Structure

- **iam/** - IAM (Identity and Access Management) learning guide with 7 progressive chapters covering authentication/authorization fundamentals, policy models, credentials, assume role, federated identity, workload identity, and best practices
- **k8s/** - Kubernetes learning guide covering container orchestration fundamentals and architecture

## Writing Guidelines

When editing or adding documentation:

- Follow the existing "first principles" (第一性原理) teaching approach - explain the "why" before the "what"
- Use ASCII diagrams for visual explanations (see existing files for style reference)
- Structure content in progressive layers (Layer 1, Layer 2, etc.) building from fundamentals to advanced topics
- Include comparison tables for related concepts
- Write in Chinese, matching the existing content language
- Number files sequentially (e.g., `01-topic.md`, `02-topic.md`)
- Each topic area has a `*-learning-roadmap.md` file that serves as the index and overview
