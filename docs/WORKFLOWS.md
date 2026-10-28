# Step-by-Step Workflows — Agent Identity Foundations (APEX009)

This document provides visual workflow diagrams (control-plane setup and data-plane runtime execution) for every step in Lab **APEX009**.

---

## 1. End-to-End Lab Execution Roadmap

```mermaid
flowchart LR
    T1["Task 1<br/>Configure Env Vars,<br/>Run lab2_prereqs.sh,<br/>Enable Agent Identity APIs"] --> T2["Task 2<br/>Create GitHub API-Key<br/>Auth Provider<br/>(github-mcp-auth-provider)"]
    T2 --> T3["Task 3<br/>Refactor github-agent,<br/>Deploy with SPIFFE Identity,<br/>Bind IAM & Test A2A"]
    T3 --> T4["Task 4<br/>Create Salesforce 2LO<br/>Auth Provider<br/>(salesforce-2lo-auth-provider)"]
    T4 --> T5["Task 5<br/>Refactor salesforce-agent,<br/>Redeploy to Cloud Run,<br/>Bind SA IAM & Test A2A"]
    T5 --> APP["Appendix<br/>Verify Telemetry &<br/>Model IAM Grants"]
```

---

## 2. Task 1 Workflow: Environment & Prerequisites Setup

```mermaid
flowchart TD
    A["Start in Cloud Shell"] --> B["Export PROJECT_ID, LOCATION=us-central1, REGION=us-central1"]
    B --> C["Resolve PROJECT_NUMBER & ORGANIZATION_ID via gcloud"]
    C --> D["Copy gs://<PROJECT_ID>-bucket/apex_labs to ~/apex_labs"]
    D --> E["Execute bash apex_labs/lab2_prereqs.sh"]
    E --> E1["Enable Core APIs (aiplatform, run, cloudbuild, artifactregistry, iam, logging, cloudtrace)"]
    E1 --> E2["Provision Vertex AI Service Agent & Grant Model/Telemetry Roles"]
    E2 --> E3["Install uv & google-agents-cli==1.0.0 to ~/.local/bin"]
    E3 --> E4["Copy github-agent & salesforce-agent from GCS bucket to ~/agents"]
    E4 --> E5["Deploy starter salesforce-agent to Cloud Run (--max-instances 4)"]
    E5 --> F["Write ~/agents/github-agent/.env with GITHUB_AUTH_PROVIDER_URI"]
    F --> G["Verify agents-cli info (v1.0.0)"]
    G --> H["Enable Agent Identity APIs:<br/>agentidentity, agentidentitycredentials, sts, iamcredentials"]
```

---

## 3. Task 2 Workflow: Creating the GitHub API-Key Auth Provider

```mermaid
sequenceDiagram
    autonumber
    actor User as Operator
    participant Shell as Cloud Shell Env
    participant GCloud as gcloud alpha agent-identity
    participant AM as Auth Manager API<br/>(agentidentity.googleapis.com)

    User->>Shell: read -rs GITHUB_PAT (stores PAT in memory only)
    User->>GCloud: auth-providers create github-mcp-auth-provider<br/>--project=$PROJECT_ID --location=$LOCATION --api-key=$GITHUB_PAT
    GCloud->>AM: POST /v1alpha/projects/{P}/locations/{L}/authProviders?authProviderId=github-mcp-auth-provider
    AM-->>GCloud: Long-Running Operation -> Completed
    User->>GCloud: auth-providers list --project=$PROJECT_ID --location=$LOCATION
    GCloud->>AM: GET /v1alpha/projects/{P}/locations/{L}/authProviders
    AM-->>User: name: .../authProviders/github-mcp-auth-provider, state: ENABLED
    User->>Shell: export GITHUB_AUTH_PROVIDER_URI="projects/.../authProviders/github-mcp-auth-provider"
```

---

## 4. Task 3 Workflow: Refactoring & Deploying GitHub Agent with SPIFFE Identity

### 4.1 Deployment & IAM Binding Workflow
```mermaid
flowchart TD
    A["Edit ~/agents/github-agent/agent/agent.py"] --> A1["TODO 1: CredentialManager.register_auth_provider(GcpAuthProvider())"]
    A1 --> A2["TODO 2: github_auth_scheme = GcpAuthProviderScheme(name=GITHUB_AUTH_PROVIDER_URI)"]
    A2 --> A3["TODO 3: toolset = McpToolset(..., auth_scheme=github_auth_scheme, header_provider=github_header_provider)"]
    A3 --> B["Run deploy_with_identity.sh (DISPLAY_NAME=github-agent-v2)"]
    B --> B1["Step 1: POST /v1beta1/.../reasoningEngines<br/>with spec.identityType='AGENT_IDENTITY' only"]
    B1 --> B2["Poll LRO -> Obtain ENGINE_ID & EFFECTIVE_IDENTITY (SPIFFE)"]
    B2 --> B3["Step 2: Grant project IAM roles (aiplatform.user, logging.logWriter, etc.)<br/>to principal://<EFFECTIVE_IDENTITY>"]
    B3 --> B4["Step 3: Package source tarball & PATCH /reasoningEngines/<ENGINE_ID><br/>(excluding identityType from updateMask)"]
    B4 --> C["Export GITHUB_AGENT_IDENTITY, GITHUB_ENGINE_ID, GITHUB_AGENT_URL"]
    C --> D["Bind roles/agentidentity.user on github-mcp-auth-provider<br/>to GITHUB_AGENT_IDENTITY & user:<EMAIL>"]
    D --> E["Test with agents-cli run --url $GITHUB_AGENT_URL --mode a2a --app-name agent"]
```

### 4.2 Runtime MCP Credential Resolution State Machine
```mermaid
stateDiagram-v2
    [*] --> A2ARequestReceived : User sends prompt via agents-cli (mode: a2a)
    A2ARequestReceived --> ModelReasoning : ADK Runner invokes Gemini 3.5 Flash
    ModelReasoning --> ToolCallTriggered : Gemini selects MCP tool (e.g., search_repositories)
    ToolCallTriggered --> ResolveAuthScheme : McpToolset inspects github_auth_scheme
    ResolveAuthScheme --> ExchangeSPIFFE : GcpAuthProvider presents SPIFFE workload identity to Auth Manager
    ExchangeSPIFFE --> CredentialStored : Auth Manager validates roles/agentidentity.user & returns API key
    CredentialStored --> HeaderProviderCallback : McpToolset invokes github_header_provider(readonly_ctx)
    HeaderProviderCallback --> MCPHttpCall : Attaches {"Authorization": "Bearer <token>"} header
    MCPHttpCall --> ModelSynthesis : GitHub Copilot MCP Server returns JSON payload
    ModelSynthesis --> [*] : Final response streamed over A2A
```

---

## 5. Task 4 Workflow: Creating the Salesforce 2-Legged OAuth Auth Provider

```mermaid
sequenceDiagram
    autonumber
    actor User as Operator
    participant Shell as Cloud Shell Env
    participant GCloud as gcloud alpha agent-identity
    participant AM as Auth Manager API<br/>(agentidentity.googleapis.com)

    User->>Shell: Export SF_DOMAIN, SF_CLIENT_ID & read -rs SF_CLIENT_SECRET
    User->>GCloud: auth-providers create salesforce-2lo-auth-provider<br/>--two-legged-oauth-client-id=$SF_CLIENT_ID<br/>--two-legged-oauth-client-secret=$SF_CLIENT_SECRET<br/>--two-legged-oauth-token-url=https://$SF_DOMAIN/services/oauth2/token
    GCloud->>AM: POST /v1alpha/projects/{P}/locations/{L}/authProviders?authProviderId=salesforce-2lo-auth-provider
    AM-->>GCloud: Created authProvider [salesforce-2lo-auth-provider]
    User->>GCloud: auth-providers list --project=$PROJECT_ID --location=$LOCATION
    AM-->>User: Both github-mcp-auth-provider & salesforce-2lo-auth-provider state: ENABLED
    User->>Shell: export SALESFORCE_AUTH_PROVIDER_URI="projects/.../authProviders/salesforce-2lo-auth-provider"
```

---

## 6. Task 5 Workflow: Refactoring & Redeploying Salesforce Agent on Cloud Run

### 6.1 Code Refactoring & Cloud Run Deployment Workflow
```mermaid
flowchart TD
    A["Refactor ~/agents/salesforce-agent"] --> A1["app/tools.py:<br/>TODO 1: Register GcpAuthProvider & build salesforce_auth_config<br/>TODO 2: Add credential: AuthCredential param & _extract_token(credential)<br/>TODO 3: Return AuthenticatedFunctionTool"]
    A1 --> A2["app/agent.py:<br/>Build credential-free SalesforceClient(domain=SALESFORCE_DOMAIN)"]
    A2 --> A3["app/salesforce_client.py:<br/>Remove client_id/secret & authenticate(); derive self.instance_url from domain"]
    A3 --> A4[".env:<br/>Remove SALESFORCE_CLIENT_ID/SECRET; add SALESFORCE_AUTH_PROVIDER_URI"]
    A4 --> A5["pyproject.toml & uv.lock:<br/>Add google-adk[gcp,agent-identity] & run uv lock"]
    A5 --> B["Redeploy to Cloud Run via agents-cli deploy --max-instances 4"]
    B --> C["Resolve Cloud Run Service Account (SF_SA)"]
    C --> D["Bind roles/agentidentity.user on salesforce-2lo-auth-provider<br/>to serviceAccount:$SF_SA & user:<EMAIL>"]
    D --> E["Disable IAP (--no-iap) & test A2A query with agents-cli run"]
```

### 6.2 Runtime 2LO Function Tool Credential Injection Flow
```mermaid
sequenceDiagram
    autonumber
    participant Caller as agents-cli (A2A Client)
    participant Agent as salesforce_agent (Cloud Run)
    participant AFT as AuthenticatedFunctionTool<br/>(ADK Framework)
    participant AM as Agent Identity Auth Manager
    participant SFToken as Salesforce OAuth2 Endpoint
    participant ToolFn as search_salesforce(query, credential)
    participant SFClient as SalesforceClient (SOSL/REST)

    Caller->>Agent: "Search for any architecture decision records about timeouts."
    Agent->>AFT: Model invokes search_salesforce(query="architecture decision records timeouts")
    AFT->>AM: GcpAuthProvider requests credential for salesforce-2lo-auth-provider<br/>(authenticated as Cloud Run serviceAccount:$SF_SA)
    alt Token not cached or expired in Auth Manager
        AM->>SFToken: POST /services/oauth2/token (grant_type=client_credentials)
        SFToken-->>AM: 200 OK { "access_token": "..." }
    end
    AM-->>AFT: Returns AuthCredential(oauth2.access_token)
    AFT->>ToolFn: Invokes search_salesforce(query=..., credential=AuthCredential)
    ToolFn->>ToolFn: access_token = _extract_token(credential)
    ToolFn->>SFClient: client.search_files(access_token, query)
    SFClient->>SFClient: GET https://<SF_DOMAIN>/services/data/v62.0/search?q=FIND...
    SFClient-->>ToolFn: List of matching documents + excerpts
    ToolFn-->>Agent: {"status": "success", "data": documents}
    Agent-->>Caller: Final answer citing matching Salesforce Files
```
