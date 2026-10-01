---
description: "Outbound calls whose response lands back on the record."
---

# External services

::: tip Who can do this
<Perm role="PME" /> — the application role that opens **Settings**.
:::

An external service is an outbound HTTP call Exto makes on your behalf, whose
response is mapped back onto the record. Use one to fetch a value from a system
of record, validate against it, or push a record into it as part of an action.

They are configured per module, in **Module designer → External services**.

## List page

Every service defined on the module. Each has a **type**:

- **Field** — attached to a field, called when that field is used.
- **Form** — attached to the form, called on load or on demand.

A service can also be attached to a **workflow action**, so pressing Approve
makes the call. See [Steps & actions](/build/steps-and-actions).

## Defining a service

### Request

| Setting | Values |
| --- | --- |
| **Name** | Required. |
| **Type** | Field or Form. |
| **Method** | `GET`, `POST`, `PUT`, `PATCH`, `DELETE`. |
| **URL** | Required. |
| **Timeout** | Milliseconds, from 1,000 to 1,800,000. Defaults to 3,000. |

### Authentication

| Type | Needs |
| --- | --- |
| **None** | — |
| **Basic** | Username and password. |
| **Bearer** | A token. |
| **User MS access token** | Uses the signed-in user's Microsoft token. |
| **User Google access token** | Uses the signed-in user's Google token. |
| **Connectors** | A connector type — MS Dynamics, AutoDesk or SAP — and a connection. |

The two user-token options call out **as the user**, so the external system
sees who is acting rather than a shared service account.

### Setup tabs

Below the type, name, authentication, method and URL, a service is configured
across six tabs: **Request mapping**, **Response mapping**, **Response fields**,
**Success Script**, **Error mapping** and **Configuration**. The first three
decide what is sent, what comes back, and where it lands.

### Request mapping

Each row copies one value into the call.

| Column | Meaning |
| --- | --- |
| **Source field** | Where the value comes from — the record, such as `ctx.record.projectId`, or a form field, such as `ctx.mainForm.survey_type`. |
| **Target field** | Where it goes on the call. Start with `body.` to send it in the request body, then the API's key, matching its case exactly — `body.tenantId`. |
| **Required** | Blocks the call when the value is empty. Tick it for values the API cannot work without. |
| **Code** | Opens a script for the row. |

A `GET` call sends no body, so its request mapping usually stays empty.

### Response mapping

Each row takes a value from the reply and writes it onto the form.

- **Source field** — the key in the reply, such as `name`. Use it directly when
  the reply already holds the value to show.
- **Target field** — where the value goes. `*` means the row's result *is* the
  response, which is how a script reshapes the whole reply.
- **Code** — click `</>` to open the row's **Transform script**.
  `getResponseBody()` returns the reply; return the value the form should get.

::: tip Always return something
End a transform script with a fallback, such as `return [];`, so a failed or
empty reply does not break the field.
:::

### Response fields

**Response fields** describe what to read back. Response mapping can only pick
fields declared here, so add one row per value you want to use.

| Column | Meaning |
| --- | --- |
| **Name** | The key in the reply. It must match exactly. |
| **Label** | The name shown in the response mapping picker. |
| **Type** | **Text** for one value, or **Array** for a list with its own child fields. |

Array fields carry children — which is how a nested response maps onto a
sub-table.

### Mapping options

- **Add tenant information** — includes tenant context in the request.
- **Add project information** — includes project context.
- **Add user information** — includes the calling user.
- **Force single object as response mapping when response is an array** — takes
  the first element when the endpoint returns a list but the field expects one
  value. It sits on the **Configuration** tab. Turn it **off** to fill a
  sub-table, so one lookup can fill several rows at once.

### Logging

| Setting | Effect |
| --- | --- |
| **Level** | `Debug`, `Info`, `Warn` or `Error`. |
| **Log data after mapping** | Records the mapped result, not just the raw response. |
| **Log request and response** | Records both bodies. |
| **Log headers** | Records headers too. |

::: warning Logging bodies logs whatever is in them
Request and response logging captures payloads verbatim, including anything
sensitive the endpoint returns. Raise the level while debugging, then lower it.
:::

## Example: a Field service that posts

`Category_Lookup` is a **Field** service behind an auto-complete field that
looks up site categories.

| Setting | Value |
| --- | --- |
| **Type** | Field |
| **Name** | `Category_Lookup` |
| **Authentication type** | None — the sample API needs none. Pick whatever your API needs. |
| **Method** | `POST` |
| **URL** | `https://api.example.com/v1/category-lookup` |
| **Form field** | Category (`site_category`) on the Site Survey Form |

### 1. Map form values to the request body

<Shot src="integrations/es-field-request-mapping" alt="Request mapping for Category_Lookup"
  caption="Three rows send the tenant, the project and the selected survey type." />

1. **Method** — any method the API expects; here `POST`.
2. **URL** — the address the service calls.
3. **Request mapping** — the first setup tab.
4. **Source field** — `ctx.record.tenantId`, `ctx.record.projectId` and
   `ctx.mainForm.survey_type`.
5. **Target field** — `body.tenantId`, `body.projectId` and `body.surveyType`.
6. **Add** — one row per value the API needs.
7. **Row options** — tick **Required** for values the API cannot work without.

The API receives:

```json
{
  "tenantId": "ABCDEFG",
  "projectId": "PRJABCD",
  "surveyType": "Structural"
}
```

### 2. Link an auto-complete field to the service

Save the service first — a field's **External service** list only shows
services saved on the same module.

<Shot src="integrations/es-field-autocomplete-link" alt="An auto-complete field pointed at Category_Lookup"
  caption="The Category field's properties, with External service as its data source." />

1. **Auto-complete** — add one from the **Selection** group.
2. **The field** — click it on the form; its properties open on the right.
3. **Name** — such as `site_category`. Refer to it in request mapping as
   `ctx.mainForm.site_category`. It is locked once the field is saved.
4. **Data source** — **External service**.
5. **External service** — `Category_Lookup`.
6. **Mandatory** — optional; tick it if users must pick a value before saving.

See [Forms](/build/forms) and [Field types](/build/field-types).

### 3. Map the response to the form

<Shot src="integrations/es-field-response-mapping" alt="Response mapping with a transform script"
  caption="One row, target *, with a transform script that keeps only active categories." />

1. **Response mapping** tab.
2. **Source field** — a key in the reply; unused here, because the script
   shapes the reply.
3. **Target field** — `*`, so the script's result is the response.
4. **Code** — `</>` opens the transform script.
5. **Transform script** — below.
6. **Add** — one row per value you map.

The API replies with every category:

```json
{
  "success": true,
  "categories": [
    { "code": "STR", "name": "Structural", "active": true },
    { "code": "ELE", "name": "Electrical", "active": false }
  ]
}
```

The script returns only the active ones, so users see Structural but not
Electrical:

```js
const resp = getResponseBody();
if (resp.success) {
  return resp.categories.filter(c => c.active);
}
return [];
```

### 4. Declare the response fields

<Shot src="integrations/es-field-response-fields" alt="The Response fields tab for Category_Lookup"
  caption="One Text field, name, labelled Category." />

1. **Response fields** tab.
2. **Name** — `name`, matching the reply exactly.
3. **Label** — `Category`, shown in the response mapping picker.
4. **Type** — **Text**.
5. **Add** — one row per value in the reply.

With the same reply you could declare either shape:

```text
name          Text    "Structural"

categories    Array
  code        Text    "STR"
  name        Text    "Structural"
```

## Example: a Get request

`User_Lookup` is also a **Field** service, but it calls the API with `GET`.
Picking a user in an auto-complete field also fills an auto-populated field.

| Setting | Value |
| --- | --- |
| **Type** | Field |
| **Name** | `User_Lookup` |
| **Authentication type** | None |
| **Method** | `GET` |
| **URL** | `https://api.example.com/v1/users` |
| **Request mapping** | None — leave it empty for `GET`. |
| **Form fields** | Inspector (`inspector`, auto-complete) and Country (auto populate) on the Site Survey Form |

### 1. Map the response

<Shot src="integrations/es-get-response-mapping" alt="Response mapping for User_Lookup"
  caption="Method Get, and one response mapping row with a transform script." />

1. **Method** — `GET`. Nothing is sent in the body.
2. **Response mapping** tab.
3. **Code** — `</>` opens the transform script.
4. **Transform script** — joins `firstName` and `lastName` into `fullName`, and
   lifts `country` out of `address`.

```js
const resp = getResponseBody();
const users = Array.isArray(resp) ? resp : (resp.users || []);
return users.map(u => ({
  fullName: `${u.firstName} ${u.lastName}`,
  country: u.address?.country || ''
}));
```

| One user in the reply | What the script returns |
| --- | --- |
| `{ "id": 1, "firstName": "Emily", "lastName": "Johnson", "address": { "city": "Phoenix", "country": "United States" } }` | `[{ "fullName": "Emily Johnson", "country": "United States" }]` |

::: warning Map only what the form needs
APIs often return far more than you use, including personal details. Return
only the values the form shows.
:::

### 2. Declare the response fields

<Shot src="integrations/es-get-response-fields" alt="The Response fields tab for User_Lookup"
  caption="Both values the script returns, declared as Text." />

1. **Response fields** tab.
2. **`fullName`** — Text. The name shown in the Inspector list.
3. **`country`** — Text. The value for the auto-populated Country field.

### 3. Link the form fields

Link the Inspector field as in the first example, then add the Country field.

<Shot src="integrations/es-get-form-fields" alt="Inspector linked to User_Lookup, with Country beside it"
  caption="Inspector calls User_Lookup; Country is filled from its reply." />

1. **Inspector** — the auto-complete field, named `inspector`.
2. **Data source** — **External service**.
3. **External service** — `User_Lookup`.
4. **Country** — the auto populate field. It stays empty until a user is picked
   in Inspector, then shows `country`.

A form user types "Emi" in Inspector, picks **Emily Johnson**, and Country
shows **United States**.

## Reading the result

A service's result is stamped onto the record and shown in its **External
service** widget. An **Auto populate** field can copy a value from a service
directly into a field — see [Forms](/build/forms).

## The log

**Settings → External service log** records outbound calls, in the detail the
service's log level allows. When a service appears to do nothing, this is where
the reason is — usually a timeout, an auth failure, or a response shape that
does not match the mapping.

<Shot src="integrations/external-service" alt="The external service log"
  caption="One call's summary, with the request and response data after mapping." />

## Permissions

Defining external services, and reading the external service log, both happen
under **Settings** and require <Perm role="PME" />.

The call itself runs as the *user who triggered it* when the service uses a
user access token, and as the configured credentials otherwise — so check which
identity the far end will see before choosing an authentication type.
