---
title: "Household Hub — from accountability engine to something people actually use"
entry_type: note
subtype: diary
date: 2026-07-19
author: mdp
projects: [casehub-life]
tags: [ui, design, sse, household-hub]
issue: "#74"
---

# Household Hub — from accountability engine to something people actually use

Eight layers of foundation work later, casehub-life has SLA enforcement, commitment tracking, trust-weighted routing, GDPR erasure, case-based reasoning, and 32 LLM-backed workers across 8 case plans. What it doesn't have is a way for anyone to see any of it.

That's the gap this session starts to close. The question that kicked it off was simple: how would a human actually use this? There's no UI. No inbox. No dashboard. No way for a household member to know what needs their attention, what the system decided on their behalf, or what's stuck.

## Four concerns, not three

I started with three layers — operational dashboard, personal task inbox, decision journal. The first thing that emerged was a fourth: management. Not system-ops visibility, but per-person task inboxes filtered by role, pending approvals, delegations waiting for response. The inbox is where oversight gates, family votes, and watchdog alerts land as actionable items for a specific person.

Then a fifth concern surfaced: ambient intake. The system shouldn't just wait for explicit commands — it should listen to WhatsApp, email, and calendar, classify incoming messages using neocortex's CBR and memory, and create structure automatically. A message from the plumber saying "I'll come Thursday at 2pm" becomes a calendar entry, updates the contractor case, and notifies the household.

The product concept crystallised as "Household Hub" — one web app where 3-5 household members go for all their life needs. Structured views for observation, embedded Claudony for conversational interaction, bidirectional sync with Google Calendar/Contacts/Tasks so the system meets people where they already are.

## Design principle: own the model, project into external systems

The key architectural decision: life owns the rich domain model (trust scores, SLA, commitments, audit trail, CBR), but external systems (Google Contacts, Calendar, Tasks) are the day-to-day interaction surface for the user. ExternalActor syncs bidirectionally with Google Contacts — life enriches with trust and case history; Google owns name and phone number. A WorkItem deadline becomes a Google Calendar event. Completing a task in Google Tasks completes the WorkItem.

For demos, everything ships standalone with pre-populated data. For production, you connect your Google account and the system integrates with what you already use.

## Backend landed, frontend deferred

We got the backend MVP done in four commits: `LifeCaseType.domain()` mapping with a V111 migration adding the domain column to `LifeCaseTracker`, GET `/life-cases` list and detail endpoints with `LifeCaseVisibilityPolicy` (juniors filtered, admin and member see everything), SSE event infrastructure with a `LifeEventBroadcaster` bridging CDI events to Server-Sent Events streams, and a Quarkus demo profile with Flyway seeds at V9000+.

The SSE design settled on snapshot-on-reconnect — no event replay, no Last-Event-ID tracking. When a client reconnects it gets the full current state. Simpler than replay and honest about what SSE can guarantee with `DROP` overflow.

The frontend module (Lit SPA via Quinoa composing blocks-ui components) we deliberately deferred. blocks-ui has an impressive component library — work-item-workbench, KPI rows, trust panels, approval gates, notification inbox, channel feeds — but we need to verify the packages are actually published to GitHub Packages before writing the composition layer. Better to get the wiring right than rush a build that can't resolve its dependencies.

The IoT webapp can be embedded or linked for house management. Claudony provides the conversational surface for human-in-the-loop decisions — or xterm.js for power users who prefer a terminal.

## What closed, what opened

Engine#738 (PlanAdapter wiring) landed, which meant #52 (the CBR epic) could finally close — all five child issues done, all upstream dependencies resolved. That's Layer 8 fully complete.

#74 (Household Hub Phase 1) is the new epic. The backend portion is done. Task 5 (life-ui frontend) is next session's work, once we've confirmed blocks-ui package availability.

The interesting thing about this session is the shift in perspective. Eight layers of foundation work were all about the engine — what the system *does*. This is the first time we're designing what the system *looks like* to the people it serves. The accountability properties don't change, but the value proposition does: from "formally tracked, SLA-enforced, tamper-evident" to "one place where your household just works."
