# Registration and Login — Manual API Test Cases

| Project detail | Value |
| --- | --- |
| Tester | Bhoomika K. |
| Execution date | 24 November 2026 |
| Tool | Postman |
| APIs | DummyJSON and Simple Grocery Store |
| Scope | Shopper registration, login, token refresh, grocery-client registration, and input validation |

## 1. Request setup

Select **Storefront QA • public practice** in Postman. For requests with a JSON body, select **Body → raw → JSON** and use `Content-Type: application/json`.

| Variable | Value or purpose |
| --- | --- |
| `{{base_url}}` | `https://dummyjson.com` |
| `{{grocery_url}}` | `https://simple-grocery-store-api.click` |
| `{{demo_username}}` | Published DummyJSON practice username |
| `{{demo_password}}` | Password for the practice username |
| `{{refresh_token}}` | Refresh token saved after successful shopper login |
| `{{run_email}}` | Fresh synthetic email for grocery-client registration. Reuse the same value for the duplicate-email test. |

### Postman request reference

| Request | Method and URL | Baseline request body |
| --- | --- | --- |
| Authentication → AU01 — Register a sample shopper | `POST {{base_url}}/users/add` | `{"firstName":"Mira","lastName":"Sample","email":"mira@example.test"}` |
| Authentication → AU02 — Log in with the published practice account | `POST {{base_url}}/auth/login` | `{"username":"{{demo_username}}","password":"{{demo_password}}","expiresInMins":30}` |
| Authentication → AU03 — Refresh the shopper session | `POST {{base_url}}/auth/refresh` | `{"refreshToken":"{{refresh_token}}","expiresInMins":30}` |
| Authentication → AU04 — Register the grocery checkout client | `POST {{grocery_url}}/api-clients` | `{"clientName":"Portfolio QA","clientEmail":"{{run_email}}"}` |

## 2. Positive cases

| ID | Postman request | Input or precondition | Expected result | Actual result | Status |
| --- | --- | --- | --- | --- | --- |
| AU01 | Authentication → AU01 · `POST /users/add` | Submit the sample shopper from the request reference. | HTTP 201. Response contains a numeric ID and the submitted name and email. | `201 Created`. ID `209`, name `Mira Sample`, and email `mira@example.test` matched the request. | Pass |
| AU02 | Authentication → AU02 · `POST /auth/login` | Use the published credentials for `emilys`. | HTTP 200. Response identifies the shopper and includes nonempty access and refresh tokens. | `200 OK`. ID `1`, username `emilys`, and both tokens were returned. | Pass |
| AU03 | Authentication → AU03 · `POST /auth/refresh` | Use the refresh token from AU02 and numeric expiry `30`. | HTTP 200 with nonempty access and refresh tokens. | `200 OK`. Both tokens were returned. Postman showed 4/4 checks passed. | Pass for token issuance |
| AU04 | Authentication → AU04 · `POST /api-clients` | Use a valid client name and a fresh synthetic email. | HTTP 201 with a nonempty `accessToken`. | `201 Created`. An access token was returned. | Pass |

## 3. Negative cases

Each row identifies the saved request to use in the **Negative Tests** folder. During manual execution, some inputs were also tested by editing the equivalent request in the Authentication folder.

| ID | Postman request | Request body or input change | Expected result | Actual result | Status |
| --- | --- | --- | --- | --- | --- |
| NG01 | Negative Tests → NG01 · `POST {{grocery_url}}/api-clients` | Send `{"clientName":"Portfolio QA","clientEmail":"{{run_email}}"}` using the email successfully registered in AU04. | HTTP 409 with a duplicate-registration error. | `409 Conflict`. “API client already registered. Try a different email.” | Pass |
| NG02 | Negative Tests → NG02 · `POST {{grocery_url}}/api-clients` | Send `{"clientName":"Portfolio QA","clientEmail":"bad-email"}`. | HTTP 400 with an email validation error. | `400 Bad Request`. “Invalid or missing client email.” | Pass |
| NG03 | Negative Tests → NG03 · `POST {{grocery_url}}/api-clients` | Send only `{"clientEmail":"qa-ng03-recheck-1741@example.com"}`. Omit `clientName`. | HTTP 400 with a client-name validation error and no token. | `400 Bad Request`. “Invalid or missing client name.” No token was returned. | Pass |
| NG04 | Negative Tests → NG04 · `POST {{base_url}}/auth/login` | Keep the valid username and send an incorrect password. | HTTP 400 with a credentials error and no tokens. | Login was rejected. Captured response showed `400 Bad Request` and “Invalid credentials”. Valid credentials succeeded as a control. | Pass |
| NG05 | Negative Tests → NG05 · `POST {{base_url}}/auth/login` | Send only `{"username":"emilys"}`. Omit the entire `password` field. | HTTP 400 with a required-field error and no tokens. | “Username and password required” was reported. The status and exact request body for that response still need to be confirmed together. | Partial |

## 4. Exploratory cases

| ID | Postman request | Input change | Expected result | Actual result | Status |
| --- | --- | --- | --- | --- | --- |
| EXP01 | Authentication → AU01 · `POST {{base_url}}/users/add` | Send a blank request body. Repeat with **Body → none**. | Portfolio rule: registration requires an email and should return a 4xx validation error when the body is empty. | First run returned `200 OK`, ID `209`, and blank names and email. Repeat with **Body → none** returned `201 Created`. | Fail against portfolio rule; simulator observation |
| EXP02 | Authentication → AU02 · `POST {{base_url}}/auth/login` | Send an incorrect username with the password from the valid practice account. | Login is rejected. | “Invalid credentials” was reported. Individual HTTP status was not recorded. | Rejection observed |
| EXP03 | Authentication → AU02 · `POST {{base_url}}/auth/login` | Send an incorrect username and an incorrect password. | Login is rejected. | “Invalid credentials” was reported. Individual HTTP status was not recorded. | Rejection observed |
| EXP04 | Authentication → AU03 · `POST {{base_url}}/auth/refresh` | Replace the valid `refreshToken` with an invalid value. | Refresh request is rejected. | `403` with “Invalid refresh token”. | Rejection verified |
| EXP05 | Authentication → AU03 · `POST {{base_url}}/auth/refresh` | Send a valid `refreshToken` and omit `expiresInMins`. | Tokens are issued using the default expiry. | Token issuance succeeded. The actual token lifetime was not measured. | Issuance verified |
| EXP06 | Authentication → AU03 · `POST {{base_url}}/auth/refresh` | Send a valid refresh token and `"expiresInMins":"43201"` as a JSON string. | Portfolio expectation: invalid input receives a 4xx validation response. | `500 Internal Server Error`. “Maximum access token expire time can be 43200 minutes”. | Fail against expected status |
| EXP07 | Authentication → AU02 · `POST {{base_url}}/auth/login` | Send valid credentials and `"expiresInMins":"43201"` as a JSON string. | Portfolio expectation: invalid input receives a 4xx validation response. | `500 Internal Server Error` with the same maximum-expiry message. | Fail against expected status |

**Registration ID observation:** Both the valid registration and the first blank-body response returned ID `209`. DummyJSON simulates user creation. The evidence establishes that the response value repeated; it does not establish that two stored accounts shared an ID.

## 5. Numeric expiry boundary cases

The API error message named `43200` minutes as the maximum. These cases check the maximum and one minute above it. Enter each value as a JSON **number**, without quotation marks. Keep the other fields unchanged.

| ID | Postman request | Request body | Expected result | Actual result | Status |
| --- | --- | --- | --- | --- | --- |
| BND01 | Authentication → AU02 · `POST {{base_url}}/auth/login` | `{"username":"{{demo_username}}","password":"{{demo_password}}","expiresInMins":43200}` | HTTP 200 with access and refresh tokens at the maximum. | A `200` at `43200` was reported. The endpoint and JSON type for that run were not captured. | Pending confirmation |
| BND02 | Authentication → AU02 · `POST {{base_url}}/auth/login` | `{"username":"{{demo_username}}","password":"{{demo_password}}","expiresInMins":43201}` | A 4xx validation response and no tokens. | The captured login failure used the quoted string `"43201"`. The numeric result is unconfirmed. | Pending confirmation |
| BND03 | Authentication → AU03 · `POST {{base_url}}/auth/refresh` | `{"refreshToken":"{{refresh_token}}","expiresInMins":43200}` | HTTP 200 with access and refresh tokens at the maximum. | No response tied to this exact refresh request was recorded. | Pending confirmation |
| BND04 | Authentication → AU03 · `POST {{base_url}}/auth/refresh` | `{"refreshToken":"{{refresh_token}}","expiresInMins":43201}` | A 4xx validation response and no tokens. | The captured refresh failure used the quoted string `"43201"`. The numeric result is unconfirmed. | Pending confirmation |

## 6. Proposed shopper-registration contract

These cases describe requirements for a storefront that stores shopper accounts. They require an implementing sandbox. No Postman requests are configured for these proposed endpoints.

| ID | Proposed request | Input or precondition | Expected result | Status |
| --- | --- | --- | --- | --- |
| CX01 | `POST /v1/shoppers` | Submit a new synthetic email and a valid sample password. | HTTP 201. A unique shopper ID and normalized email are returned. The account can be used for login. | Blocked — sandbox required |
| CX02 | `POST /v1/sessions` | Use the credentials registered in CX01. | HTTP 200 with an access token. The signed-in profile matches the shopper ID created by CX01. | Blocked — sandbox required |
| CX03 | `POST /v1/shoppers` | Resubmit the email already registered in CX01. | HTTP 409 with `EMAIL_IN_USE`. Only the original account exists. | Blocked — sandbox required |
| CX14 | `POST /v1/shoppers` | Supply `mira.example.test` as the email, with an otherwise valid request. | HTTP 422 with a field error for email. No shopper ID is issued. | Blocked — sandbox required |
| CX15 | `POST /v1/shoppers` | Send a boolean for `email` and a synthetic password. Inspect the error response. | HTTP 422 with a field-level error. The response omits password values, tokens, and internal diagnostics. | Blocked — sandbox required |
