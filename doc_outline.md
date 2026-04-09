# MilkSafe 5-Port Reader Development Specification

## Recommended Workflow

Use this order:

1. Review and freeze the section structure.
2. Review core rules and flow behavior.
3. Review local/cloud and API behavior.
4. Add final payload examples and endpoint details from Swagger.
5. Add the short implementation appendix.
6. Convert the approved markdown into `doc.docx`.

Why this workflow:

- Figma and the prototype already explain most navigation and screen layout.
- This document should focus on the rules that are easy to implement incorrectly.
- API examples should be added after the behavior is agreed, not before.

## Proposed Structure

## 1. Purpose

- what this document is for
- how it relates to the high-level requirements
- how it complements Figma and the prototype

## 2. Source Links

- high-level requirements reference
- production Swagger
- prototype URL
- Figma URL

## 3. Glossary

- sample ID = backend route
- group = flow
- test record
- grouped test record
- verification
- anonymous mode
- logged-in mode

## 4. Core Device Rules

- 5 channels
- shared heating plate
- moving scanner
- same-channel confirmation
- global temperature states
- no GPS upload
- no negative control

## 5. Login and User Modes

### 5.1 Logged-In Mode

- same user type as DRC and MobileLabs
- user creation in MilkSafe Cloud web
- site-linked configuration

### 5.2 Anonymous Mode

- local capture
- obfuscated upload
- not synced history state
- reassignment after login for grouped tests
- no reassignment for verification

## 6. Test Types and Configuration

- bundled backend JSON for offline use
- refresh on device restart
- site-specific config for logged-in users
- local config for anonymous users
- global QR setting
- QR-prefilled type is not manually editable

## 7. Test Execution Rules

- supported run types
- incubation rules
- temperature validation
- repeat-incubation rule
- cassette handling

## 8. Confirmation Flow

- maximum 3 tests
- outcome rules
- positive / negative / inconclusive paths

## 9. Modal Queue Behavior

- queued result/decision modals
- one visible modal at a time
- no lost pending decisions

## 10. Controls

- positive control
- animal control
- no follow-up verification flow from failed controls

## 11. Verification

- started from Settings
- slot choice behavior
- pass criteria
- upload behavior
- anonymous verification endpoint target
- verification threshold rules
- overdue startup warning modal note
- separate verification history

## 12. History, Comments, and Detail Screens

- grouped test/control history
- separate verification history
- group-level comments
- test detail requirements
- verification detail requirements
- local-only light-intensity chart for normal tests

## 13. Local vs Cloud Behavior

- logged-in grouped tests
- anonymous grouped tests
- verification records
- factory reset behavior

## 14. API Integration

- main endpoint set
- grouped upload only after flow completion
- do not use deprecated single-record upload endpoints
- grouped test payload strategy
- verification payload strategy
- mobile applications as behavior reference
- planned `DeviceHealth/anonymous`

## 15. Settings

- password-protected settings
- non-protected actions
- configuration rules that matter for implementation

## 16. Localization

- Tolgee workflow
- Figma access via InFlow Software
- supported languages

## 17. Quantitative Tests and Aflatoxin

- quantitative rules
- Aflatoxin handling
- calibration loading

## 18. Export, Printing, and LINS

- CSV
- Excel
- local CSV/Excel structure from `export.csv`
- LINS
- print behavior

## 19. Out of Scope

- negative control
- annotation editing
- GPS location upload
- restoring history from cloud after reset

## 20. Implementation Appendix

- environments
- authentication and login sequence
- grouped test upload example
- anonymous grouped upload rules
- verification upload example
- planned `DeviceHealth/anonymous`
- `readerData` handling
- local export notes
