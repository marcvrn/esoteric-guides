# Connect Codex to Google Ads through MCP

This tutorial connects a local Codex session to a Google Ads account. At the end, Codex can list accounts and read campaign data through Google's MCP server. The setup runs on your computer and does not require changes to an application or a deployed service.

We verified this path on 2026-11-02 with a Google Cloud project and a Google Ads account with production campaigns. Google's [MCP integration guide](https://developers.google.com/google-ads/api/docs/developer-toolkit/mcp-server) is the upstream reference.

## Before you start

You need:

- A Google account with access to the intended Google Ads account. Read-only Ads access is enough for reporting.
- Editor or equivalent access to your Google Cloud project, plus someone who can manage its OAuth consent screen and API access level.
- `gcloud`, `pipx`, Codex, and Python 3.10 or newer on the local machine.

Check the tools before changing Google settings:

```sh
command -v gcloud
command -v pipx
command -v codex
python3 --version
```

On the Mac used for this setup, `pipx` itself was installed under `~/Library/Python/3.9/bin`, but its default interpreter was Python 3.9. The Google Ads MCP package requires Python 3.10 or newer. We passed `--python /opt/homebrew/bin/python3`, which was Python 3.13, to every `pipx run` command. The path to the `pipx` executable does not determine the Python version that runs the package.

Keep OAuth client JSON files, Application Default Credentials (ADC), and access tokens out of the repository and chat. The commands below store ADC at `~/.config/gcloud/application_default_credentials.json`.

## 1. Enable the Google Ads API in the Cloud project

Set your Google Cloud project ID and enable the API:

```sh
ADS_PROJECT_ID='your-project-id'
gcloud services enable googleads.googleapis.com --project "$ADS_PROJECT_ID"
gcloud services list --enabled --project "$ADS_PROJECT_ID" \
  --filter='config.name:googleads.googleapis.com' \
  --format='value(config.name)'
```

The second command should print `googleads.googleapis.com`. Enabling the API grants the project **Test** access. Test access does not permit queries against a production Ads account. Step 5 upgrades it to Explorer.

Google [stopped requiring developer tokens for the current MCP setup](https://developers.google.com/google-ads/api/docs/developer-toolkit/mcp-server) in September 2026. Older examples in the MCP repository still mention `GOOGLE_ADS_DEVELOPER_TOKEN`. Follow the current integration guide and do not add a token to this setup.

## 2. Set up OAuth consent and a Desktop client

Open [Google Auth Platform](https://console.cloud.google.com/auth/overview). Select the same Cloud project that you used in step 1. Keep it selected throughout these steps.

1. In **Audience**, use an External app if the Ads user is a personal Google account. While the app is in Testing, add the exact Google account that will authorize the MCP server under **Test users**.
2. In **Data Access**, add `https://www.googleapis.com/auth/adwords`. Google may show the short form `.../auth/adwords`.
3. In **Clients**, create an OAuth client with application type **Desktop app**. Download its JSON file to a local location outside the repository, such as Downloads.

Use the Desktop client's JSON from the same Cloud project. Confirm that its client ID matches the Desktop entry in Cloud Console. Several files named `client_secret_*.json` can coexist in Downloads, and their filenames do not tell you which client type is active. In our test, two older downloads produced a loopback-flow block or a generic HTTP 400. A fresh Desktop client with its client secret present completed authorization.

Google's [OAuth client instructions](https://docs.cloud.google.com/mcp/set-up-authentication-mcp-servers#desktop) describe the Desktop app choice. The [Google Ads MCP repository](https://github.com/googleads/google-ads-mcp#option-2-configure-credentials-using-application-default-credentials) requires the Ads scope for ADC.

## 3. Authorize the Google Ads user locally

Set the path to the JSON you just downloaded, then run the login. Replace the example email with the Google account that can see the Ads account at [ads.google.com](https://ads.google.com/).

```sh
ADS_GOOGLE_ACCOUNT='your-ads-user@example.com'
ADS_CLIENT_JSON="$HOME/Downloads/client_secret_YOUR_DESKTOP_CLIENT.json"

gcloud auth application-default login "$ADS_GOOGLE_ACCOUNT" \
  --client-id-file="$ADS_CLIENT_JSON" \
  --scopes=https://www.googleapis.com/auth/adwords,https://www.googleapis.com/auth/cloud-platform,https://www.googleapis.com/auth/userinfo.email
```

The browser should open a Google consent page. Choose the same Google account you named in `ADS_GOOGLE_ACCOUNT` and approve the requested scopes. Leave the terminal command running until the browser returns to its local callback. Success prints `Credentials saved to file: [...]`.

`gcloud` requires the Cloud Platform scope when this command specifies scopes. The `userinfo.email` scope lets you verify the ADC identity. Check both the token and the account identity without printing the token:

```sh
gcloud auth application-default print-access-token >/dev/null && echo 'ADC works'

python3 - <<'PY'
import json
import subprocess
import urllib.request

token = subprocess.check_output(
    ['gcloud', 'auth', 'application-default', 'print-access-token'],
    text=True,
).strip()
request = urllib.request.Request(
    'https://www.googleapis.com/oauth2/v3/userinfo',
    headers={'Authorization': f'Bearer {token}'},
)
with urllib.request.urlopen(request, timeout=15) as response:
    print(json.load(response)['email'])
PY
```

The printed email must be the Google Ads user. `gcloud auth list` shows the CLI's active account, which is not proof of the account stored in ADC.

If ADC already exists but token refresh fails, replace it with this login. Our old ADC file impersonated a service account that no longer existed. Google returned `Gaia id not found`, even though the JSON file was present.

## 4. Give that Google account access to Ads

Signing into the Cloud project and adding an OAuth test user does not grant Google Ads account access. Check the Google Ads UI while signed in as the same account. If it cannot see the intended account, invite it from an Ads account that has administrator access:

1. Open **Admin → Access and security → Users** in Google Ads.
2. Select **+**, enter the exact email address, and choose **Read-only** for reporting.
3. Send the invitation. Open the invitation in the invited Google account and accept it.

If the campaigns belong to a manager account, invite the user at the manager level when that is the intended scope. Google's [Ads access guide](https://support.google.com/google-ads/answer/6372672) and [manager access guide](https://support.google.com/google-ads/answer/9977851) cover both cases.

If Ads says the email domain is not allowed, an Ads administrator can open **Admin → Access and security → Security → Allowed domains** and add the required domain. A manager account can enforce this setting on its child accounts. Adding `gmail.com` allows future invitations to any Gmail address, although each user still needs a separate invitation. You can avoid widening the allowed-domain list by authorizing the MCP server with an existing Ads user instead. See Google's [allowed domains guide](https://support.google.com/google-ads/answer/12864084) and [manager security guide](https://support.google.com/google-ads/answer/12865295).

## 5. Apply for Explorer API access

Open the actual [Google Ads API Overview](https://console.cloud.google.com/google/ads-apis/overview). Confirm that your Cloud project is selected. If the current level is **Test**, expand **Upgrade access level** and apply for **Explorer**. Google may approve the application automatically.

Use this path, `/google/ads-apis/overview`. The generic Cloud API details page at `/apis/api/googleads.googleapis.com/overview` shows whether the service is enabled, but it is not the Explorer application page. We lost time on that distinction during setup.

Without Explorer or higher, account discovery can work while a production account search fails with:

```text
The Google Cloud project is only approved for use with test accounts.
To access non-test accounts, apply for Explorer, Basic or Standard access.
```

Google's [access-level guide](https://developers.google.com/google-ads/api/docs/api-policy/access-levels) explains the Test and Explorer limits.

## 6. Register the MCP server in Codex

Run this on the computer where Codex runs:

```sh
codex mcp add google-ads-mcp \
  --env GOOGLE_PROJECT_ID=your-project-id \
  --env GOOGLE_APPLICATION_CREDENTIALS="$HOME/.config/gcloud/application_default_credentials.json" \
  -- "$(command -v pipx)" run \
    --python "$(command -v python3)" \
    --spec git+https://github.com/googleads/google-ads-mcp.git \
    google-ads-mcp
```

Check the selected Python version first if `python3` does not resolve to Python 3.10 or newer. The `--python` argument is needed even when the `pipx` executable is installed correctly.

Now open `~/.codex/config.toml`. Add this allow list under `[mcp_servers.google-ads-mcp]`, next to its `command` and `args` entries:

```toml
enabled_tools = ["customers_list_accessible_customers", "metadata_get_resource_metadata", "search_search"]
```

The MCP integration guide describes a read-only server, but the repository version we installed also exposed `assets_create_asset`, `assets_upload_video_asset`, and two label-management tools. This Codex allow list exposes the three reporting tools we need. It controls what Codex can call; it does not make the underlying server process read-only for other clients. OpenAI documents [`enabled_tools` in the Codex configuration reference](https://developers.openai.com/codex/config-reference).

If account access goes through a manager account, you can also set `GOOGLE_ADS_LOGIN_CUSTOMER_ID` or pass `login_customer_id` to a search tool call. Do not set it for a directly accessible client account without a reason. Google's [MCP guide](https://developers.google.com/google-ads/api/docs/developer-toolkit/mcp-server) documents this option.

## 7. Check the connection

Check the saved Codex entry:

```sh
codex mcp get google-ads-mcp
```

The output should show `enabled: true`, `transport: stdio`, the three `enabled_tools`, and the two environment variable names. Codex masks their values in this output. Start a new Codex session so it loads the new MCP configuration.

Ask Codex to list accessible Google Ads customers. The `customers_list_accessible_customers` tool returns IDs as plain strings in `structuredContent.result`. Do not assume each string starts with `customers/`. Our first test checker made that assumption and falsely reported zero accounts.

Then ask for a read-only query, such as the account status or campaigns in the last 30 days. A working `search_search` call takes a customer ID, a resource such as `customer` or `campaign`, and a list of fields. Our final setup returned one directly accessible account and a successful `customer.id` search. That search proves that both the Ads user and the Cloud project can access the production account.

## Troubleshooting

| Symptom | Check and fix |
| --- | --- |
| `pipx` says the package requires Python `>=3.10` | Pass `--python` with a Python 3.10 or newer path. On our Mac, `pipx` defaulted to Python 3.9 while Homebrew had Python 3.13. |
| An ADC JSON file exists, but token refresh fails | Run `gcloud auth application-default print-access-token >/dev/null` to check refresh without printing the token. Replace stale ADC with the Desktop client login in step 3. |
| Google says the loopback flow is blocked | Confirm that the downloaded client ID is the **Desktop app** client shown in the same Cloud project. We saw this error with the wrong downloaded client file. |
| Google shows a generic HTTP 400 during consent | Check the exact client ID, the OAuth test user, and the Ads scope. A fresh Desktop client resolved this in our setup; the old client's precise failure cause was not confirmed. |
| Consent finishes in the browser, but the terminal still waits | Keep the local callback command running until Google redirects to it. A stopped command cannot save the credential. |
| Account discovery returns an empty list | Verify the ADC email with step 3, accept the Ads invitation, and check that the same user sees the account in the Ads UI. Read `structuredContent.result` as plain customer IDs. |
| Account discovery works, but search rejects a production account | Check the Cloud project's access level on the **Google Ads API Overview** page. Apply for Explorer or higher. |
| Ads rejects an invitation because the email domain is not allowed | Check **Security → Allowed domains** on the Ads account or its manager account. Consider authorizing with an existing Ads user before widening the domain list. |
| The server is configured but its tools are absent in Codex | Run `codex mcp get google-ads-mcp`, then start a new Codex session. The current session may keep its earlier tool catalog. |

This setup uses Google's local `stdio` transport. It does not deploy a Cloud Run service or expose an HTTP endpoint.
