---
name: spotify-migration-architect-fe
description: Use when migrating legacy Spotify API to custom Spring Boot backend and structuring React/Redux components using Vertical Slicing.
---

# SKILL: Spring Boot & React Strangler Fig Migration

## Description
Expert architect and developer skill for migrating legacy Spotify API integrations to a custom Spring Boot 3 + Keycloak backend while preserving existing React/Redux frontend structures using Vertical Slicing.

## Core Directives
1. **Strangler Fig Pattern:**
    - Gradually replace endpoints pointing to Spotify API with custom Spring Boot backend endpoints.
    - Leave untouched legacy modules until they are explicitly marked for migration.

2. **Vertical Slicing & Folder Structure:**
    - Group code by features/slices (e.g., `account`, `catalog`).
    - Never create unapproved folders at the root directory. Follow existing naming conventions (PascalCase for components, camelCase for functions/hooks).

3. **Backend & Security Alignment:**
    - All API calls must include `Authorization: Bearer <token>` via Axios Interceptors.
    - Respect Spring Boot DTO contracts and pagination standards.

4. **Safety & Control Constraints:**
    - **NEVER** auto-commit or run git commands.
    - **NEVER** install new npm packages (`npm install`) without explicit user permission.
    - Always list files to be created or modified before applying large changes.