---
layout: post
title: "The first harness with real auth — two surprises on the way"
date: 2026-06-22
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-life]
tags: [oidc, security, rbac, quarkus, casehub-platform]
---

`CurrentPrincipal.groups()` has always returned an empty set in casehub-life. Every harness
is in the same state — casehub-platform-oidc exists and is production-ready, but nobody had
actually wired it yet. That changes today. Life#40 closes with `OidcCurrentPrincipal` active in
production, `@RolesAllowed` on all five REST resources, and RBAC-differentiated risk thresholds
in the classifier (life#26 bundled in — there was no point implementing per-role thresholds
while groups() was permanently empty).

Two things didn't go the way I expected.

## TenantScopedPrincipal was silently winning

I assumed the only gap was the missing dep. Add casehub-platform-oidc, configure Quarkus OIDC,
done. Claude flagged what I'd missed: `TenantScopedPrincipal @RequestScoped @Unremovable` from
casehub-work is already on the classpath and also implements `CurrentPrincipal`. Neither bean is
a `@DefaultBean`, neither is `@Alternative`. Two `@RequestScoped` implementations of the same
interface — Quarkus throws `AmbiguousResolutionException` at startup. The comment in the original
`application.properties` even said "excluded here so TenantScopedPrincipal wins," which told me
the previous exclusion list had been written optimistically, in anticipation of OIDC landing, but
the exclusion entry for TenantScopedPrincipal itself was never written. Adding it to
`quarkus.arc.exclude-types` resolved it. Obvious in retrospect; completely invisible before.

## Dev mode was still broken

`%dev.quarkus.keycloak.devservices.enabled=false` is the standard fix for "don't start a
Keycloak container in dev." But with `@RolesAllowed` on all endpoints, that's only half the
picture. Endpoints still return 401 because Quarkus security enforcement runs independently
of which auth mechanisms are configured — disabling DevServices doesn't disable the
authorization interceptor.

The actual fix is `%dev.quarkus.security.auth.enabled-in-dev-mode=false`. It activates
`DevModeDisabledAuthorizationController` in quarkus-security-runtime-spi, which makes
`isAuthorizationEnabled()` return false. Both `EagerSecurityHandler` (the REST path) and
`StandardSecurityCheckInterceptor` (the CDI path) check that flag and bail early. Set it
alongside `%dev.quarkus.oidc.enabled=false` and dev mode is fully open again. I found this
by reading the 3.32.2 source jar — it's not in any documentation I could find.

## The classifier

With real groups() in play, wiring RBAC thresholds into `LifeActionRiskClassifier` was
straightforward. Admin gets elevated thresholds (spend: £500 vs £100, contractor: £500 vs £200).
Junior — anyone who is neither admin nor member — always gates on AMOUNT_THRESHOLD types
regardless of amount.

The interesting design decision was the negative definition. I defined `isJunior(boolean admin)`
as "not admin AND not member" rather than "contains JUNIOR." An unknown role — a service account,
a new household type that doesn't exist yet — gets the most restrictive treatment automatically.
A financial gate that silently grants spending authority to unrecognised identities is the wrong
failure mode.

The `admin` flag gets computed once and passed down through the classification chain. Claude
caught that the original draft called `principal.groups()` twice per classification — once for
admin, once for member — through two separate proxy lookups. With `ContextNotActiveException`
possible on each (workers fire on the scheduler, no request context), one is preferable to two.
The refactor was a one-parameter change.

## What this enables

The same wiring pattern goes into casehub-devtown and casehub-clinical next — the protocol
exists now. Life was the reference implementation; the other two harnesses are
parameter-substitution at this point.

Life#26 was the first actual consumer of `groups()` doing real work. Life#41 (junior's
data-level task filter — own-tasks-only reads) is the next gap that auth wiring opened up.
