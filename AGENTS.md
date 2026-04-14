# 5-Port Reader Documentation Repository

## Purpose

This repository is for writing one concise, developer-facing implementation document for the MilkSafe 5-port reader.

- The main deliverable is `doc.docx`, which will later be mirrored into Google Docs.
- The document should stay actionable, not overly long.
- It should describe product requirements, user flows, technical behavior, backend integration, selected API payloads, and diagrams needed for implementation.
- It should not speculate when product sources conflict. Resolve conflicts with the user first, then document the resolution here and in `doc.docx`.
- Do not describe the device as "Android" in the product spec unless the user explicitly asks for platform details.

## Current Source Map

### In this repository

- `High level requirements.docx`
- `openapi.json`
- `export.csv`

### External reference URLs

- Production Swagger URL: `https://chrhansenmilksafefunctions-prod.azurewebsites.net/api/swagger/ui#/DeviceHealth/post_api_webapi_v2_DeviceHealth`
- Prototype URL: `https://5-port-reader-prototype.milksafe-preview.workers.dev/`

### Nearby prototype repository

Path:

- `/Users/butcha/Developer/Milksafe/misc/5port_reader_prototype`

Relevant files actually present there:

- `/Users/butcha/Developer/Milksafe/misc/5port_reader_prototype/claude.md`
- `/Users/butcha/Developer/Milksafe/misc/5port_reader_prototype/docs/reader_spec.md`
- `/Users/butcha/Developer/Milksafe/misc/5port_reader_prototype/docs/device_flow/README.md`
- `/Users/butcha/Developer/Milksafe/misc/5port_reader_prototype/docs/confirmation_flow/COMPLETE_DEVICE_FLOW.md`
- `/Users/butcha/Developer/Milksafe/misc/5port_reader_prototype/docs/confirmation_flow/CONFIRMATION_FLOW_OUTCOMES.md`

Note:

- The prototype repo includes `claude.md` with prototype-level device context and an estimated `1280x800` display spec.
- There is still no `agents.md` file in the prototype repo.

### iOS reference application

Path:

- `/Users/butcha/Developer/Milksafe/milksafe-ios-app`

Important reference areas:

- Authentication and session handling
- Anonymous mode behavior
- Test type loading and site configuration
- Grouped test upload and retry sync
- Confirmation-flow result calculation
- QR validation and cassette reuse checks
- Localization language set

## Product Context Confirmed So Far

- The new device is a hardware reader with 5 ports that reads milk-testing cassettes.
- It is the multi-port counterpart to the existing single-slot Desktop Reader Connect (DRC).
- The new device should preserve the core domain behavior of DRC and the mobile apps, while improving UX/UI and supporting 5 simultaneous channels.
- Each channel can run its own test flow independently.
- Confirmation flow always stays in the same physical channel.
- The system groups related test records into one flow/group and uploads that grouped result to the cloud.
- The 5-port reader must upload completed flows through grouped test upload only; it must not upload normal test records one by one through deprecated single-record endpoints.
- In this documentation, `group` and `flow` mean the same thing.
- In backend terminology, UI `sample ID` maps to backend field `route`.
- A group can contain up to 3 tests in the current intended confirmation flow.
- Logged-in and anonymous operation are both required.
- Logged-in users authenticate with username and password.
- Account creation happens outside the device.
- Customers can have multiple sites.
- Site configuration determines which test types are enabled for logged-in users.
- Anonymous users should be able to enable or disable test types locally on the device.
- In anonymous mode, test records still capture the entered sample ID and operator ID locally on the device.
- Test groups are cached locally and uploaded when connectivity is available.
- Test history is local to the device; product direction currently says history is uploaded but not downloaded back after reset.
- Anonymous grouped test records are uploaded in obfuscated form through the anonymous endpoint and should appear in device history as `not synced`.
- Anonymous grouped test records keep the reader serial number; that field is not obfuscated.
- When the user later logs in on the same device, previously anonymous grouped test records should be reassigned to that user and uploaded normally to their site, mirroring the current iOS behavior.
- Comments are intended to exist at the group/flow level, not as editable per-test annotations on the device.
- Positive control and animal control are required.
- Positive control and animal control are single-test control flows with no confirmation flow.
- A positive control that does not produce a positive result should simply be shown as a failed control state with a cross; it does not trigger verification or any extra flow.
- Animal control uses the same runtime behavior as positive control, but with a different control annotation/type.
- Verification is required.
- Verification is run from Settings via a dedicated `Run verification` action, not from the normal run-test flow.
- Verification passes when the measured ratio is within `0.9` to `1.1`.
- Verification includes temperature measurement and light intensity.
- Verification uses one light-intensity value and should fail if that value is at or below `500000`.
- Verification uploads use the device-health verification flow and should still upload in anonymous mode.
- The reader has one scanner moving horizontally across the slots and one heating plate.
- Verification therefore does not need to be repeated for every slot.
- The user may choose which slot to use for verification; slot `1` is the default, but the slot choice is not saved and has no backend significance.
- The reader cannot reliably detect cassette removal in all situations; insertion detection is the dependable event.
- The reader does not know the user's GPS location; product direction currently says upload empty location like DRC.
- Final display resolution for the device spec is `800x480`.
- Verification history is separate from test/control history and is accessed from Settings.
- Negative control is not part of the 5-port reader scope.
- The temperature rule should be documented explicitly as a global heater setting for the whole device with states `Off`, `40 C`, or `50 C`, and software should validate compatibility against the selected test type.
- Cloud device-health logic uses a default verification threshold of `250` aggregated tests across all five slots.
- The user may lower the local warning threshold on the device to any value at or below `250` in Settings, but the cloud/system default remains `250`.
- When the device exceeds the threshold, the current product baseline is a warning in Settings; a startup warning popup is acceptable but not yet a locked prototype requirement.
- The intended behavior is also a small startup warning modal when the device is overdue for verification, but this is not implemented in the current prototype.
- Individual test-detail screens should include a light-intensity chart for each test record.
- Light intensity for normal test records is local-only device data used for the test-detail chart and is not uploaded to the backend.
- Anonymous verification records should still be uploaded to the anonymous data site/customer, but they are not reassigned to the user after login.
- CSV and Excel exports are local device exports, not backend export endpoints.
- CSV and Excel should use the same structure, based on `export.csv` in this repository.
- LIMS integration/export is also required, but the exact implementation is intentionally left to BioEasy.
- Known backend/app environments are development, staging, pre-production, and production.
- Pre-production is the recommended main testing environment and product direction says it is refreshed from production on the first day of each month.
- Authentication in the mobile application uses Azure B2C ROPC token exchange, stores both access and refresh tokens locally, and refreshes access on `401` using the refresh token.
- Do not upload `readerData` from the 5-port reader.
- Date and time must always be shown on the device in the user-configured local timezone, and uploads must include the correct timezone offset so cloud views can display the correct local test time.

## Languages / Localization

The 5-port reader should follow the same language coverage as the existing iOS app unless product direction changes.

Languages currently present in the iOS app:

- `en`
- `de`
- `el`
- `uk`
- `es`
- `da`
- `et`
- `it`
- `sk`
- `hu`
- `tr`
- `pl`
- `lt`
- `ru`
- `fr`
- `fi`
- `pt`

Product direction from the user:

- Text keys and English base values should be managed in Tolgee.io first.
- Developers should define the translation key structure and English source text there.
- The exported format can then be adapted to whatever the device implementation needs.
- InFlow Software will create a Tolgee.io project and assign edit permissions to BioEasy developers

## iOS Implementation Notes Worth Reusing

These are reference behaviors from the iOS app. Reuse them where they match the 5-port reader product, but do not blindly copy UI behavior when the prototype or user direction says otherwise.

### Authentication and session

- Login uses Azure B2C token exchange and then fetches the current user from `/Users/me`.
- The iOS app only accepts users of type `Operator` for login.
- After successful login, the app stores:
  - username
  - user id
  - site id
  - shared-user flag
  - email
- The app supports continue-without-login mode.
- When an anonymous user later logs in, the app migrates anonymous cached test groups and route/operator suggestions to the logged-in user.

### Test type loading

- Logged-in behavior:
  - fetch current user
  - use site id
  - fetch site data from `/Sites/{id}`
  - fetch all test types from `/TestTypes`
  - combine the site's enabled test types with the full test type list into per-type config
- Anonymous behavior:
  - fetch all test types
  - cache local per-type config on device
- In iOS, a `TestTypeConfig` determines:
  - whether a test type is enabled
  - whether QR scanning is enabled for that test type - this is not case for 5 port reader, 5 port reades has a global QR on and off setting in the Settings

### Grouped test upload and retry

- The iOS app uploads grouped flows, not just isolated records, via:
  - `POST /api/webapi/v2/GroupedTestRecords`
  - `POST /api/webapi/v2/GroupedTestRecords/anonymous`
- The sync engine retries unsynced groups every 60 seconds.
- Anonymous upload in iOS changes some fields:
  - `route` becomes `nil`
  - `operatorId` becomes the local `deviceUUID`
  - location is obfuscated if location exists
- For the 5-port reader, current product direction says to upload empty location like DRC, so treat iOS location handling as reference only.

### QR and cassette validation

- The iOS app validates that the scanned cassette QR matches the selected test type.
- It blocks reuse of the same cassette:
  - within the current flow
  - across previously used cassette serials cached on device
- QR-derived cassette data includes:
  - test type code
  - batch number
  - serial number
  - manufacturing date derived from batch format

### Group / flow result calculation

The iOS domain model currently implements the 3-step confirmation logic consistent with the intended 5-port confirmation flow:

- Test 1 negative => group negative
- Test 1 positive, user stops => group inconclusive
- Test 2 positive => group positive
- Test 2 negative, user stops => group inconclusive
- Test 3 positive => group positive
- Test 3 negative => group negative

Important distinction:

- In the current iOS UI, positive control, animal control, and verification are marked after reading from the test record screen.
- In the 5-port reader prototype, the scenario is selected earlier as part of the run-test flow. This also means that a negative positive control can be uploaded to the cloud. In the iOS app this is impossible as you can only label a test as a positive control when the resutl if positive. This is fine. 
- In the 5-port reader, a failed positive control is still kept as a positive-control record and shown with a failed-state cross. Animal control follows the same runtime pattern with its own control type.
- For the new device, follow the new device flow, not the legacy iOS UI sequence.

### Verification logic in iOS

- The iOS app checks verification pass/fail by confirming every measured substance level is within `0.9...1.1`.
- Device usage count is reset after a verification test. The count is also counted in the backend and reset after a verification upload. It is important to upload the verificaton so that this count is reset on backend
- The iOS home screen warns about verification need based on usage count.

## OpenAPI Notes

OpenAPI is the contract reference, but it contains a wider surface than the implementation document will likely need.

For the 5-port documentation:

- Prefer the production Swagger URL above as the primary endpoint reference when writing final endpoint examples.
- Prefer the iOS app's minimal proven payload subset when documenting request examples.
- Use `openapi.json` to verify field names, enums, and endpoint availability.
- Only add extra OpenAPI fields to the spec when the 5-port requirements explicitly need them.
- The local `openapi.json` is useful for schemas and endpoint names, but it should not be treated as authoritative for production server/auth configuration.
- `DeviceHealth` is authenticated right now in production, but the target behavior for this project is to allow anonymous verification uploads as well. Document the target behavior and call out the server-side dependency where needed.

### Primary endpoints likely needed in the spec

- `POST /api/webapi/v2/GroupedTestRecords`
- `POST /api/webapi/v2/GroupedTestRecords/anonymous`
- `GET /api/webapi/v2/TestTypes`
- `GET /api/webapi/v2/Users/me`
- `GET /api/webapi/v2/Sites/{id}`
- `GET /api/webapi/v2/Firmware/{type}/latest`
- `POST /api/webapi/v2/GroupedTestRecords/{id}/comment`

### Endpoints not to use for the 5-port reader

- `POST /api/webapi/v2/TestRecords`
- `POST /api/webapi/v2/TestRecords/anonymous`

These old single-record uploads are not the intended integration path for the new device.

### Useful grouped-test payload fields

From `CreateGroupedTestRecordCommand` and the iOS upload model, the main fields worth documenting are:

- group-level:
  - `customerId`
  - `siteId`
  - `comments`
  - `appVersion`
  - `testRecords`
- per test record:
  - `testDate`
  - `testDateOffset`
  - `readerSerialNumber`
  - `route`
  - `result`
  - `testTypeId`
  - `operatorId`
  - `substances`
  - `annotation`
  - `manufacturingDate`
  - `batchNumber`
  - `rawQR`
  - `cassetteId`
  - `appVersion`
  - `temperature` if verification/temperature payload is required by final product design

## Writing Rules For The Main Specification

- Keep the document concise and implementation-oriented.
- Do not make it sound like AI
- Write the document as direct implementation guidance for developers building the 5-port reader software.
- Do not narrate source lookup inside the specification. Do not write phrases such as "this is what was found in iOS" or similar source-reporting language. Resolve the rule first, then document the resulting requirement directly.
- Prefer short sections, bullets, and brief explanatory paragraphs.
- Include diagrams only where they remove ambiguity.
- Include endpoint examples and payload examples only for the fields developers actually need.
- Prefer proven behavior from existing MilkSafe implementations over OpenAPI breadth.
- When a product rule differs from iOS or DRC, document the difference explicitly.
- If a source is contradictory, do not resolve it by assumption. Record the conflict and get user clarification first.
- Keep any implementation appendix short and limited to login/auth sequencing, upload examples, anonymous-upload rules, verification upload, and local export notes.

## Document Additions To Include

When drafting `doc.docx`, include these supporting sections because they remove ambiguity without making the document too long.

- A short glossary:
  - `sample ID` in the UI = `route` in backend payloads
  - `group` = `flow`
- A `Source Links` section for:
  - high-level requirements document
  - Swagger / API reference
  - prototype URL
- A `Local vs Cloud Behavior` section for:
  - logged-in test groups
  - anonymous test groups
  - verification records
  - behavior after login
  - behavior after factory reset
- An `Out of Scope` section including:
  - negative control
  - device-side annotation editing
  - GPS-based location upload
- A dedicated section for quantitative tests, especially Aflatoxin-related behavior.
