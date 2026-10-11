# Example Issue

## Summary

A deployment unexpectedly failed in production due to a configuration mismatch.

## Context

- Service: API gateway
- Environment: production
- Affected window: 2026-10-11

## Findings

The service was configured with an outdated connection string and the rollout process did not validate the target environment before promotion.

## Resolution

- Updated the configuration values
- Added a pre-deploy validation check
- Documented the RCA in the repository
