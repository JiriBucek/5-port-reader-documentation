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

### 14.2 Payload Strategy

Use the minimum payload that matches the required device behavior.

For grouped test upload, focus on:

- `customerId`
- `siteId`
- `comments`
- `appVersion`
- `testRecords`

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
- LINS on/off
- Sound on/off

Date/time:

- the user sets date, time, and timezone
- upload timestamps with timezone data

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

## 18. Export, Printing, and LINS

Required capabilities:

- export grouped test records locally to CSV
- export grouped test records locally to Excel
- export to LINS
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

LINS export:

- LINS export is required in addition to CSV and Excel
- the exact LINS export structure is not defined in this document yet and should be added when provided

## 19. Out of Scope

- negative control
- editing annotations on the device
- GPS-based location upload
- restoring historical records from backend back onto the device after reset
