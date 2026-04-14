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

The same user type that is used to log in to DRC and the mobile applications should also log in to this device.

That user type is `Operator`.

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
- do not display a backend group ID for anonymous grouped test records
- only display a group ID for logged-in synced grouped test records

Anonymous reassignment behavior:

- if the user later logs in on the same device, previously anonymous grouped test records should be assigned to that user and uploaded normally to their site
- this mirrors the current mobile applications behavior

Anonymous verification behavior:

- anonymous verification records should also upload
- they are not reassigned after login!

## 6. Test Types and Configuration

### 6.1 Source of Test Types

Test types must be bundled into the device software as a JSON file using the exact JSON response from the backend test-types endpoint that is current at the time of development.

Reason:

- some devices may never connect to the internet
- the bundled test types must preserve the real backend IDs

### 6.2 Offline and Refresh Behavior

Recommended behavior:

- bundle the full test-types JSON in the software
- on every device restart (or any other regular event), attempt to download fresh test types
- if logged in, also load the site-specific test type configuration
- when fresh test types are successfully loaded, replace the bundled/cached set with the current backend version

Anonymous mode:

- load all test types
- allow local enable/disable of these test types

Logged-in mode:

- load all test types
- apply site-enabled test type configuration from cloud. Filter all the test types only for those that are allowed for the particular site
- the device should not let the user override site enablement locally. The user can only view the site's test type configuration in the Settings / Test Types

### 6.3 QR Setting

The 5-Port Reader uses a global QR scanning setting.

Rules:

- QR scanning can be turned on or off in Settings
- when QR scanning is enabled and the QR is read correctly, the test type is prefilled in the configuration modal
- when the test type is prefilled from QR, the user must not be able to change it manually
- if QR scanning is disabled, the user selects the test type manually
- if the scanned test type is not enabled for the site or local anonymous configuration, the device must show a warning and block the run

### 6.4 Configuration field behavior

- when `Sample ID` is present in the configuration flow, it is the first data-entry field
- if `Operator ID` is also present, completing `Sample ID` must move focus immediately to `Operator ID`
- this auto-advance must happen both after scanner input and after manual keyboard entry confirmed with `Enter / Done`
- the user should not need to manually select `Operator ID` after completing `Sample ID`

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

- the device temperature is set manually by the user at device level in the Settings
- when the cassette is identified, the device must validate the current heater temperature against the selected test type configuration
- if the device temperature is outside the allowed range for that test type, block the test and show a warning

### 7.3 Repeat Incubation Rule

For normal tests only:

- if the first test result is positive and that test was incubated, repeat incubation for 2 more minutes
- judge the outcome based on the repeated reading

This rule does not apply to controls.

### 7.4 Cassette Handling

- insertion can be detected reliably
- cassette removal cannot be relied on and is not detected by the reader
- the user can interrupt incubation
- the device does not eject the cassette automatically. This is to stop contamination of the surroundings with a potentially positive milk

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
- if the result is not positive, show the failed state with a cross. This is opposite to the standard tests where a cross icon is displayed for a positive test. The Positive control test is expected to be positive, not negative.
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
- slot choice can be optionally saved locally in the reader
- slot choice has no backend meaning, there is not backend endpoint key to upload the slot number to

### 11.3 Pass Criteria

Verification includes:

- ratio measurement
- measured temperature vs device temperature comparison
- light-intensity - as one number

Pass rules:

- every measured ratio must be within `0.9` to `1.1`
- measured temperature must be within `±2 C`
- light intensity must be greater than `500000` units

### 11.4 Verification Upload

Verification uses the device-health endpoint, not grouped test endpoint.

- `DeviceHealth` endpoint is authenticated - use this one for the logged in users
- Use `DeviceHealth/anonymous` for anonymous / logged out users. This endpoint does not require authentication


### 11.5 Verification History

- verification history is separate from test/control history
- it is accessible from Settings
- verification records are cached locally
- verification records are uploaded to backend
- after factory reset, local verification history is lost
- uploaded verification records remain in backend
- there is no way to fetch the verification history from the backend. You only show the locally stored verifications and upload them to the backend but you do not download the history of verifications

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
- grouped test history may optionally be downloaded back from cloud after a device reset, but only grouped test records created on this particular device may be shown
- do not display test records recorded on a different device that may have used the same operator credentials
- anonymous grouped test records should show `not synced` and should not show a group ID
- show group ID only for logged-in grouped test records that have synced successfully

### 12.2 Verification History

- separate from test/control history
- accessed from Settings

### 12.3 Comments

- comments are group-level
- the user can comment on the whole group/history item
- comment presence should be indicated in grouped history
- annotation editing on the device is out of scope
- annotation editing can only be performed in the web console

### 12.4 Detail Screens

Test/control detail should include:

- result
- test type
- date/time in the device timezone
- user
- operator ID
- route / sample ID
- upload status
- measured substances and ratio values
- port number, only cached locally on the device and not uploaded to backend
- local light-intensity chart similar to DRC

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
- recommended retry cadence for unsynced grouped test records is every 60 seconds
- display a group ID only after the grouped record has synced successfully
- if a logged-in grouped record is still unsynced, do not display a group ID yet

### 13.2 Anonymous Grouped Test Records

- store full local values for local behavior and later reassignment
- upload through anonymous grouped-test endpoint in obfuscated form
- show as `not synced` in device history
- keep reader serial number un-obfuscated
- do not display a backend-assigned group ID for anonymous grouped test records, even if one exists in backend
- for anonymous grouped test records, the UI state is `not synced`
- only logged-in synced grouped test records should display a group ID

When the user later logs in:

- assign cached anonymous grouped test records to that user
- upload them normally to the user's site

### 13.3 Verification Records

- upload through device-health flow
- anonymous verification uploads use the `DeviceHealth/anonymous` endpoint
- anonymous verification records are not reassigned after login

### 13.4 Factory Reset

- local test/control history is removed
- local verification history is removed
- uploaded grouped test records remain in backend
- uploaded verification records remain in backend
- grouped test history may optionally be downloaded back after reset, but only records created on this device may be shown
- do not restore grouped test records from other devices that used the same operator credentials
- verification history is not restored back onto the device from backend

## 14. API Integration

### 14.1 Endpoint Set

Main endpoints expected for this device:

- `GET /api/v2/users/me`
- `GET /api/v2/TestTypes`
- `GET /api/v2/Sites/{id}`
- `GET /api/v2/Firmware/{type}/latest`
- `POST /api/v2/GroupedTestRecords`
- `POST /api/v2/GroupedTestRecords/anonymous`
- `POST /api/v2/DeviceHealth`
- `POST /api/v2/DeviceHealth/anonymous`
- `POST /api/v2/GroupedTestRecords/{id}/comment`

Path rule:

- for the live MilkSafe app hosts, use the `/api/v2` path prefix
- do not use `/api/webapi/v2` on those hosts
- the current Swagger path naming does not match the live app-host path prefix
- for `GET /api/v2/Firmware/{type}/latest`, use reader-type values such as `PortableReaderConnect` and `DesktopReader`
- do not use device-type values such as `Portable` or `Desktop` in that firmware route

Do not use:

- deprecated single-record upload endpoints such as `POST /api/v2/TestRecords`
- deprecated anonymous single-record upload endpoints such as `POST /api/v2/TestRecords/anonymous`
- this device must upload normal test results through grouped test upload only

### 14.2 Annotation

Set `annotation` on each test record included in grouped test upload.

Use these annotation values in 5-Port Reader grouped test records:

- `Original`
- `Confirmation`
- `SecondConfirmation`
- `PositiveControl`
- `AnimalControl`

Normal flow:

- the first test record in a flow uses `Original`
- the second test record in the same flow uses `Confirmation`
- the third test record in the same flow uses `SecondConfirmation`
- if the user stops after the first positive test, upload the single existing test record as `Original`
- if the user stops after the second test, upload the two existing test records as `Original` and `Confirmation`

Control flow:

- positive control uploads use `PositiveControl`
- animal control uploads use `AnimalControl`
- a failed positive control still uses `PositiveControl`
- a failed animal control still uses `AnimalControl`

Do not use these annotation values in the 5-Port Reader grouped test upload flow:

- `Rejected`
- `Deleted`
- `Verification`

Verification rule:

- do not upload verification through grouped test records
- do not send grouped test records with `Verification` annotation
- verification-through-test-record annotation is deprecated for this device
- upload verification through the device-health endpoints only

### 14.3 Payload Strategy

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
- `testDateOffset` if sent
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

Result value rule:

- newly created 5-Port Reader test records should only use `Positive` and `Negative`
- do not create new records with `WeakPositive`
- `WeakPositive` is a deprecated backend value and should only be supported in decoding logic for old records if such records are ever fetched from cloud
- if a fetched backend record contains `WeakPositive`, decode and handle it as `Positive`
- device-local history only shows records created on this device, so normal 5-Port Reader history should not create or show new `WeakPositive` records

Do not upload `readerData`.

Date/time rule:

- `testDate` is the test timestamp in ISO 8601 date-time format and should include the timezone offset in the timestamp itself, for example `2026-04-09T10:26:00+02:00`
- `testDateOffset` is the local timezone offset configured on the device at the moment of testing, for example `+02:00` or `-05:00`
- the current mobile application grouped upload format sends the offset inside `testDate`
- the current API schemas also include a separate optional `testDateOffset` field
- if the device sends `testDateOffset`, it must match the offset already embedded in `testDate`
- do not send `testDate` as `Z` together with a different local `testDateOffset`
- this is required so the cloud can display the test time correctly in the timezone in which the test was taken

For verification upload:

- `readerSerialNumber`
- `testDate`
- `testDateOffset` if sent
- `comments`
- `result`
- `operatorId`
- `substances`
- `deviceType`
- `temperature`
- `readerSoftwareVersion`
- `appVersion`

### 14.4 Mobile Application Reference

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
- Sound on/off

Other settings/toggles:

- Printer on/off
- Comments on/off
- Route / Sample ID on/off
- Operator ID on/off
- Incubator on/off
- LIMS on/off. LIMS configuration is not yet specified and is left to BioEasy.

Date/time:

- the user sets date, time, and timezone
- display date and time in the device using the user-selected local timezone
- always upload timestamps with timezone data
- send `testDate` with the correct local timezone offset embedded in the timestamp, for example `2026-04-09T10:26:00+02:00`
- if `testDateOffset` is also sent, it must match the offset embedded in `testDate`
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
- machine translation is used to translate the English base strings to all supported languages
- translations are reviewed by human reviewers for every language
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
- for quantitative test types, fetch and cache `measurementMethod` and `quantitativeRange` from the test type configuration
- `quantitativeRange` contains `measurableRangeMin`, `measurableRangeMax`, `negativeRangeMin`, and `negativeRangeMax`
- do not hardcode quantitative limits in the device
- use the fetched measurable range to format the displayed quantitative value
- if the measured value is within the inclusive measurable range, display the formatted numeric value
- if the measured value is below `measurableRangeMin`, display `< {measurableRangeMin}`
- if the measured value is above `measurableRangeMax`, display `> {measurableRangeMax}`
- examples: `< 15`, `> 150`
- use normal device locale number formatting for in-range values
- the value-formatting rule above applies to quantitative result display in test detail, history detail, print output, and any other screen where the numeric level is shown
- the positive/negative result label still comes from the test result itself; the measurable range only changes how the numeric level is displayed
- fetch and store `negativeRangeMin` and `negativeRangeMax` together with the measurable range as part of the quantitative test type configuration
- if a quantitative test type is missing `quantitativeRange`, fall back to displaying the formatted numeric value without the `<` or `>` threshold substitution
- keep the last 5 loaded calibration curves on the device and automatically remove older ones when new ones are loaded
- when a quantitative test type is selected, the configuration must include calibration-curve selection
- calibration-curve selection is only visible for quantitative test types

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
- this refresh overwrites pre-production data changes with production data

### 20.2 Authentication and Login Sequence

Authentication is based on Azure B2C ROPC token exchange in the current mobile application implementation.

Login request:

- method: `POST`
- content type: `application/x-www-form-urlencoded`
- token path: `/token`
- required fields: `username`, `password`, `grant_type=password`, `scope=openid <client_id> offline_access`, `client_id`, `response_type=token id_token`

Local device token behavior:

- both `access_token` and `refresh_token` are stored locally
- authenticated API requests use `Authorization: Bearer <access_token>`
- if an authenticated request returns `401`, the client calls the same token endpoint with `grant_type=refresh_token`
- the new access token and refresh token are stored again after refresh
- practical behavior is a persistent session until logout, as long as token refresh continues to work

Login and configuration load sequence:

Recommended sequence:

1. Authenticate with MilkSafe Cloud username and password for the same user type used in DRC and the mobile applications.
   That user type is `Operator`.
2. Store the authenticated session/token.
3. Call `GET /api/v2/users/me`.
4. Call `GET /api/v2/TestTypes` to obtain the full list of test types.
5. Use the user/site context from `GET /api/v2/users/me` to call `GET /api/v2/Sites/{id}` and obtain the site's test type configuration.
6. Cache the full test type list and the effective site configuration locally, including `measurementMethod` and `quantitativeRange` for quantitative test types.
7. After successful login, move cached anonymous grouped test records into the logged-in queue, clear their remote group IDs, and reupload them through `POST /api/v2/GroupedTestRecords`.

Important:

- do not reassign anonymous verification records after login
- if the device stays anonymous, keep using local test type configuration plus anonymous upload behavior
- when uploading tests or verification, always include timezone data that matches the device configuration

### 20.3 Grouped Test Upload Example

Use `POST /api/v2/GroupedTestRecords`.

Implementation notes:

- send one grouped payload per completed flow
- the `testRecords` list contains all test records from that finished flow
- do not upload test records one by one as they are produced
- if the first test is negative, upload the group immediately because the flow is complete
- if confirmation is required, keep the records locally and upload only after the user aborts or after the final result is known
- do not use deprecated `POST /api/v2/TestRecords` or `POST /api/v2/TestRecords/anonymous` for this device
- the example below follows the minimum payload pattern already used by the mobile applications
- `testDate` should include the local timezone offset in the timestamp, for example `2026-04-09T10:26:00+02:00`
- if `testDateOffset` is sent, it must match the offset embedded in `testDate`
- use `POST /api/v2/GroupedTestRecords/{id}/comment` for group comments when comments are added after the group already exists
- the comment request body is a raw JSON string, for example `"Operator note"`

```json
{
  "appVersion": "1.0.0",
  "testRecords": [
    {
      "testDate": "2026-04-09T10:26:00+02:00",
      "readerSerialNumber": "PR22050201",
      "route": "SAMPLE-182",
      "result": "Positive",
      "testTypeId": 3,
      "operatorId": "OP-17",
      "substances": [
        {
          "substanceId": 3,
          "level": 0.42,
          "result": "Positive",
          "readerResultReference": "T1"
        },
        {
          "substanceId": 4,
          "level": 0.18,
          "result": "Negative",
          "readerResultReference": "T1"
        },
        {
          "substanceId": 5,
          "level": 0.15,
          "result": "Negative",
          "readerResultReference": "T1"
        },
        {
          "substanceId": 7,
          "level": 0.10,
          "result": "Negative",
          "readerResultReference": "T1"
        }
      ],
      "annotation": "Original",
      "appVersion": "1.0.0"
    },
    {
      "testDate": "2026-04-09T10:31:00+02:00",
      "readerSerialNumber": "PR22050201",
      "route": "SAMPLE-182",
      "result": "Positive",
      "testTypeId": 3,
      "operatorId": "OP-17",
      "substances": [
        {
          "substanceId": 3,
          "level": 0.43,
          "result": "Positive",
          "readerResultReference": "T2"
        },
        {
          "substanceId": 4,
          "level": 0.16,
          "result": "Negative",
          "readerResultReference": "T2"
        },
        {
          "substanceId": 5,
          "level": 0.14,
          "result": "Negative",
          "readerResultReference": "T2"
        },
        {
          "substanceId": 7,
          "level": 0.11,
          "result": "Negative",
          "readerResultReference": "T2"
        }
      ],
      "annotation": "Confirmation",
      "appVersion": "1.0.0"
    }
  ]
}
```

### 20.4 Anonymous Grouped Upload Rules

Use `POST /api/v2/GroupedTestRecords/anonymous`.

Rules:

- keep the full local values for route/sample ID, operator ID, and user-related linkage so the group can later be reassigned after login
- in the anonymous grouped upload payload, omit `customerId`, `siteId`, `route`, and `operatorId`
- do not send user-identity fields in the anonymous grouped upload payload
- upload `readerSerialNumber` in plain form
- upload empty location
- keep the local history state as `not synced`
- do not display any backend-assigned group ID for anonymous grouped test records
- if backend silently assigns a group ID on anonymous upload, ignore it for device UI
- when the user later logs in, reupload the cached groups through the normal grouped endpoint

### 20.5 Verification Upload Example

Use `POST /api/v2/DeviceHealth`.

Implementation notes:

- send one payload per verification run
- `substances[*].intensity` is part of the verification payload
- `testDate` should include the local timezone offset in the timestamp, for example `2026-04-09T10:26:00+02:00`
- if `testDateOffset` is sent, it must match the offset embedded in `testDate`
- `appVersion` must not exceed 20 characters
- `readerSoftwareVersion` must not exceed 20 characters
- pass/fail is determined by the verification rules in this document, not only by HTTP success

```json
{
  "readerSerialNumber": "5PR-000123",
  "testDate": "2026-04-09T10:26:00+02:00",
  "comments": "Scheduled verification",
  "result": "Negative",
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

### 20.6 Anonymous Verification Endpoint

Use `POST /api/v2/DeviceHealth/anonymous`.

- use the same payload structure as `POST /api/v2/DeviceHealth`
- upload anonymous verification records there
- do not reassign anonymous verification records after login

Important:

- this endpoint is live on the app host even though it is not present in the current Swagger

### 20.7 Local Export Notes

- CSV and Excel are local device exports, not backend report downloads
- use the exact column order and header names from `export.csv`
- preserve the header spelling from the reference file, including `Timedate`
- Excel uses the same dataset and column structure as CSV
- LIMS integration/export is also required, but its exact implementation is intentionally left to BioEasy
