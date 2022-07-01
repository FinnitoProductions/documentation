---
id: what-is-authorizer
title: What is Authorizer?
sidebar_label: What is Authorizer?
---

The `Authorizer` contains a single `Authorize` method, which is invoked for each incoming API call. `Authorize` receives information about the API call, along with the role and permission claims of the caller.

Authorizer allows for a wide range of authorization logic, including call target, role/permissions claims, and other data available to the system.

## Configuration

The following arguments must be passed to the Authorizer:

- `context.Context`: General context of the call.
- `authorization.Claims`: Claims about the roles assigned to the caller. Its intended use is [described below](#claims).
- `authorization.CallTarget`: Target of the API call.

Authorizer then returns one of two decisions:

- `DecisionDeny`: the requested API call is not invoked and an error is returned to the caller.
- `DecisionAllow`: the requested API call is invoked.

Configure your `Authorizer` when you start the server via the `temporal.WithAuthorizer` [server option](/server/options/#withauthorizer).

If an `Authorizer` is not set in the server options, Temporal uses the `nopAuthority` authorizer that unconditionally allows all API calls to pass through.
