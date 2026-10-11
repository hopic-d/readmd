# Example Root Cause Analysis

## Incident

A nightly job failed after an upstream API returned malformed data.

## Root Cause

The upstream client assumed all fields were present in every response. A schema change introduced optional values, and the parser did not guard against missing keys.

## Impact

- Failed nightly processing
- Delayed downstream reports
- Increased manual investigation time

## Corrective Actions

- Added defensive parsing for optional fields
- Added regression tests for schema variation
- Added monitoring for malformed API responses
