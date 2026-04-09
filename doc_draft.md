# MilkSafe 5-Port Reader Development Specification

## 1. Purpose

This document is the implementation specification for the MilkSafe 5-Port Reader.

It is a more detailed version of the high-level requirements document and should be used to explain the rules and behaviors that are not obvious from the design alone.

Use this document for:

- product rules
- flow rules
- data handling
- local vs cloud behavior
- backend integration
- terminology

Do not use this document to restate visual design that is already clear in Figma or in the prototype.

## 2. Source Links

- High-level requirements reference: `https://docs.google.com/document/d/1169Cg05CEVyzrp3IAxD_V3O6Rp4qwvTSVriLLasQL_0/edit?usp=sharing`
- Production Swagger: `https://chrhansenmilksafefunctions-prod.azurewebsites.net/api/swagger/ui#/DeviceHealth/post_api_webapi_v2_DeviceHealth`
- Interactive prototype: `https://5-port-reader-prototype.milksafe-preview.workers.dev/`
- Figma design: `https://www.figma.com/design/3enH8QjK0fSqVMVGbe3wte/MilkSafe?node-id=3351-9971`

Design usage rules:

- use Figma as the design source of truth
- use the prototype for intended behavior when in doubt
- if implementation developers need Figma access, they should request it from InFlow Software

## 3. Glossary

- `Sample ID` in the device UI = backend field `route`
- `Group` = `flow`
- `Test record` = one individual read of one cassette
- `Grouped test record` = one flow containing 1 to 3 related test records
- `Verification` = separate device-health verification flow, not a normal grouped test flow
- `Anonymous mode` = device used without login, with obfuscated upload behavior
- `Logged-in mode` = device used with MilkSafe Cloud credentials and site-linked configuration

## 4. Core Device Rules

- The device has 5 visible test channels on one screen.
- Each channel can run independently.
- A confirmation flow always stays in the same physical channel as the original test.
- The device has one shared heating plate for all channels.
- The device has one scanner moving horizontally across the channels.
- Display resolution is `800 x 480`.
- The device-wide heater states are `Off`, `40 C`, and `50 C`.
- The reader does not know GPS location. Upload empty location like DRC.
- Negative control is not implemented.

## 5. Login and User Modes

### 5.1 Logged-In Mode

The same user type that is used to log in to DRC and MobileLabs should also log in to this device.

These users and their passwords are created for the site in MilkSafe Cloud web.

Logged-in behavior:

- user signs in with username and password
- the device loads the user's site-linked test type configuration
- grouped test records upload normally to the user's site
- verification records upload normally to device-health/backend

### 5.2 Anonymous Mode

Anonymous behavior:

- the user can continue without login
- test types are configured locally on the device
- sample ID and operator ID are still entered and stored locally
- grouped test records upload through the anonymous grouped-test endpoint in obfuscated form
- reader serial number is still uploaded and is not obfuscated
- anonymous grouped test records are shown in device history as `not synced`

Anonymous reassignment behavior:

- if the user later logs in on the same device, previously anonymous grouped test records should be assigned to that user and uploaded normally to their site
- this mirrors the current mobile applications behavior

Anonymous verification behavior:

- anonymous verification records should also upload
- they are not reassigned after login

## 6. Test Types and Configuration

### 6.1 Source of Test Types

Test types must be bundled into the device software as a JSON file using the exact JSON response from the backend test-types endpoint that is current at the time of development.

Reason:

- some devices may never connect to the internet
- the bundled test types must preserve the real backend IDs

### 6.2 Offline and Refresh Behavior

Recommended behavior:

- bundle the full test-types JSON in the software
- on every device restart, attempt to download fresh test types
- if logged in, also load the site-specific configuration
- when fresh test types are successfully loaded, replace the bundled/cached set with the current backend version

Anonymous mode:

- load all test types
- allow local enable/disable

Logged-in mode:

- load all test types
- apply site-enabled test type configuration from cloud
- the device should not let the user override site enablement locally

### 6.3 QR Setting

The 5-Port Reader uses a global QR scanning setting.

Rules:

- QR scanning can be turned on or off in Settings
- when QR scanning is enabled and the QR is read correctly, the test type is prefilled in the configuration modal
- when the test type is prefilled from QR, the user must not be able to change it manually
- if QR scanning is disabled, the user selects the test type manually
- if the scanned test type is not enabled for the site or local anonymous configuration, the device must show a warning and block the run

## 7. Test Execution Rules

### 7.1 Available Run Types

The device supports these run types from the normal test flow:

- Normal test
- Positive control
- Animal control

Verification is not started from the normal run-test flow. It is started from Settings.

### 7.2 Incubation Rules

Incubation is not determined by cassette type alone.

Correct rule:

- incubation is only allowed for test types that have temperature and incubation time configured in backend test-type data
- if the selected test type does not have the required incubation configuration, do not offer incubation
- if the selected test type does have temperature and incubation time configured, allow `Read and incubate`

Temperature validation:

- the device temperature is set manually by the user at device level
- when the cassette is identified, the device must validate the current heater state against the selected test type configuration
- if the device temperature is outside the allowed range for that test type, block the test and show a warning

### 7.3 Repeat Incubation Rule

For normal tests only:

- if the first test result is positive or invalid and that test was incubated, repeat incubation for 2 more minutes
- judge the outcome based on the repeated reading

This rule does not apply to controls.

### 7.4 Cassette Handling

- insertion can be detected reliably
- cassette removal cannot be relied on in all cases
- the user can interrupt incubation
- if incubation is interrupted, the cassette must be removed manually
- the device does not eject the cassette

## 8. Confirmation Flow

Confirmation flow applies to normal tests only.

Rules:

- maximum 3 tests per flow
- all tests stay in the same channel
- the group/flow result is determined from the completed flow

Outcome rules:

1. Test 1 negative -> final flow result is `Negative`
2. Test 1 positive -> confirmation required
3. If any confirmation test is positive -> final flow result is `Positive`
4. If both confirmation tests are negative -> final flow result is `Negative`
5. If the user stops before the flow reaches a natural conclusion -> final flow result is `Inconclusive`

Detailed paths:

- Test 1 negative -> Negative
- Test 1 positive, user stops -> Inconclusive
- Test 1 positive, Test 2 positive -> Positive
- Test 1 positive, Test 2 negative, user stops -> Inconclusive
- Test 1 positive, Test 2 negative, Test 3 positive -> Positive
- Test 1 positive, Test 2 negative, Test 3 negative -> Negative

Upload timing:

- do not upload individual test records one by one
- keep the flow locally until the final flow result is known
- upload the whole flow through grouped test upload only after the flow is complete
- if Test 1 is negative, the flow is complete and can be uploaded immediately
- if confirmation is required, upload only after the user aborts or after the final confirmation result is known

## 9. Modal Queue Behavior

This is a critical runtime behavior and must be implemented explicitly.

Multiple channels can finish or require user action at nearly the same time.

Example:

- channel 1 finishes positive and needs a confirmation decision
- channel 2 also finishes and needs a modal
- channel 3 also needs a modal

Required behavior:

- decision modals must be queued
- only one modal is visible at a time
- after the user resolves the modal for one channel, the next waiting modal must open immediately
- queued modals must not be lost
- the queue must preserve all pending user decisions

This applies especially to:

- confirmation continuation / abort decisions
- any result-driven modal that requires immediate user input

## 10. Controls

Controls are single-test flows and do not use confirmation flow.

### 10.1 Positive Control

- selected before the run starts
- uploaded as a control record
- no confirmation flow
- show the control status directly in the UI
- if the result is positive, show the success state
- if the result is not positive, show the failed state with a cross
- do not trigger verification or any additional flow from this result

### 10.2 Animal Control

- selected before the run starts
- uploaded as a control record
- no confirmation flow
- use the same runtime behavior as positive control
- use the animal-control annotation/type instead of the positive-control annotation/type

## 11. Verification

### 11.1 Entry Point

Verification is started from Settings using `Run verification`.

It is separate from the normal test flow.

### 11.2 Slot Behavior

- the user may choose which slot to use for verification
- slot 1 is the default
- slot choice is not saved anywhere
- slot choice has no backend meaning

### 11.3 Pass Criteria

Verification includes:

- ratio measurement
- measured temperature
- device temperature
- one light-intensity number

Pass rules:

- every measured ratio must be within `0.9` to `1.1`
- measured temperature must be within `±2 C`
- light intensity must be greater than `500000`

### 11.4 Verification Upload

Verification uses the device-health flow, not grouped test upload.

Current and target backend behavior:

- current production `DeviceHealth` is authenticated
- backend will be extended
- a new endpoint will be added: `DeviceHealth/anonymous`
- it will use the same payload as `DeviceHealth`
- it will be used for anonymous verification upload, similar to anonymous test-record/group upload

This new endpoint is not yet in the current Swagger but should be treated as planned target behavior for implementation.

### 11.5 Verification History

- verification history is separate from test/control history
- it is accessible from Settings
- verification records are cached locally
- verification records are uploaded to backend
- after factory reset, local verification history is lost
- uploaded verification records remain in backend

### 11.6 Verification Threshold

Cloud behavior:

- the system default is `250` aggregated tests across all 5 slots
- backend/cloud device health treats devices above this value as outstanding for verification
- verification upload resets this count in backend

Device behavior:

- the user can set a lower local warning threshold in Settings
- the local threshold must be `250` or lower
- the cloud default remains `250`
- the minimum required UI behavior is a warning in Settings when the threshold is exceeded
- on device startup, the device should also show a small warning modal when verification is overdue
- this startup modal is not implemented in the current prototype, so it should be treated as intended behavior beyond the current prototype baseline

## 12. History, Comments, and Detail Screens

### 12.1 Test and Control History

- tests and controls are shown together
- records are shown within their groups/flows
- history only shows data taken on this device
- grouped results are cached locally and uploaded later when connectivity is available
- history is not downloaded back from cloud after reset

### 12.2 Verification History

- separate from test/control history
- accessed from Settings

### 12.3 Comments

- comments are group-level
- the user can comment on the whole group/history item
- comment presence should be indicated in grouped history
- annotation editing on the device is out of scope

### 12.4 Detail Screens

Test/control detail should include:

- result
- test type
- date/time
- user
- operator ID
- route / sample ID
- upload status
- measured substances and values
- port number
- local light-intensity chart

Important:

- light intensity for normal test records stays local on the device
- it is not uploaded to backend

Verification detail should include:

- verification result
- ratio values
- measured temperature
- device temperature
- light intensity
- print action

## 13. Local vs Cloud Behavior

### 13.1 Logged-In Grouped Test Records

- cache locally
- upload normally to the user's site
- retry upload when connectivity becomes available

### 13.2 Anonymous Grouped Test Records

- store full local values for local behavior and later reassignment
- upload through anonymous grouped-test endpoint in obfuscated form
- show as `not synced` in device history
- keep reader serial number un-obfuscated

When the user later logs in:

- assign cached anonymous grouped test records to that user
- upload them normally to the user's site

### 13.3 Verification Records

- upload through device-health flow
- anonymous verification uploads use the new `DeviceHealth/anonymous` target endpoint
- anonymous verification records are not reassigned after login

### 13.4 Factory Reset

- local test/control history is removed
- local verification history is removed
- uploaded grouped test records remain in backend
- uploaded verification records remain in backend
- historical records are not restored back onto the device from backend

## 14. API Integration

### 14.1 Endpoint Set

Main endpoints expected for this device:

- `GET /api/webapi/v2/Users/me`
- `GET /api/webapi/v2/TestTypes`
- `GET /api/webapi/v2/Sites/{id}`
- `POST /api/webapi/v2/GroupedTestRecords`
- `POST /api/webapi/v2/GroupedTestRecords/anonymous`
- `POST /api/webapi/v2/DeviceHealth`
- `POST /api/webapi/v2/DeviceHealth/anonymous` (planned backend extension)
- `POST /api/webapi/v2/GroupedTestRecords/{id}/comment`

Do not use:

- deprecated single-record upload endpoints such as `POST /api/webapi/v2/TestRecords`
- deprecated anonymous single-record upload endpoints such as `POST /api/webapi/v2/TestRecords/anonymous`
- this device must upload normal test results through grouped test upload only

### 14.2 Payload Strategy

Use the minimum payload that matches the required device behavior.

For grouped test upload, focus on:

- `customerId`
- `siteId`
- `comments`
- `appVersion`
- `testRecords`
- `testRecords` contains the full finished flow, not individual partial uploads

Per test record:

- `testDate`
- `testDateOffset`
- `readerSerialNumber`
- `route`
- `result`
- `readerData`
- `testTypeId`
- `operatorId`
- `substances`
- `annotation`
- `manufacturingDate`
- `batchNumber`
- `rawQR`
- `cassetteId`
- `appVersion`

`readerData` rule:

- `readerData` is the raw byte response from the reader, encoded to base64 for upload
- do not transform it into a custom JSON structure before upload
- treat it as opaque raw device output

Date/time rule:

- `testDate` and `testDateOffset` must represent the device-local date/time and the configured timezone offset at the time of testing
- do not upload timestamps without timezone context
- this is required so the cloud can display the test time correctly in the timezone in which the test was taken

For verification upload:

- `readerSerialNumber`
- `testDate`
- `testDateOffset`
- `comments`
- `result`
- `readerData`
- `operatorId`
- `substances`
- `deviceType`
- `temperature`
- `readerSoftwareVersion`
- `appVersion`

### 14.3 Mobile Application Reference

Where backend behavior is shared, follow the same implementation pattern as the current mobile applications for:

- authentication
- grouped test upload
- anonymous grouped-test reassignment after login
- related cloud data handling

## 15. Settings

Only document settings rules that matter for implementation.

Password-protected:

- QR scanning
- Microswitch
- Verification threshold
- Test types
- MilkSafe Cloud login
- Date and time
- Language
- Software update
- Factory reset
- Temperature
- Verification history
- Internet connection

Not password-protected:

- Run verification
- Load quant curve
- About

Other settings/toggles:

- Printer on/off
- Comments on/off
- Route / Sample ID on/off
- Operator ID on/off
- Incubator on/off
- LIMS on/off
- Sound on/off

Date/time:

- the user sets date, time, and timezone
- display date and time in the device using the user-selected local timezone
- always upload timestamps with timezone data
- always send the correct `testDateOffset` that matches the timezone configured on the device, for example `+02:00`
- backend/cloud must receive timezone information so test times can be shown correctly in the timezone in which the test was taken

Software update:

- USB update
- internet update

## 16. Localization

- use Tolgee.io for localization workflow
- InFlow Software creates the Tolgee project
- BioEasy developers receive edit access
- developers create the translation key structure
- developers provide all English base strings
- Tolgee exports are then adapted for device implementation

Supported languages should match the current mobile applications unless changed later:

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

## 17. Quantitative Tests and Aflatoxin

Quantitative tests should be documented separately from the default qualitative cassette flow.

Rules:

- incubation is only allowed if backend test-type configuration includes temperature and incubation time
- quantitative/strip flows may require manual type selection
- Aflatoxin flow should follow the current desktop-reader approach
- Aflatoxin calibration curve must be loadable through ID chip or QR scanning

## 18. Export, Printing, and LIMS

Required capabilities:

- export grouped test records locally to CSV
- export grouped test records locally to Excel
- export to LIMS
- print individual test results
- print grouped test results
- print verification results

Export rules:

- CSV and Excel use the same column structure
- Excel is the same dataset as CSV in spreadsheet form
- do not depend on backend CSV or Excel export endpoints for device export
- use the provided `export.csv` sample as the reference structure for local export

CSV and Excel column structure:

- `Username`
- `Site`
- `Customer`
- `Updated By`
- `ImageAttached`
- `Route`
- `Operator`
- `Timedate`
- `Annotation`
- `Comment`
- `Overall Result`
- `Beta-lactams Result`
- `Beta-lactams Level`
- `Ceftiofur Result`
- `Ceftiofur Level`
- `Tetracyclines Result`
- `Tetracyclines Level`
- `Sulfonamides Result`
- `Sulfonamides Level`
- `Raw QR`
- `Test Type`
- `Batch Number`

LIMS export:

- LIMS integration/export is required in addition to CSV and Excel
- the exact LIMS format, transport, and implementation are left to BioEasy

## 19. Out of Scope

- negative control
- editing annotations on the device
- GPS-based location upload
- restoring historical records from backend back onto the device after reset

## 20. Implementation Appendix

### 20.1 Environments

Known environments:

- Development API base URL: `https://dev-milksafe.chr-hansen.com/api/v2`
- Development login token URL: `https://chrhmilksafedev.b2clogin.com/tfp/chrhmilksafedev.onmicrosoft.com/B2C_1A_ROPC_Auth/oauth2/v2.0/token`
- Development OAuth client ID: `db37c287-9f52-460d-a80e-4b15a5c2f6e8`
- Staging API base URL: `https://stg-milksafe.chr-hansen.com/api/v2`
- Staging login token URL: `https://chrhmilksafestaging.b2clogin.com/tfp/chrhmilksafestaging.onmicrosoft.com/B2C_1A_ROPC_Auth/oauth2/v2.0/token`
- Staging OAuth client ID: `3755b774-3b08-4dc2-9763-ab793fdfcb3b`
- Pre-production API base URL: `https://preprd-milksafe.chr-hansen.com/api/v2`
- Pre-production login token URL: `https://chrhmilksafepreprod.b2clogin.com/tfp/chrhmilksafepreprod.onmicrosoft.com/B2C_1A_ROPC_Auth/oauth2/v2.0/token`
- Pre-production OAuth client ID: `f8afc25d-b6bf-441e-92da-0e5aad497983`
- Production API base URL: `https://milksafe.chr-hansen.com/api/v2`
- Production login token URL: `https://chrhmilksafe.b2clogin.com/tfp/chrhmilksafe.onmicrosoft.com/B2C_1A_ROPC_Auth/oauth2/v2.0/token`
- Production OAuth client ID: `3d572dff-4080-4317-87e6-e5fbe6c4c6ca`

Environment note:

- pre-production is the main recommended testing environment for this project
- pre-production mirrors production closely
- production data is copied to pre-production on the first day of each month

### 20.2 Authentication and Login Sequence

Authentication is based on Azure B2C ROPC token exchange in the current mobile application implementation.

Login request:

- method: `POST`
- content type: `application/x-www-form-urlencoded`
- token path: `/token`
- required fields: `username`, `password`, `grant_type=password`, `scope=openid <client_id> offline_access`, `client_id`, `response_type=token id_token`

Token behavior in the current mobile applications:

- both `access_token` and `refresh_token` are stored locally
- authenticated API requests use `Authorization: Bearer <access_token>`
- if an authenticated request returns `401`, the client calls the same token endpoint with `grant_type=refresh_token`
- the new access token and refresh token are stored again after refresh
- practical behavior is a persistent session until logout, as long as token refresh continues to work
- exact server-side token lifetime is not documented in this specification

Login and configuration load sequence:

Recommended sequence:

1. Authenticate with MilkSafe Cloud username and password for the same user type used in DRC and MobileLabs.
2. Store the authenticated session/token.
3. Call `GET /api/webapi/v2/Users/me`.
4. Use the returned user/site context to call `GET /api/webapi/v2/TestTypes`.
5. Call `GET /api/webapi/v2/Sites/{id}` and apply the site's enabled test types.
6. Cache the full test type list and the effective site configuration locally.
7. After successful login, move cached anonymous grouped test records into the logged-in queue, clear their remote group IDs, and reupload them through `POST /api/webapi/v2/GroupedTestRecords`.

Important:

- do not reassign anonymous verification records after login
- if the device stays anonymous, keep using local test type configuration plus anonymous upload behavior
- when uploading tests or verification, always include the correct timezone offset from the device configuration

### 20.3 Grouped Test Upload Example

Use `POST /api/webapi/v2/GroupedTestRecords`.

Implementation notes:

- send one grouped payload per completed flow
- the `testRecords` list contains all test records from that finished flow
- do not upload test records one by one as they are produced
- if the first test is negative, upload the group immediately because the flow is complete
- if confirmation is required, keep the records locally and upload only after the user aborts or after the final result is known
- do not use deprecated `POST /api/webapi/v2/TestRecords` or `POST /api/webapi/v2/TestRecords/anonymous` for this device
- the example below follows the minimum payload pattern already used by the mobile applications
- `readerData` should be base64-encoded reader output
- `testDateOffset` must match the timezone configured on the device
- use `POST /api/webapi/v2/GroupedTestRecords/{id}/comment` for group comments when comments are added after the group already exists

```json
{
  "appVersion": "1.0.0",
  "testRecords": [
    {
      "testDate": "2026-04-09T10:26:00Z",
      "testDateOffset": "+02:00",
      "readerSerialNumber": "5PR-000123",
      "route": "SAMPLE-182",
      "result": "Positive",
      "readerData": "<base64-reader-data>",
      "testTypeId": 101,
      "operatorId": "OP-17",
      "substances": [
        {
          "substanceId": 1,
          "level": 0.42,
          "result": "Positive",
          "readerResultReference": "T1"
        }
      ],
      "annotation": "Original",
      "manufacturingDate": "2026-01-15",
      "batchNumber": "250428E002",
      "rawQR": "05250428E002002627",
      "cassetteId": "002627",
      "appVersion": "1.0.0"
    }
  ]
}
```

### 20.4 Anonymous Grouped Upload Rules

Use `POST /api/webapi/v2/GroupedTestRecords/anonymous`.

Rules:

- keep the full local values for route/sample ID, operator ID, and user-related linkage so the group can later be reassigned after login
- do not upload route/sample ID, operator identity, or user identity in plain form
- upload `readerSerialNumber` in plain form
- upload empty location
- keep the local history state as `not synced`
- when the user later logs in, reupload the cached groups through the normal grouped endpoint

Reference note:

- the current mobile applications use the anonymous grouped endpoint with anonymous field transformation
- in the mobile applications, `route` is removed and `operatorId` is replaced with a device identifier
- the 5-Port Reader should follow the same anonymous-endpoint pattern, with the product rules in this document taking precedence

### 20.5 Verification Upload Example

Use `POST /api/webapi/v2/DeviceHealth`.

Implementation notes:

- send one payload per verification run
- `readerData` should be base64-encoded reader output
- `substances[*].intensity` is part of the verification payload
- `testDateOffset` must match the timezone configured on the device
- pass/fail is determined by the verification rules in this document, not only by HTTP success

```json
{
  "readerSerialNumber": "5PR-000123",
  "testDate": "2026-04-09T10:26:00Z",
  "testDateOffset": "+02:00",
  "comments": "Scheduled verification",
  "result": "Negative",
  "readerData": "<base64-reader-data>",
  "testTypeId": 0,
  "operatorId": "OP-17",
  "substances": [
    {
      "substanceId": 1,
      "level": 1.0,
      "result": "Negative",
      "readerResultReference": "V1",
      "intensity": 523456
    }
  ],
  "deviceType": "Portable",
  "temperature": {
    "measuredTemperature": 40.1,
    "deviceTemperature": 40.0
  },
  "readerSoftwareVersion": "1.0.0",
  "appVersion": "1.0.0"
}
```

### 20.6 Planned Anonymous Verification Endpoint

Target behavior:

- use `POST /api/webapi/v2/DeviceHealth/anonymous`
- use the same payload structure as `POST /api/webapi/v2/DeviceHealth`
- upload anonymous verification records there
- do not reassign anonymous verification records after login

Dependency:

- this endpoint is a planned backend extension and is not yet present in the current Swagger

### 20.7 ReaderData

`readerData` should be documented and handled as raw reader output.

Current mobile application behavior:

- the reader returns raw measurement bytes as `[UInt8]`
- the application stores those exact raw bytes in `readerData`
- upload serializes `readerData` as a base64 string

Qualitative flow details from the current parser:

- the parser strips the first 2 bytes and the last 4 bytes from the raw response
- the remaining payload is processed in 7-byte sets
- each set contains position, height, and area values
- the app uses those parsed values to calculate ratios and results, but the uploaded `readerData` remains the original raw bytes

Quantitative flow details from the current parser:

- the app still stores the original raw bytes in `readerData`
- it also interprets the response as UTF-8 text split by `#` for app-side parsing of quantitative result data
- even in this case, the upload field should still contain the original raw bytes encoded to base64

Implementation rule:

- treat `readerData` as opaque raw device output for backend traceability and future reprocessing
- do not replace it with parsed ratios, JSON objects, or debug strings

### 20.8 Local Export Notes

- CSV and Excel are local device exports, not backend report downloads
- use the exact column order and header names from `export.csv`
- preserve the header spelling from the reference file, including `Timedate`
- Excel uses the same dataset and column structure as CSV
- LIMS integration/export is also required, but its exact implementation is intentionally left to BioEasy
