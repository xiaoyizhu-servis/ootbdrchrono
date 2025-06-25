
# DrChrono Integration Setup Guide

This guide walks through the configuration steps for integrating DrChrono with Servis.ai.

---

## 1. DrChrono API Setup

**Login to DrChrono**
Navigate to: **Account → API**
Fill in the following fields:

### Required Fields

* **Redirect URIs**
  `https://freeagent.network/integration-oauth/callback`

* **Callback URL**
Use the following URL format, but replace the UUID at the end with your **current `team_id`**:

```
https://freeagent.network/drchronowebhook/{team_id}
```

**Example:**

```
https://freeagent.network/drchronowebhook/6c61e96d-214b-4bbd-a0c2-8ff5ff2e5e41
```

* **Secret Token**
  `servis`

* **Active**
  ✅ Select to enable

* **Copy**

  * `Client ID`
  * `Client Secret`

    > You’ll need these for the Servis.ai integration step.

### Events to Subscribe

* `APPOINTMENT_CREATE`
* `APPOINTMENT_DELETE`
* `APPOINTMENT_MODIFY`
* `PATIENT_CREATE`
* `PATIENT_MODIFY`

Once complete, **Save** the config. Then click **Verify** under the **Webhook** section.
✅ Wait a few seconds for status to show: `This webhook is verified`.

---

## 2. Servis.ai Integration Configuration

**Login to Servis.ai**
Navigate to: **SETTINGS → INTEGRATIONS → Integrations → Add Integration**

### Required Fields

* **Name**
  `OOTB DrChrono Integration`

  > ⚠ Make sure name is **exactly** the same.

* **Authentication Type**
  `OAuth`

* **Authentication Code**
  Use the following template and **replace** `client_id` and `client_secret` with the values copied from the **DrChrono API setup section**:

  ```jsx
  (async function(inputs, context){
    try {
      const client_id = ''; // from DrChrono API setup
      const client_secret = ''; // from DrChrono API setup

      console.log("Is user authorized?: ", inputs.is_authorized);
      console.log("Last authorized at:", inputs.last_authorized_at);
      console.log("access_token is:", inputs.access_token);
      console.log("refresh_token is:", inputs.refresh_token);
      console.log("access token expires at", inputs.expires_at);
      
      const currentDate = new Date();
      const expiryDate = new Date(inputs.expires_at);
      
      if (currentDate.getTime() > expiryDate.getTime()) {
        console.log("ACCESS TOKEN EXPIRED, REFRESHING");
        
        const data = "refresh_token=" + inputs.refresh_token + "&grant_type=refresh_token&client_id=" + client_id + "&client_secret=" + client_secret;
        const basic_token = Buffer.from(client_id + ":" + client_secret).toString("base64");

        const config = {
          headers: {
            "Accept": "application/json",
            "Content-Type": "application/x-www-form-urlencoded",
            "Authorization": "Basic " + basic_token,
          }
        };

        const response = await context.libs.axios.post("https://drchrono.com/o/token/", data, config);

        if (response.status === 200) {
          console.log("REFRESH ACCESS TOKEN SUCCESSFULLY");
          const responseData = response.data;

          context.updateAccessTokens({
            data: {
              access_token: responseData.access_token,
              refresh_token: responseData.refresh_token,
              expires_in: responseData.expires_in,
            }
          });

          return {
            ...inputs,
            responseData: responseData,
            access_token: responseData.access_token,
            refresh_token: responseData.refresh_token,
            expires_in: responseData.expires_in,
          };
        }
      }

      return inputs;

    } catch (err) {
      console.log("ERROR:", err);
    }
  }(inputs, context));
  ```

* **OAuth URL**
  `https://drchrono.com/o/authorize/`

* **Queryparams for OAuth URL**

  ```jsx
  {
     "client_id": "", // from DrChrono API setup - remove all text after //
     "response_type": "code",
     "redirect_uri": "https://freeagent.network/integration-oauth/callback"
  }
  ```

* **OAuth Callback Code**

  ```jsx
  (async function(inputs, context){
    const client_id = ''; // from DrChrono API setup
    const client_secret = ''; // from DrChrono API setup

    const data = "code=" + inputs.request.queryParams.code + "&grant_type=authorization_code&client_id=" + client_id + "&redirect_uri=https://freeagent.network/integration-oauth/callback";
    const basicToken = "Basic " + Buffer.from(client_id + ":" + client_secret).toString("base64");

    const config = {
      headers: {
        "Accept": "application/json",
        "Content-Type": "application/x-www-form-urlencoded",
        "Authorization": basicToken,
      }
    };

    const response = await context.libs.axios.post("https://drchrono.com/o/token/", data, config);
    var responseData = response.data;

    return {
      data: responseData,
      access_token: responseData.access_token,
      refresh_token: responseData.refresh_token,
      expires_in: responseData.expires_in,
      refresh_token_expires_in: responseData.refresh_token_expires_in,
    };
  }(inputs, context));
  ```

### Final Steps

* Click `Authorize` under the new `OOTB DrChrono Integration`
  → A DrChrono login popup will appear
* After successful auth, click `Test`
  You should see logs like:

  ```jsx
  [console.log]: Is user authorized?:
  true
  [console.log]: Last authorized at:
  "2025-06-06T23:51:46.283Z"
  [console.log]: access_token is: HK8TE9KdJOffVghCYrdhmkvPPGTv76
  [console.log]: refresh_token is: 999t2V3lAMnERr2Py6L4ZEQtQFc9qn
  ...
  ```

---

## 3. Monitor Integration Logs

Go to:
🔗 [All Notes View](https://freeagent.network/entity/note_fa/view/all)

* Create and **Save a new view** with the name:
  `DrChrono Integration Log`

  > Make sure the name is **exactly** as shown.

---

## 4. Device-Employee Setup

Go to:
🔗 [Device-Employee Setup](https://freeagent.network/admin-settings/app-setup/device_employee)

* **Update** the **Singular Version of Name**
  → Prevents app name collision

---

## 5. Choice List Updates

### Patient App

* **Patient Stage**
  Set `Unique Value` to `A/I/D`
  Update as follows:

  | Value               | Unique Value |
  | ------------------- | ------------ |
  | Active              | A            |
  | Inactive            | I            |
  | Inactive - Deceased | D            |

* **Language**
  Update as follows:

  | Value | Unique Value |
  | ------- | ------------ |
  | English | eng        |
  | Spanish | spa        |
  | Blank   | blank        |

### Appointment App

* **Appointment Type**

  > Might vary across customers. Ask a dev to update as needed.

---

## 6. Reset Unique Fields

Remove all existing unique fields then add below:

**These fields are critical and must exist in each respective app with the exact names as shown below:**


| Entity      | Field            |
| ----------- | ---------------- |
| Patient     | Chart ID         |
| Appointment | Claim ID         |
| Employee    | Chrono Doctor ID |

---

## 7. Supported Field Mapping

> ⚠️ **READ CAREFULLY: The sections below are informational only. You do NOT need to configure or input anything.**
> These lists are provided **just to show you** what fields the integration currently supports. If a field exists in your app **with the same name**, it will be synced. If it doesn’t, it will be skipped.

---

### Patient Fields (Synced if app field matches the name exactly)

* Dr Chrono ID
* Chart ID
* First Name
* Middle Name
* Last Name
* Date of Birth
* Sex
* Patient SSN
* Language
* Status
* Home Phone
* Cell Phone
* Email
* Emergency Contact Name
* Emergency Contact Phone
* Emergency Contact Relation
* Disable SMS/Txt
* Primary Provider (Employee)
* Patient Flags
* Patient Flags and Descriptions
* Date of Last Appointment
* Insurance Company
* Insurance ID Number
* Insurance Group Number
* Secondary Insurance
* Secondary Insurance ID Number
* Secondary Insurance Group Number
* Responsible Party Name
* Responsible Party Relation
* Responsible Party Email
* Responsible Party Phone
* Address

---

### Appointment Fields (Synced if app field matches the name exactly)

* Patient
* Provider
* Appt Profile
* Appt Status
* Date of Service
* Reason
* Duration
* Claim ID

---

### Employee Fields (Synced if app field matches the name exactly)

* First Name
* Last Name
* Work Phone
* Personal Phone
* Chrono Doctor ID
* NPI
* Work Email

---

> ✅ **Reminder:** These fields are not settings. You do not need to copy or enter them anywhere. Just make sure your app has fields with the same names if you want them to sync.


