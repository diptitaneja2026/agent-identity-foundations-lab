# Operational Runbook — Agent Identity Foundations (APEX009)

> **Lab Link:** [https://explore.qwiklabs.com/classrooms/21504/labs/115874](https://explore.qwiklabs.com/classrooms/21504/labs/115874)

---

## Task 1: Set Variables, Download Starter Code & Enable APIs

### 1.1 Set Project, Location & Organization Variables
```bash
gcloud auth login
export PROJECT_ID=qwiklabs-gcp-01-ea27a66f2dee
gcloud auth application-default login

export LOCATION=us-central1
export REGION=us-central1
gcloud config set project ${PROJECT_ID}
export PROJECT_NUMBER=$(gcloud projects describe ${PROJECT_ID} --format='value(projectNumber)')
export ORGANIZATION_ID=$(gcloud projects get-ancestors ${PROJECT_ID} --format=json | jq -r '.[] | select(.type=="organization") | .id')
echo "Project: ${PROJECT_ID} (${PROJECT_NUMBER}), Org: ${ORGANIZATION_ID}, Location: ${LOCATION}, Agents: ${REGION}"
```

**Expected Output:**
```text
Project: qwiklabs-gcp-01-ea27a66f2dee (423033817941), Org: 616463121992, Location: us-central1, Agents: us-central1
```

### 1.2 Download & Run Lab Setup Script
```bash
gcloud storage cp -r gs://${PROJECT_ID}-bucket/apex_labs .
bash apex_labs/lab2_prereqs.sh
export PATH="$HOME/.local/bin:$PATH"
```

### 1.3 Create `.env` for `github-agent` & Verify `agents-cli`
```bash
cd ~/agents/github-agent
cat << EOF > .env
GOOGLE_CLOUD_PROJECT=$PROJECT_ID
GOOGLE_CLOUD_LOCATION=global
GOOGLE_GENAI_USE_VERTEXAI=True
MODEL=gemini-3.5-flash
GITHUB_MCP_URL=https://api.githubcopilot.com/mcp/
GITHUB_AUTH_PROVIDER_URI=projects/$PROJECT_ID/locations/$LOCATION/authProviders/github-mcp-auth-provider
EOF

agents-cli info
```

**Expected Output:**
```text
CLI version:        1.0.0
```

### 1.4 Enable the Agent Identity APIs
```bash
gcloud services enable \
  iam.googleapis.com \
  agentidentity.googleapis.com \
  aiplatform.googleapis.com \
  agentidentitycredentials.googleapis.com \
  iamcredentials.googleapis.com \
  sts.googleapis.com \
  --project=${PROJECT_ID}
```

---

## Task 2: Create the GitHub API-Key Auth Provider

### 2.1 Store GitHub Personal Access Token & Create Auth Provider
```bash
read -rs GITHUB_PAT && echo
# Paste your GitHub PAT and press Enter

gcloud alpha agent-identity auth-providers create github-mcp-auth-provider \
  --project=${PROJECT_ID} \
  --location=${LOCATION} \
  --api-key="${GITHUB_PAT}"
```

### 2.2 Verify Auth Provider & Save Resource URI
```bash
gcloud alpha agent-identity auth-providers list \
  --project=${PROJECT_ID} \
  --location=${LOCATION}

export GITHUB_AUTH_PROVIDER_URI="projects/${PROJECT_ID}/locations/${LOCATION}/authProviders/github-mcp-auth-provider"
echo ${GITHUB_AUTH_PROVIDER_URI}
```

**Expected Output:**
```yaml
---
authProviderTypeParams:
  apiKey: {}
createTime: '2026-10-28T08:31:00.816327Z'
name: projects/qwiklabs-gcp-01-ea27a66f2dee/locations/us-central1/authProviders/github-mcp-auth-provider
state: ENABLED
updateTime: '2026-10-28T08:31:00.816327Z'
projects/qwiklabs-gcp-01-ea27a66f2dee/locations/us-central1/authProviders/github-mcp-auth-provider
```

---

## Task 3: Move the GitHub Agent onto the Auth Provider

### 3.1 Update `~/agents/github-agent/agent/agent.py`
```bash
cat << 'EOF' > ~/agents/github-agent/agent/agent.py
import os

from dotenv import load_dotenv
from google.adk.agents import Agent
from google.adk.apps import App
from google.adk.auth.credential_manager import CredentialManager
from google.adk.integrations.agent_identity import GcpAuthProvider, GcpAuthProviderScheme
from google.adk.models import Gemini
from google.adk.tools.mcp_tool import McpToolset, StreamableHTTPConnectionParams
from google.genai import types

load_dotenv()

MODEL = os.environ["MODEL"]
GITHUB_MCP_URL = os.getenv("GITHUB_MCP_URL", "https://api.githubcopilot.com/mcp/")
# Full resource name of the API-key auth provider you created in Task 2.
GITHUB_AUTH_PROVIDER_URI = os.environ["GITHUB_AUTH_PROVIDER_URI"]

# Only expose the read tools this agent needs.
GITHUB_TOOL_FILTER = ["search_repositories", "search_issues", "list_issues"]

# TODO 1: register the Google Cloud auth provider so the
# CredentialManager can resolve the auth provider credential using
# the agent's own identity. This runs once at import.
CredentialManager.register_auth_provider(GcpAuthProvider())

# TODO 2: create the auth scheme that names which auth provider the ADK
# should resolve before the tool runs, e.g.
#     github_auth_scheme = GcpAuthProviderScheme(name=GITHUB_AUTH_PROVIDER_URI)
github_auth_scheme = GcpAuthProviderScheme(name=GITHUB_AUTH_PROVIDER_URI)


def github_mcp_toolset() -> McpToolset:
    """MCP toolset for the GitHub server, authenticated via auth manager."""
    toolset = None

    def github_header_provider(readonly_ctx) -> dict[str, str]:
        if not toolset or not toolset.get_auth_config():
            return {}
        auth_config = toolset.get_auth_config()
        cred = None
        if readonly_ctx and hasattr(readonly_ctx, "get_credential"):
            cred = readonly_ctx.get_credential(auth_config.credential_key)
        if not cred:
            cred = auth_config.exchanged_auth_credential
        if not cred or not cred.http:
            return {}
        token = None
        if cred.http.credentials and cred.http.credentials.token:
            token = cred.http.credentials.token
        elif cred.http.additional_headers:
            token = cred.http.additional_headers.get(
                "X-API-Key"
            ) or cred.http.additional_headers.get("X-GOOG-API-KEY")
        if token:
            return {"Authorization": f"Bearer {token}"}
        return {}

    # TODO 3: Build and return the McpToolset as github_mcp_toolset.
    toolset = McpToolset(
        connection_params=StreamableHTTPConnectionParams(url=GITHUB_MCP_URL),
        auth_scheme=github_auth_scheme,
        tool_filter=GITHUB_TOOL_FILTER,
        header_provider=github_header_provider,
    )
    return toolset


root_agent = Agent(
    name="github_agent",
    model=Gemini(
        model=MODEL,
        retry_options=types.HttpRetryOptions(attempts=3),
    ),
    description=(
        "Specialized assistant for GitHub: searches repositories, issues, and "
        "pull requests via the GitHub MCP server."
    ),
    instruction=(
        "You are a specialized assistant for interacting with GitHub. "
        "Use the provided tools to search for repositories, find issues, and "
        "retrieve pull-request information, then answer with what you find. "
        "Be concise and cite the repository names / issue numbers you used."
    ),
    tools=[github_mcp_toolset()],
)

app = App(root_agent=root_agent, name="app")
EOF
```

### 3.2 Deploy to Agent Runtime with Agent Identity Enabled
```bash
cd ~/agents/github-agent
chmod 777 ./deploy_with_identity.sh
REGION=us-central1 A2A_APP_NAME=agent AUTH_PROVIDER_NAME=github-mcp-auth-provider DISPLAY_NAME=github-agent-v2 ./deploy_with_identity.sh
```

### 3.3 Export Effective Identity & Reasoning Engine URL
```bash
export GITHUB_ENGINE_ID="5834468567717773312" # Replace with Reasoning Engine ID from deploy output
export GITHUB_AGENT_IDENTITY="principal://agents.global.org-${ORGANIZATION_ID}.system.id.goog/resources/aiplatform/projects/${PROJECT_NUMBER}/locations/${REGION}/reasoningEngines/${GITHUB_ENGINE_ID}"
export GITHUB_AGENT_URL="https://${REGION}-aiplatform.googleapis.com/v1/projects/${PROJECT_NUMBER}/locations/${REGION}/reasoningEngines/${GITHUB_ENGINE_ID}"
echo ${GITHUB_AGENT_URL}
```

### 3.4 Grant IAM Access to the Auth Provider
```bash
gcloud alpha agent-identity auth-providers add-iam-policy-binding github-mcp-auth-provider \
  --project=${PROJECT_ID} \
  --location=${LOCATION} \
  --role="roles/agentidentity.user" \
  --member="${GITHUB_AGENT_IDENTITY}"

gcloud alpha agent-identity auth-providers add-iam-policy-binding github-mcp-auth-provider \
  --project=${PROJECT_ID} \
  --location=${LOCATION} \
  --role="roles/agentidentity.user" \
  --member="user:$(gcloud config get-value account)"
```

### 3.5 Test the Deployed GitHub Agent
```bash
agents-cli run --url ${GITHUB_AGENT_URL} --mode a2a --app-name agent "List all my open GitHub issues"
agents-cli run --url ${GITHUB_AGENT_URL} --mode a2a --app-name agent "Find the google/adk-python repository"
```

---

## Task 4: Create the Salesforce 2LO Auth Provider

```bash
export SF_DOMAIN="YOUR_SALESFORCE_MY_DOMAIN"   # e.g. yourorg.my.salesforce.com
export SF_CLIENT_ID="YOUR_CONSUMER_KEY"
read -rs SF_CLIENT_SECRET && echo

gcloud alpha agent-identity auth-providers create salesforce-2lo-auth-provider \
  --project=${PROJECT_ID} \
  --location=${LOCATION} \
  --two-legged-oauth-client-id="${SF_CLIENT_ID}" \
  --two-legged-oauth-client-secret="${SF_CLIENT_SECRET}" \
  --two-legged-oauth-token-url="https://${SF_DOMAIN}/services/oauth2/token"

gcloud alpha agent-identity auth-providers list \
  --project=${PROJECT_ID} \
  --location=${LOCATION}

export SALESFORCE_AUTH_PROVIDER_URI="projects/${PROJECT_ID}/locations/${LOCATION}/authProviders/salesforce-2lo-auth-provider"
echo ${SALESFORCE_AUTH_PROVIDER_URI}
```

---

## Task 5: Update the Salesforce Agent to Use the Auth Provider

### 5.1 Update `~/agents/salesforce-agent/app/tools.py`
See [`src/salesforce-agent/app/tools.py`](../src/salesforce-agent/app/tools.py).

### 5.2 Update `~/agents/salesforce-agent/app/agent.py`
See [`src/salesforce-agent/app/agent.py`](../src/salesforce-agent/app/agent.py).

### 5.3 Update `~/agents/salesforce-agent/app/salesforce_client.py`
See [`src/salesforce-agent/app/salesforce_client.py`](../src/salesforce-agent/app/salesforce_client.py).

### 5.4 Update `~/agents/salesforce-agent/.env`
```bash
cat << EOF > ~/agents/salesforce-agent/.env
GOOGLE_CLOUD_PROJECT=${PROJECT_ID}
GOOGLE_CLOUD_LOCATION=global
GOOGLE_GENAI_USE_VERTEXAI=True
MODEL=gemini-3.5-flash
SALESFORCE_DOMAIN=${SF_DOMAIN}
SALESFORCE_AUTH_PROVIDER_URI=${SALESFORCE_AUTH_PROVIDER_URI}
EOF
```

### 5.5 Update Dependencies & Redeploy to Cloud Run
```bash
cd ~/agents/salesforce-agent
sed -i 's/google-adk\[gcp\]/google-adk[gcp,agent-identity]/' pyproject.toml
uv lock

agents-cli deploy --project ${PROJECT_ID} --region ${REGION} \
  --no-confirm-project \
  --max-instances 4 \
  --update-env-vars "$(grep -v '^#' .env | grep -v '^$' | grep -v '^GOOGLE_CLOUD_PROJECT' | paste -sd,)"
```

### 5.6 Grant Service Account Access to `salesforce-2lo-auth-provider`
```bash
export SF_SA=$(gcloud run services describe salesforce-agent --region ${REGION} \
  --format='value(spec.template.spec.serviceAccountName)')
echo ${SF_SA}

gcloud alpha agent-identity auth-providers add-iam-policy-binding salesforce-2lo-auth-provider \
  --project=${PROJECT_ID} \
  --location=${LOCATION} \
  --role="roles/agentidentity.user" \
  --member="serviceAccount:${SF_SA}"

gcloud alpha agent-identity auth-providers add-iam-policy-binding salesforce-2lo-auth-provider \
  --project=${PROJECT_ID} \
  --location=${LOCATION} \
  --role="roles/agentidentity.user" \
  --member="user:$(gcloud config get-value account)"
```

### 5.7 Disable IAP & Test over A2A
```bash
gcloud run services update salesforce-agent --region=${REGION} --no-iap
export SF_URL=$(gcloud run services describe salesforce-agent --region ${REGION} --format='value(status.url)')
agents-cli run --url ${SF_URL} --mode a2a "Search for any architecture decision records about timeouts."
```

---

## Appendix: Verify Supporting IAM Grants

```bash
# 1. Verify GitHub Agent Identity project roles
gcloud projects get-iam-policy ${PROJECT_ID} \
  --flatten="bindings[].members" \
  --filter="bindings.members:${GITHUB_AGENT_IDENTITY}" \
  --format="value(bindings.role)"

# 2. Verify Salesforce Cloud Run Service Account project roles
gcloud projects get-iam-policy ${PROJECT_ID} \
  --flatten="bindings[].members" \
  --filter="bindings.members:serviceAccount:${SF_SA}" \
  --format="value(bindings.role)"
```
