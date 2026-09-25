# Caller ID API

CallerAPI helps carriers, CPaaS, UCaaS and VoIP providers increase ARPU by offering voice fraud protection to their customers. Get Tier 1 level intelligence, sell it under your own brand, make your users and regulators happy while increasing profits.

One GET returns the business name, a spam status, and the complaint evidence. The carrier keeps its own brand on the screen. Get a key in the [dashboard](https://callerapi.com/dashboard). Plans are on the [pricing page](https://callerapi.com/pricing).

## Sell it as a white-label VAS

1. Send `GET /api/lookup/{phone}` with your `X-Auth` key.
2. Show `business_info.business_name` on the caller screen under your brand.
3. Warn the subscriber when `is_spam` is true.
4. Block the call when `spam_score` crosses your threshold.
5. Charge for that protection as a monthly add-on.

A verified business can still have complaints. Read `is_spam` and `spam_score` before you treat the call as safe. `reputation` is `VERIFIED` only when the business record is claimed.

Each lookup costs 1 credit. Carrier data (HLR) is off unless you pass `hlr=true`. HLR uses the same credit.

## Authentication

Send your API key on every request in the `X-Auth` header.

```
X-Auth: YOUR_API_KEY
```

Replace `YOUR_API_KEY` with the key from the dashboard.

## Check credits

```
GET https://callerapi.com/api/me
```

```json
{
  "status": "success",
  "email": "ops@example.com",
  "credits_spent": 23498,
  "credits_monthly": 250000,
  "credits_left": 226502
}
```

`credits_left` is the remaining lookups for the current plan period.

```bash
curl -H 'X-Auth: YOUR_API_KEY' 'https://callerapi.com/api/me'
```

## Look up a phone number

```
GET https://callerapi.com/api/lookup/{phone}
```

`{phone}` is the number in +E.164 form, for example `+16502530000`. The API also accepts the same digits without the plus. If your HTTP client turns `+` into a space, encode it as `%2B`.

Add `?hlr=true` when you need the current carrier, the line type, and the ported status. HLR is off unless you set that flag. HLR adds wait time. It does not add a credit.

### Response

This sample is a live lookup of `+16502530000`. The complaint list is cut to one row. Counts in the sample change over time. `total_complaints` is the full count. `complaints` holds the newest rows, up to 200.

```json
{
  "status": "success",
  "data": {
    "phone": "+16502530000",
    "entity_type": "BUSINESS",
    "reputation": "VERIFIED",
    "total_complaints": 96,
    "is_spam": true,
    "spam_score": 99,
    "business_info": {
      "business_name": "Google",
      "category": "services",
      "city": "Mountain View",
      "industry": "Internet Services",
      "state": "CA",
      "country": "US",
      "verified": true
    },
    "complaints": [
      {
        "CreatedDate": "2026-08-18T12:37:29Z",
        "ViolationDate": "2026-08-18T12:37:29Z",
        "ConsumerState": "California",
        "Subject": "Other",
        "RecordedMessageOrRobocall": "Y"
      }
    ]
  }
}
```

The payload can include more fields. Read the fields on this page.

### Fields

`entity_type` is `BUSINESS` when a business record exists. Otherwise it is `UNKNOWN`.

`reputation` is `VERIFIED` when that business record is claimed. It is `SPAM` when complaints exist and the record is not claimed. It is `UNKNOWN` when there is no business record and no complaints.

`is_spam` is true when at least one complaint exists.

`spam_score` is an integer from 0 to 100. A new complaint scores 100. The score is lower when the newest complaint is older. The score is 0 when that complaint is older than 180 days.

`total_complaints` is the full complaint count. The `complaints` array is capped at 200 rows.

`business_info` is null when no business record matches. When it matches, it includes `business_name`, `category`, `industry`, `city`, `state`, `country`, and `verified`.

Each complaint includes:

- `CreatedDate`: when the report was stored
- `ViolationDate`: when the call happened
- `ConsumerState`: region on the report
- `Subject`: report category
- `RecordedMessageOrRobocall`: `Y` for a recorded call or robocall, otherwise `N`

### Carrier data (HLR)

`carrier_info` is present only when the request includes `hlr=true`. This sample is the same number.

```json
{
  "country": {
    "code": "1",
    "iso": "US",
    "name": "United States"
  },
  "network": {
    "carrier": "Lumen",
    "type": "CLEC",
    "ocn": "8826",
    "spid": "8824",
    "original": {
      "carrier": "Verizon Business",
      "spid": "8824"
    }
  },
  "number": {
    "msisdn": "16502530000",
    "local_format": "16502530000",
    "type": "CLEC",
    "landline": true,
    "mobile": false,
    "lrn": "14159686199",
    "valid": "true",
    "ported": true,
    "ported_date": "2014-12-23T16:07:47Z",
    "timezone": "America/Los_Angeles"
  }
}
```

`network.carrier` is the current carrier. `network.original.carrier` is the carrier that first issued the number. `number.ported` is true when those carriers differ. `number.landline` and `number.mobile` say the line type. `number.valid` is the string `true` or `false`.

If the HLR check fails, `carrier_info.error` is a string. The name and spam fields still return.

### Errors

A missing or invalid key returns HTTP 401. The body is `{"error":"Missing API key"}` or `{"error":"Invalid API key"}`.

No credits left returns HTTP 402. The body includes `credits_needed`.

A missing phone number returns HTTP 400.

A failed complaint load or business load returns HTTP 500. The name and spam fields are absent on that response.

## Code examples

<details>
<summary>cURL</summary>

```bash
curl -H 'X-Auth: YOUR_API_KEY' \
  'https://callerapi.com/api/lookup/+16502530000'
```

With carrier data:

```bash
curl -H 'X-Auth: YOUR_API_KEY' \
  'https://callerapi.com/api/lookup/+16502530000?hlr=true'
```

</details>

<details>
<summary>Python</summary>

```python
import requests

phone = "+16502530000"
url = f"https://callerapi.com/api/lookup/{phone}"
headers = {"X-Auth": "YOUR_API_KEY"}

response = requests.get(url, headers=headers)
print(response.json())
```

With carrier data, pass `params={"hlr": "true"}`.

</details>

<details>
<summary>JavaScript</summary>

```javascript
const phone = '+16502530000';

fetch(`https://callerapi.com/api/lookup/${encodeURIComponent(phone)}`, {
  headers: {
    'X-Auth': 'YOUR_API_KEY'
  }
})
  .then((response) => response.json())
  .then((data) => console.log(data))
  .catch((error) => console.error(error));
```

With carrier data, add `?hlr=true` to the URL.

</details>

<details>
<summary>PHP</summary>

```php
$phone = rawurlencode('+16502530000');
$ch = curl_init("https://callerapi.com/api/lookup/$phone");
curl_setopt($ch, CURLOPT_RETURNTRANSFER, 1);
curl_setopt($ch, CURLOPT_HTTPHEADER, array('X-Auth: YOUR_API_KEY'));

$response = curl_exec($ch);
curl_close($ch);

$data = json_decode($response, true);
print_r($data);
```

With carrier data, append `?hlr=true` to the URL.

</details>

Full product site: [callerapi.com](https://callerapi.com/).
