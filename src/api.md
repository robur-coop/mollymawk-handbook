Mollymawk's API system allows mollymawk to be used via programmatic access, such as in CI/CD pipelines.

## Base URL

If you deploy mollymawk, your base url will be what you use below. For this documentation, we will use an example url:

```http
https://mollymawk.test
```

All paths in this document are relative to this base.

---

## Authorization

Most API endpoints require **Bearer token authentication**.
Include an `Authorization` header of the form:

```http
Authorization: Bearer <ACCESS_TOKEN>
```

The `<ACCESS_TOKEN>` is an api key for the user. See [Generate a new API Key](./tokens.md#creating-a-new-token)

---

## Request Headers

All api endpoints expect:

```http
Content-Type: application/json or multipart/form-data
Accept: application/json
Authorization: Bearer <ACCESS_TOKEN>
```

---

## Response Envelope

All responses share a common envelope:

```json
{
  "status": 200,
  "title": "Success",
  "success": true,
  "data": ...
}
```

- `status` — HTTP-like status code as an integer (e.g. 200, 400, 404, 500)
- `title` — short string version describing the status
- `success` — boolean indicating overall success
- `data` — endpoint-specific payload (object or string, depending on call)

---

## Endpoints

### Authentication

Authenticates a user using their email and password. Upon success, it returns the user's profile data. This endpoint can be used to retrieve an api token.

**Endpoint:**

```http
POST /api/login
```

**Content-Type:**
`application/json`

**Request Parameters:**

The request body must be a JSON object containing the following fields:

| Field      | Type   | Required | Description                                              |
| :--------- | :----- | :------- | :------------------------------------------------------- |
| `email`    | string | Yes      | The email address registered to the account.             |
| `password` | string | Yes      | The user's password. Must be at least 8 characters long. |

**Example:**

```bash
curl -X POST https://mollymawk.test/api/login \
  -H "Content-Type: application/json" \
  -d '{
    "email": "user@example.com",
    "password": "password123"
  }'
```

**Success Response (example):**

```json
{
    "status": 200,
    "title": "200",
    "success": true,
    "data": {
        "name": "user",
        "email": "user@example.com",
        "email_verified": null,
        "password": "eVHL6CIc5phEoElSOACRk1Kztd3CFZD68vzHAt9StFI=",
        "uuid": "1f42ba42-5458-479b-8095-8450afde8d6b",
        "tokens": [
            {
                "token_type": "Bearer",
                "value": "6bc27967-30cd-4090-af8f-3997c47c25e1",
                "expires_in": 2419200,
                "created_at": "2025-12-05 04:14:38-00:00",
                "last_access": "2025-12-05 04:14:38-00:00",
                "name": "fobud",
                "usage_count": 13
            },
            ...
        ],
        "cookies": [
            {
                "name": "molly_session",
                "created_at": "2025-12-08 16:53:03-00:00",
                "value": "YTI2Y2JhNmUtMmI5NS00OTQ3LTg5ZTAtY2E1MDJmOThjYjhj",
                "expires_in": 604800,
                "uuid": "1f42ba42-5458-479b-8095-8450afde8d6b",
                "last_access": "2025-12-08 16:53:03-00:00",
                "user_agent": "Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/141.0.0.0 Safari/537.36"
            },
            ...
        ],
        "created_at": "2025-12-07 23:12:26-00:00",
        "updated_at": "2025-12-08 16:23:36-00:00",
        "email_verification_uuid": null,
        "active": true,
        "super_user": true,
        "unikernel_updates": [
            {
                "name": "mello",
                "job": "hello",
                "uuid": "6cc1a49d-eb1e-4f46-8471-6452206d1475",
                "config": {
                    "typ": "solo5",
                    "compressed": false,
                    "fail_behaviour": {
                        "restart": null,
                        "exit_code": [],
                        "all_exit_codes": false
                    },
                    "cpuid": 0,
                    "memory": 0,
                    "block_devices": [],
                    "network_interfaces": [],
                    "arguments": []
                },
                "timestamp": "2025-12-07 23:08:24-00:00"
            }
        ]
    }
}
```

**Error Responses:**

If the request fails, the server returns a JSON string containing the error message.

- **Status Code:** `400 Bad Request`

  - `"All fields must be filled."` — One or more required fields are missing or empty.
  - `"Invalid email address."` — The email format is incorrect.
  - `"Password must be at least 8 characters long."` — The password provided is too short.
  - `"This account does not exist."` — The email provided is not found in the database.
  - `"This account is not active"` — The user account has been deactivated by an administrator.
  - `"Invalid email or password."` — The password provided does not match the stored hash.

- **Status Code:** `500 Internal Server Error`

  - May occur if there is a failure updating the user session in the storage backend.

### Unikernel Operations

#### List unikernels

Retrieves unikernel information for a user accross all albatross servers.

**Endpoint:**

```http
GET /api/unikernels
```

**Example:**

```bash
curl -X GET https://mollymawk.test/api/unikernels \
  -H "Authorization: Bearer <access_token>"
```

**Success Response (example):**
Returns a JSON object containing `data` (a list of lists, where each inner list contains unikernel objects from a specific Albatross instance) and `errors` (a list of errors from instances that could not be queried).

```json
{
  "data": [
    [
      {
        "name": "hello",
        "fail_behaviour": {
          "restart": null,
          "exit_code": [],
          "all_exit_codes": true
        },
        "cpuid": 10,
        "memory": 32,
        "block_devices": [],
        "network_interfaces": [],
        "arguments": []
      }
    ]
  ],
  "errors": []
}
```

_Note: If an Albatross instance fails to respond, it will appear in the `errors` list rather than causing a 500 error for the whole request._

**Failure Messages:**

- **Status Code:** `400 Bad Request`

  - `"Token value is not valid <token>"` — The provided API token was not found or has expired.
  - `"User account is deactivated"` — The user is not active.

- **Status Code:** `500 Internal Server Error`
  - May occur if the server encounters an unexpected error while aggregating results, though individual instance failures are generally handled gracefully within the `errors` field of a 200 OK response.

#### Deploy a unikernel

Deploys a new unikernel to a specified albatross server. This endpoint handles the upload of the unikernel binary and its configuration parameters.

**Endpoint:**

```http
POST /api/unikernel/create
```

**Content-Type:**
`multipart/form-data`

**Request Parameters:**

The request must be sent as multipart form data containing the following fields:

| Field                    | Type          | Required | Description                                                                      |
| :----------------------- | :------------ | :------- | :------------------------------------------------------------------------------- |
| `albatross_instance`     | string        | Yes      | The name of the albatross server instance where the unikernel will be deployed.  |
| `unikernel_name`         | string        | Yes      | The name to assign to the deployed unikernel.                                    |
| `unikernel_config`       | string (JSON) | Yes      | A JSON string containing resource and runtime configuration (see details below). |
| `unikernel_force_create` | string        | No       | Set to `"true"` to force creation (e.g., overwrite existing). Defaults to false. |
| `molly_csrf`             | string        | Yes      | Leave blank for API calls                                                        |
| `binary`                 | file          | Yes      | The unikernel image file (binary) to be uploaded.                                |

**Unikernel Configuration JSON (`unikernel_config`):**

The `unikernel_config` field must be a JSON string representing the following structure:

| JSON Field           | Type          | Description                                                                                                      |
| :------------------- | :------------ | :--------------------------------------------------------------------------------------------------------------- |
| `cpuid`              | int           | The CPU ID to pin the unikernel to.                                                                              |
| `memory`             | int           | Memory allocation in MB. Default is `32`.                                                                        |
| `arguments`          | list<string>  | Command line arguments to pass to the unikernel.                                                                 |
| `fail_behaviour`     | string/object | `"quit"` or `{"restart": <exit_codes>}` (e.g. `{"restart": null}` for always, or `{"restart": {"exit_code":}}`). |
| `block_devices`      | list          | List of objects: `{"name": "internal_name", "host_device": "host_vol", "sector_size": 512}`.                     |
| `network_interfaces` | list          | List of objects: `{"name": "service", "host_device": "tap0", "mac": "..."}`.                                     |
| `startup`            | int           | Startup priority (1-100). Default is `50`.                                                                       |

**Example Request:**

```bash
curl -X POST https://mollymawk.test/api/unikernel/create \
  -H "Authorization: Bearer <access_token>" \
  -F "albatross_instance=eu-west-2" \
  -F "unikernel_name=hello-world" \
  -F "unikernel_force_create=false" \
  -F "molly_csrf=''" \
  -F 'unikernel_config={
        "cpuid": 0,
        "memory": 64,
        "arguments": [],
        "fail_behaviour": "quit",
        "network_interfaces": [],
        "block_devices": []
      }' \
  -F "binary=@/path/to/hello_world.hvt"
```

**Success Response:**

- **Status Code:** `200 OK`
- **Body:** A JSON object returned indicating the result of the operation.

```json
{
  "status": 200,
  "title": "200",
  "success": true,
  "data": "created unikernel"
}
```

**Error Responses:**

- **Status Code:** `400 Bad Request`

  - `"One or more required fields are missing."` — Fields like `albatross_instance`, `unikernel_name`, `binary`, `unikernel_config`, `molly_csrf` are missing.
  - `"Invalid unikernel arguments: ..."` — The `unikernel_config` JSON string was invalid or contained incorrect types.
  - `"Error converting instance name: ..."` — The provided instance name format is invalid.
  - `"Error converting unikernel name: ..."` — The provided unikernel name format is invalid.

- **Status Code:** `404 Not Found`

  - `"Error finding albatross instance: ..."` — The specified `albatross_instance` does not exist in the system configuration.

- **Status Code:** `500 Internal Server Error`
  - `"Albatross Query Error: ..."` — The server failed to communicate with the albatross server.
  - `"Albatross Response Error: ..."` — The albatross server returned an error string (e.g., resource exhaustion, name conflict).
  - `"Albatross JSON Error: ..."` — The response from the albatross server could not be parsed.

#### Destroy a Unikernel

Destroys (stops and removes) an existing unikernel running on a specific albatross server.

**Endpoint:**

```http
POST /api/unikernel/destroy
```

**Content-Type:**
`application/json`

**Request Parameters:**

The request body must be a JSON object containing the following fields:

| Field                | Type   | Required | Description                                                      |
| :------------------- | :----- | :------- | :--------------------------------------------------------------- |
| `name`               | string | Yes      | The name of the unikernel to destroy.                            |
| `albatross_instance` | string | Yes      | The name of the albatross server where the unikernel is running. |

**Example:**

```bash
curl -X POST https://mollymawk.test/api/unikernel/destroy \
  -H "Authorization: Bearer <access_token>" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "hello-world",
    "albatross_instance": "eu-west-2"
  }'
```

**Success Response:**

- **Status Code:** `200 OK`
- **Body:** A JSON string or object returned confirming the destruction.

```json
{
  "status": 200,
  "title": "200",
  "success": true,
  "data": "destroyed unikernel"
}
```

**Failure Messages:**

- **Status Code:** `400 Bad Request`

  - `"Couldn't find unikernel name in json"` — The `name` or `albatross_instance` fields are missing.
  - `"Error converting instance name: ..."` — The provided instance name format is invalid.
  - `"Error converting unikernel name: ..."` — The provided unikernel name format is invalid.

- **Status Code:** `404 Not Found`

  - `"Error finding albatross instance: <instance_name>"` — The specified albatross server is not configured.

- **Status Code:** `500 Internal Server Error`
  - `"Error querying albatross: ..."` — Mollymawk failed to communicate with the albatross server.
  - `"failure: ..."` — The albatross server returned a failure (e.g., the unikernel was not found or could not be stopped).

#### Restart a Unikernel

Restarts a running unikernel on a specific albatross server. This sends a restart command to albatross, which attempts to stop and then start the unikernel.

**Endpoint:**

```http
POST /api/unikernel/restart
```

**Content-Type:**
`application/json`

**Request Parameters:**

The request body must be a JSON object containing the following fields:

| Field                | Type   | Required | Description                                                      |
| :------------------- | :----- | :------- | :--------------------------------------------------------------- |
| `name`               | string | Yes      | The name of the unikernel to restart.                            |
| `albatross_instance` | string | Yes      | The name of the albatross server where the unikernel is running. |

**Example Request:**

```bash
curl -X POST https://mollymawk.test/api/unikernel/restart \
  -H "Authorization: Bearer <access_token>" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "hello-world",
    "albatross_instance": "eu-west-2"
  }'
```

**Success Response:**

- **Status Code:** `200 OK`
- **Body:** A JSON object returned by the albatross server confirming the action.

```json
{
  "status": 200,
  "title": "200",
  "success": true,
  "data": "created unikernel"
}
```

**Failure Messages:**

- **Status Code:** `400 Bad Request`

  - `"Couldn't find unikernel name in json"` — The `name` or `albatross_instance` fields are missing from the request body.
  - `"Error converting instance name: ..."` — The provided instance name format is invalid.
  - `"Error converting unikernel name: ..."` — The provided unikernel name format is invalid.

- **Status Code:** `404 Not Found`

  - `"Error finding albatross instance: <instance_name>"` — The specified albatross server could not be found in the system configuration.

- **Status Code:** `500 Internal Server Error`
  - `"Error querying albatross: ..."` — Mollymawk failed to communicate with albatross.
  - The response may also contain an error string directly from Albatross if the restart operation failed on the VMM side (e.g., unikernel not found).

#### Stream Unikernel Console

Establishes a Server-Sent Events (SSE) connection to stream the console output of a running unikernel in real-time.

**Endpoint:**

```http
GET /api/unikernel/console
```

**Content-Type:**
`text/event-stream`

**Request Parameters:**

The parameters are passed as HTTP Query Parameters (URL arguments):

| Parameter   | Type   | Required | Description                                                               |
| :---------- | :----- | :------- | :------------------------------------------------------------------------ |
| `unikernel` | string | Yes      | The name of the unikernel to monitor.                                     |
| `instance`  | string | Yes      | The name of the albatross server instance where the unikernel is running. |

**Example Request:**

```bash
curl -X GET "https://mollymawk.test/api/unikernel/console?unikernel=hello-world&instance=eu-west-2" \
  -H "Authorization: Bearer <access_token>" \
  -H "Accept: text/event-stream"
```

**Success Response:**

- **Status Code:** `200 OK`
- **Headers:**
  - `Content-Type: text/event-stream`
  - `Cache-Control: no-cache` (Typical for SSE, implied by streaming logic)
- **Body:** A stream of event data. Each event contains a JSON object with a timestamp and the console line.

```text
data:{"timestamp":"2023-10-27T10:05:00Z","line":"[INFO] Application started"}

data:{"timestamp": "2025-12-08T17:45:30-00:00","line": "2025-12-08T17:45:30-00:00: [INFO] [application] Hello World!"}
```

**Failure Messages:**

- **Status Code:** `400 Bad Request`

  - `"Couldn't convert unikernel name: <error>"` — The provided unikernel name is invalid format.
  - `"Couldn't find <param_name> in query params"` — The `unikernel` or `instance` parameter is missing.
  - `"Error with albatross instance name: <error>"` — The instance name provided is invalid.

- **Status Code:** `404 Not Found`

  - `"Couldn't find albatross instance with name: <instance_name>"` — The specified albatross server does not exist.

- **Status Code:** `500 Internal Server Error`
  - May return a general error message if the Albatross backend connection fails during the setup of the stream.

### Volume Operations

#### Create a block device

Creates a new block storage volume on a specified server and (optionally) populates it with data uploaded via the request.

**Endpoint:**

```http
POST /api/volume/create
```

**Content-Type:**
`multipart/form-data`

**Request Parameters:**

The request must be sent as a multipart form containing the following parts:

| Part Name            | Type          | Required | Description                                                                         |
| :------------------- | :------------ | :------- | :---------------------------------------------------------------------------------- |
| `albatross_instance` | string        | Yes      | The name of the albatross server instance where the volume will be created.         |
| `block_data`         | file          | Yes      | The binary data file to initialize the volume with. You can pass undefined or null. |
| `json_data`          | string (JSON) | Yes      | A JSON string containing the volume configuration (details below).                  |
| `molly_csrf`         | string        | Yes      | You can pass an empty string for this API endpoint.                                 |

**JSON Data Structure (`json_data`):**

The `json_data` form part must be a stringified JSON object with the following fields:

| JSON Field         | Type   | Description                                                           |
| :----------------- | :----- | :-------------------------------------------------------------------- |
| `block_name`       | string | The name to assign to the new volume.                                 |
| `block_size`       | int    | The size of the volume in megabytes (MB).                             |
| `block_compressed` | bool   | Whether the volume data should be compressed/decompressed by the VMM. |

**Example Request:**

```bash
curl -X POST https://mollymawk.test/api/volume/create \
  -H "Authorization: Bearer <access_token>" \
  -F "albatross_instance=eu-west-2" \
  -F "block_data=@/path/to/disk_image.img" \
  -F "molly_csrf=''" \
  -F 'json_data={
        "block_name": "data_vol_1",
        "block_size": 1024,
        "block_compressed": false
      }'
```

**Success Response:**

- **Status Code:** `200 OK`
- **Body:** A JSON object returned by the albatross server confirming the operation.

```json
{
  "status": 200,
  "title": "200",
  "success": true,
  "data": "set block device"
}
```

**Failure Messages:**

- **Status Code:** `400 Bad Request`

  - `"One or more required fields are missing."` — A required form part (e.g., `json_data`, `block_data`) is missing.
  - `"Invalid block name: <error>"` — The provided block name contains invalid characters or format.
  - `"Error converting name to albatross instance <error>"` — The instance name format is invalid.
  - `"unexpected field. got <error>"` — The JSON data contained unexpected fields or structure.
  - `"expected a dictionary"` — The `json_data` string was not a valid JSON object.

- **Status Code:** `404 Not Found`

  - `"Couldn't find albatross instance with name <name>"` — The specified `albatross_instance` does not exist.

- **Status Code:** `500 Internal Server Error`
  - `"an error with albatross. got <error>"` — The server failed to communicate with the VMM or the VMM rejected the creation request (e.g., insufficient storage space).

#### Delete a block device

Deletes an existing block storage volume from a specified albatross server.

**Endpoint:**

```http
POST /api/volume/delete
```

**Content-Type:**
`application/json`

**Request Parameters:**

The request body must be a JSON object containing the following fields:

| Field                | Type   | Required | Description                                                            |
| :------------------- | :----- | :------- | :--------------------------------------------------------------------- |
| `block_name`         | string | Yes      | The name of the volume to be deleted.                                  |
| `albatross_instance` | string | Yes      | The name of the albatross server instance where the volume is located. |

**Example:**

```bash
curl -X POST https://mollymawk.test/api/volume/delete \
  -H "Authorization: Bearer <access_token>" \
  -H "Content-Type: application/json" \
  -d '{
    "block_name": "my-data-vol",
    "albatross_instance": "eu-west-2",
  }'
```

**Success Response:**

- **Status Code:** `200 OK`
- **Body:** A JSON object returned confirming the removal.

```json
{
  "status": 200,
  "title": "200",
  "success": true,
  "data": "removed block"
}
```

**Failure Messages:**

- **Status Code:** `400 Bad Request`

  - `"Couldn't find block name in json"` — The `block_name` or `albatross_instance` fields are missing.
  - `"Error converting instance name: <error>"` — The provided instance name format is invalid.
  - `"Error converting block name: <error>"` — The provided block name format is invalid.

- **Status Code:** `404 Not Found`

  - `"Couldn't find albatross instance with name: <name>"` — The specified Albatross instance does not exist.

- **Status Code:** `500 Internal Server Error`
  - `"Error querying albatross: <error>"` — Mollymawk failed to communicate with the albatross server.
  - `"<albatross error>"` — The albatross server returned a failure (e.g., if the volume does not exist or cannot be removed).

#### Upload Data to a block device

Uploads data to an existing block device on a specific albatross server. This is used to overwrite a volume with a disk image or binary data.

_(Note: We can't upload to a block device which is currently attached to a running unikernel)._

**Endpoint:**

```http
POST /api/volume/upload
```

**Content-Type:**
`multipart/form-data`

**Request Parameters:**

The request must be sent as multipart form data.

| Part Name            | Type          | Required | Description                                                        |
| :------------------- | :------------ | :------- | :----------------------------------------------------------------- |
| `albatross_instance` | string        | Yes      | The name of the albatross server instance hosting the volume.      |
| `molly_csrf`         | string        | **Yes**  | Leave empty                                                        |
| `json_data`          | string (JSON) | Yes      | A JSON string containing the volume configuration (details below). |
| `block_data`         | file          | Yes      | The binary data file to upload to the volume.                      |

**JSON Data Structure (`json_data`):**

The `json_data` form part must be a stringified JSON object containing the following fields:

| JSON Field         | Type   | Required | Description                                                                                                                                                                                                      |
| :----------------- | :----- | :------- | :--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `block_name`       | string | Yes      | The name of the target volume.                                                                                                                                                                                   |
| `block_compressed` | bool   | Yes      | Whether the uploaded data should be handled as compressed by the albatross server.                                                                                                                               |
| `block_size`       | int    | Yes\*    | The size of the block volume in MB. _(Note: While strictly required by the server's input parser, this value is ignored during the upload process and only considered when we are creating a new block device)._ |

**Example Request:**

```bash
curl -X POST https://mollymawk.test/api/volume/upload \
  -H "Authorization: Bearer <access_token>" \
  -F "albatross_instance=eu-west-2" \
  -F "molly_csrf=''" \
  -F "block_data=@/path/to/image.iso" \
  -F 'json_data={
        "block_name": "vol1",
        "block_compressed": false,
        "block_size": 0
      }'
```

**Success Response:**

- **Status Code:** `200 OK`
- **Body:** A JSON object wrapping the Albatross response.

```json
{
  "status": 200,
  "title": "200",
  "success": true,
  "data": "set block device"
}
```

**Failure Messages:**

- **Status Code:** `400 Bad Request`

  - `"One or more required fields are missing."` — Request is missing `albatross_instance`, `json_data`, or `block_data`.
  - `"unexpected field. got <json>"` — The `json_data` structure was incorrect (e.g., missing `block_size`).
  - `"expected a dictionary"` — The `json_data` was not a valid JSON object.
  - `"Invalid block name: <error>"` — The block name format is invalid.
  - `"Error converting name to albatross instance <error>"` — The instance name format is invalid.

- **Status Code:** `404 Not Found`

  - `"Couldn't find albatross instance with name <name>"` — The specified albatross server does not exist.

- **Status Code:** `500 Internal Server Error`
  - `"an error with albatross. got <error>"` — The albatross server failed to write the data (e.g., volume does not exist, connection failed).

#### Downloading Data from a block device

Initiates a stream to download the content of a specific block device from an albatross server. The server sets the response headers to trigger a file download (`application/octet-stream`).

_(Note: We can't download from a block device which is currently attached to a running unikernel)._

**Endpoint:**

```http
POST /api/volume/download
```

**Content-Type:**
`application/json`

**Request Parameters:**

The request body must be a JSON object containing the following fields:

| Field                | Type   | Required | Description                                                     |
| :------------------- | :----- | :------- | :-------------------------------------------------------------- |
| `albatross_instance` | string | Yes      | The name of the albatross server hosting the volume.            |
| `block_name`         | string | Yes      | The name of the volume to download.                             |
| `compression_level`  | int    | Yes      | The compression level to apply (e.g., `0` for none, up to `9`). |

**Example Request:**

```bash
curl -X POST https://mollymawk.test/api/volume/download \
  -H "Authorization: Bearer <access_token>" \
  -H "Content-Type: application/json" \
  -d '{
    "albatross_instance": "eu-west-2",
    "block_name": "data_vol_1",
    "compression_level": 5,
  }' \
  --output data_vol_1_dump.img
```

**Success Response:**

- **Status Code:** `200 OK`
- **Headers:**
  - `Content-Type`: `application/octet-stream`
  - `Content-Disposition`: `attachment; filename="data_vol_1_dump"`
- **Body:** A binary stream of the volume data.

**Failure Messages:**

- **Status Code:** `400 Bad Request`

  - `"Couldn't find block name in json"` — Required fields (instance, block name, compression) are missing from the JSON body.
  - `"Couldn't convert instance name: <error>"` — The provided instance name format is invalid.
  - `"Couldn't convert block name: <error>"` — The provided block name format is invalid.

- **Status Code:** `404 Not Found`

  - `"Couldn't find albatross instance with name: <name>"` — The specified Albatross instance does not exist in the configuration.

- **Status Code:** `500 Internal Server Error`
  - May occur if the server fails to establish the data stream with the Albatross backend.

### Token Management

#### Create a new API Key (Token)

Generates a new long-lived API token (Bearer token) for the authenticated user. This token can be used to authenticate requests to other API endpoints (e.g., deploying unikernels via CI/CD pipelines).

**Endpoint:**

```http
POST /api/tokens/create
```

**Content-Type:**
`application/json`

**Request Parameters:**

The request body must be a JSON object containing the following fields:

| Field          | Type   | Required | Description                                             |
| :------------- | :----- | :------- | :------------------------------------------------------ |
| `token_name`   | string | Yes      | A descriptive name for the token (e.g., "CI Pipeline"). |
| `token_expiry` | int    | Yes      | The validity period of the token in seconds.            |

**Example Request:**

```bash
curl -X POST https://mollymawk.test/api/tokens/create \
  -H "Authorization: Bearer <existing_token>" \
  -H "Content-Type: application/json" \
  -d '{
    "token_name": "Gitlab CI",
    "token_expiry": 400000,
  }'
```

**Success Response:**

- **Status Code:** `200 OK`
- **Body:** A JSON object representing the newly created token, including its secret value.

```json
{
  "status": 200,
  "title": "200",
  "success": true,
  "data": {
    "token_type": "Bearer",
    "value": "3cb6bc72-a52d-4f4f-969c-c85465875774",
    "expires_in": 400000,
    "created_at": "2025-12-09 05:12:11-00:00",
    "last_access": "2025-12-09 05:12:11-00:00",
    "name": "Gitlab CI",
    "usage_count": 0
  }
}
```

**Failure Messages:**

- **Status Code:** `400 Bad Request`

  - `"Create token: Unexpected fields..."` — The `token_name` or `token_expiry` fields are missing or malformed.
  - `"Auth token not found..."` — Missing authentication (if not using a valid header).

- **Status Code:** `500 Internal Server Error`
  - `"Storage error with <details>"` — The server failed to save the new token to the database.

#### Upade an API Key

Updates the name and expiration settings of an existing API token. This endpoint allows modifying the metadata of a token without changing its secret value.

**Endpoint:**

```http
POST /api/tokens/update
```

**Content-Type:**
`application/json`

**Request Parameters:**

The request body must be a JSON object containing the following fields:

| Field          | Type   | Required | Description                                              |
| :------------- | :----- | :------- | :------------------------------------------------------- |
| `token_value`  | string | Yes      | The UUID string of the token to be updated.              |
| `token_name`   | string | Yes      | The new descriptive name for the token.                  |
| `token_expiry` | int    | Yes      | The new validity period in seconds (from creation time). |

**Example Request:**

```bash
curl -X POST https://mollymawk.test/api/tokens/update \
  -H "Authorization: Bearer <access_token>" \
  -H "Content-Type: application/json" \
  -d '{
    "token_value": "3cb6bc72-a52d-4f4f-969c-c85465875774",
    "token_name": "updated-gitlab-token",
    "token_expiry": 800000
  }'
```

**Success Response:**

- **Status Code:** `200 OK`
- **Body:** A JSON object representing the updated token details.

```json
{
  "status": 200,
  "title": "200",
  "success": true,
  "data": {
    "token_type": "Bearer",
    "value": "3cb6bc72-a52d-4f4f-969c-c85465875774",
    "expires_in": 800000,
    "created_at": "2025-12-09 05:12:11-00:00",
    "last_access": "2025-12-09 05:12:11-00:00",
    "name": "updated-gitlab-token",
    "usage_count": 0
  }
}
```

**Failure Messages:**

- **Status Code:** `400 Bad Request`

  - `"Update token: Unexpected fields. Got ..."` — Required fields (`token_name`, `token_expiry`, or `token_value`) are missing or invalid types.
  - `"Auth token not found"` — The Authorization header is missing

- **Status Code:** `404 Not Found`

  - `"Token not found"` — The provided `token_value` does not match any token associated with the user.

- **Status Code:** `500 Internal Server Error`
  - `"Storage error with <details>"` — The server failed to save the updated token to the database.

#### Delete an API Key

Revokes and removes a specific API token from the user's account. This invalidates the token immediately, preventing it from being used for future authentication.

**Endpoint:**

```http
POST /api/tokens/delete
```

**Content-Type:**
`application/json`

**Request Parameters:**

The request body must be a JSON object containing the following fields:

| Field         | Type   | Required | Description                                 |
| :------------ | :----- | :------- | :------------------------------------------ |
| `token_value` | string | Yes      | The UUID string of the token to be deleted. |

**Example Request:**

```bash
curl -X POST https://mollymawk.test/api/tokens/delete \
  -H "Authorization: Bearer <access_token>" \
  -H "Content-Type: application/json" \
  -d '{
    "token_value": "d3b07384-d113-4ec6-973d-285854e7432f",
  }'
```

**Success Response:**

- **Status Code:** `200 OK`
- **Body:** A JSON object confirming the deletion.

```json
{
  "status": 200,
  "title": "OK",
  "success": true,
  "data": "Token deleted succesfully"
}
```

**Failure Messages:**

- **Status Code:** `400 Bad Request`

  - `"Delete token: Unexpected fields..."` — The `token_value` field is missing from the JSON body.
  - `"Auth token not found"` — The authorization header is missing.

- **Status Code:** `500 Internal Server Error`
  - `"Storage error with <details>"` — An error occurred while trying to save the updated user record to the database.

### Administration

#### Retry Albatross Connection (Admin)

Attempts to re-initialize the connection to a specific albatross server. This is useful if the instance has become unreachable or disconnected from the dashboard. This endpoint requires administrator privileges.

**Endpoint:**

```http
GET /api/admin/albatross/retry
```

**Content-Type:**
`application/json`

**Request Parameters:**

The request uses HTTP Query Parameters (URL arguments).

| Parameter  | Type   | Required | Description                                                |
| :--------- | :----- | :------- | :--------------------------------------------------------- |
| `instance` | string | Yes      | The name of the albatross server instance to reconnect to. |

**Example Request:**

```bash
curl -X GET "https://mollymawk.test/api/admin/albatross/retry?instance=eu-west-2" \
  -H "Authorization: Bearer <access_token>"
```

**Success Response:**

- **Status Code:** `200 OK`
- **Body:** A JSON object confirming the connection status.

```json
{
  "status": 200,
  "title": "200",
  "success": true,
  "data": "Re-initialization successful, instance is back online"
}
```

**Failure Messages:**

- **Status Code:** `400 Bad Request`

  - `"Couldn't find instance in query params"` — The `instance` parameter is missing.
  - `"Error with albatross instance name: <error>"` — The provided instance name format is invalid.
  - `"You don't have the necessary permissions..."` — The user is not an administrator.

- **Status Code:** `404 Not Found`

  - `"Couldn't find albatross instance with name: <name>"` — The specified instance name is not found in the configuration.

- **Status Code:** `500 Internal Server Error`
  - `"Re-initialization failed. See error logs: <error>"` — The server could not establish a connection to the Albatross instance (e.g., network error, invalid certificates).
