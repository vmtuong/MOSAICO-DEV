# Setting up "MOSAICO - Langfuse Repository", by University of York (UoY)

This is the fork of [MOSAICO-DEV](https://github.com/MDEGroup/MOSAICO-DEV) repository that contains code and tests that interact with a Langfuse instance (self-hosted).

This setup notes by UoY team focus on setting up WP2's repository, testing, and integrating with WP1's reference agent.

## 1. Prerequisites

- Java 21 (or the JDK configured for the project)
- Maven (the project includes the Maven wrapper `mvnw`)
- Docker & Docker Compose (required to run Langfuse locally via the provided compose file)

If you prefer to use the included scripts, make sure they have execute permission:

```bash
chmod +x start-langfuse.sh run-integration-tests.sh
```

## 2. Start Langfuse (local, self-hosted) and many related services (MCP server, Ollama, Postgre database, etc.)

Using the provided script. (Alternatively, you can use [one line command](#start-docker-manually-instead-of-using-start-langfusesh-script-note-using-the-env-file-is-critical) in the terminal.)

```bash
./start-langfuse.sh
```

Verify Langfuse is reachable:

```bash
curl -I http://localhost:3000
```

## 3. Langfuse account, project setup, and API keys
- Go to Langfuse http://localhost:3000
- Register the admin account with details in .env.langfuse (note that my version of langfuse required a special character in the password)
- Create an organisation, a project within that, and generate API key for the project
- Copy Langfuse public and private keys into `.env.langfuse`, `application.properties`, `application-dev.properties`, `LangfuseServiceIntegrationTest.java`, and `llm-usecase/config.bench.json`

## 5. Run unit tests

To run all unit tests:

```bash
./mvnw test
```

Unit tests are fast and do not require a running Langfuse instance (they use mocks in test resources).

## 6. Langfuse Integration Test DOES NOT require for integration between WP1 and WP2

This is the placeholder for instructions to set up Langfuse Integration Test.

## 7. Rebuild the Docker image

During development, the image may not be up do date. Please rebuild mosaico-app ([instructions below](#rebuild-and-relaunch-mosaico-app-docker)).

## 8. Add and search agent

You can add and search for agents via [Swagger UI](#swagger-ui-link) or using the following commands in the terminal.

### Add a generic software engineering agent via `POST api/agents`

```bash
curl -X POST http://localhost:8080/api/agents \
  -H 'Content-Type: application/json' \
  -d '{
    "name": "aisp-mini-swe-agent",
    "description": "Agent who solves generic software engineering issues using a bash-only tool, including write Python script, explain code, solve coding challenges.",
    "version": "1.0.0",
    "role": "generic software engineering solver",
    "objective": "solves generic software engineering issues",
    "a2aAgentCardUrl": "http://localhost:20000/.well-known/agent-card.json",
    "deployment": {
      "mode": "ENDPOINT"
    }
  }'
```


### Add a metamodel generation agent via `POST api/agents`

```bash
curl -X POST http://localhost:8080/api/agents \
  -H 'Content-Type: application/json' \
  -d '{
    "name": "metamodel-generation-collaboration-agent",
    "description": "Collaboration agent who should handle a query for generating a metamodel.",
    "version": "1.0.0",
    "role": "collaboration agent",
    "objective": "coordinate solution agent and supervisor agent to generate and verify metamodel based on user description.",
    "a2aAgentCardUrl": "http://localhost:12000/.well-known/agent-card.json",
    "deployment": {
      "mode": "ENDPOINT"
    }
  }'
```


### Search agent by name via `GET api/agents/search/name`

```bash
curl -X 'GET' \
  'http://localhost:8080/api/agents/search/name?name=aisp-mini-swe-agent' \
  -H 'accept: */*'
```

### Search agent by role via `GET api/agents/search/role`

```bash
curl -X 'GET' \
  'http://localhost:8080/api/agents/search/role?role=generic%20software%20engineering%20solver' \
  -H 'accept: */*'
```

## 9. Inspect MCP Server

### Connect to MCP Server via MCP Inspector

Run the MCP inspector in the terminal.
```bash
npx @modelcontextprotocol/inspector
```

Navigate to `http://localhost:6274` to use the MCP inspector.

The MCP server is running and accessible using the following settings:

- **Transport Type:** `Streamable HTTP`
- **URL:** `http://localhost:8080/mcp`
- **Configuration  - Proxy Session Token** Copy and paste the session token from the terminal. You can also set ```export DANGEROUSLY_OMIT_AUTH=true``` to disable authentication.

### Available MCP Resources and Tools

- **Resources:**
  - `document/agents` - List all MOSAICO agents
  - `document/agents/{id}` - Get specific agent by ID

- **Tools:**
  - `AgentMCPTool` - Search agents by query with topK parameter

Test by using AgentMCPTool and query = "Python" and topK = 10. You should be able to see the recently-added aisp-mini-swe-agent.

See [AgentMCP.java](src/main/java/it/univaq/disim/mosaico/wp2/repository/mcp/AgentMCP.java) for implementation details.


## 10. Full Demostration with the AI Coding Extension

There should be two agents available in the repository (Notes: please use AgentMCPTool and search for "agent" to see the list.)

### Start metamodel generation agent
Start the metamodel generation agents from [mosaico-aisp](https://gitlab.eclipse.org/eclipse-research-labs/mosaico-project/aisp-prototyping/-/tree/main/a2a?ref_type=heads) (collaboration agent, solution agent, syntactic supervisor, and semantic supervisor).

### Start generic software engineering agent
Start the [aisp-mini-swe-agent](https://gitlab.eclipse.org/eclipse-research-labs/mosaico-project/aisp-mini-swe-agent) (mini-swe-agent with A2A wrapper).

### Start the reference agent
Start the Reference Agent.
  - Open the root folder with VS Code. In the Run and Debug window, choose "Reference Agent (Debug)", and click "Start debugging". This should create two integrated terminals for reference-agent and reference-repository.
  - Alternatively, you can run using two terminals:
    ```bash
    cd reference-agent
    npm run build
    node dist/reference_agent_executor.js
    ```

### Chatting using MOSAICO AI Coding Extension

Set up MOSAICO AI Coding Extension to chat with Reference Agent.
  - Launch the MOSAICO AI Coding Extension. Edit the **apiBase** of the chat model to `http://localhost:4000`. You should see the following in `config.yaml`
    ```yaml
    name: Local Config
    version: 1.0.0
    schema: v1
    models:
      - name: Reference agent - chat
        provider: mosaico
        model: mosaico-default
        apiBase: http://localhost:4000
    ```

Chat with Reference Agent. Try the following messages that demonstrates classification of SE and non-SE, two different agents for SE tasks, and context management when switching between SE and non-SE.
  - Hi
  - Write a Python script that prints "Hello world!"
  - What is the capital of France?
  - How about Germany?
  - Back to the Python script, change the string to "Good morning!!!"
  - Write a Python script that can sum, substract, multiply, divide two inputed numbers.
  - Please generate a metamodel of a simple library management system with two main concepts: Library and Book. Each Library has a name (String) and contains a collection of Books (containment reference). Each Book has a title (String).
  - Please generate a metamodel of a simple library management system with three main concepts: Library, Book, and Author. Each Library has a name (String) and contains a collection of Books and a collection of Authors (both containment references). Each Book has a title (String) and a page count(int) and has a non-containment reference to its main Author. Each Author has a name (String) and age (int) and knows which Books they have written (non-containment reference to Books). The system should clearly distinguish between containment and non-containment relationships: A Library owns its Books and its Authors (deleting the Library deletes its Books and its Authors). Authors and Books reference each other but do not manage each other's lifecycles.



## Appendix - Useful Notes:

### Swagger UI link

Access the API via http://localhost:8080/swagger-ui/index.html

### Rebuild and relaunch mosaico-app (Docker)

1) Rebuild image and restart service:

```bash
docker-compose -f docker-compose.langfuse.yml --env-file .env.langfuse build mosaico-app
docker-compose -f docker-compose.langfuse.yml --env-file .env.langfuse up -d mosaico-app
```

2) Check status/logs:

```bash
docker-compose -f docker-compose.langfuse.yml ps mosaico-app
docker-compose -f docker-compose.langfuse.yml logs -f mosaico-app
```

### Start Docker manually instead of using `start-langfuse.sh` script (note: using the env file is critical)

```bash
docker-compose -f docker-compose.langfuse.yml --env-file .env.langfuse up -d
```

### Database interaction via Docker Compose

Connect to the Langfuse PostgreSQL database

```bash
# Connect to the Langfuse PostgreSQL container
docker-compose -f docker-compose.langfuse.yml exec postgres psql -U langfuse -d langfuse
```

Once connected, you'll be in the PostgreSQL shell. Run this SQL to find the datasets table structure:

```sql
-- List all tables to find the datasets table
\dt

-- View the datasets table schema
\d datasets

-- Create the dataset (you'll need to insert into the datasets table with appropriate columns)
-- First check what columns exist and what data you need
SELECT * FROM datasets LIMIT 1;
```


### Stop mosaico-app (Docker), then build and run via Maven

If you started the Langfuse stack using `docker-compose.langfuse.yml`, the `mosaico-app` service is already running on port 8080. Thus, the running command is not needed. Build and run using the Maven wrapper with dev profile and on a different port 8081. You can stop this app on 8081 right after, as both 8080 and 8081 apps will connect to the same database, same Langfuse and Ollama instances.

Stop the container:

```bash
docker-compose -f docker-compose.langfuse.yml stop mosaico-app
```

Run the app with Maven on an alternate port (avoids conflict with 8080 container):

```bash
./mvnw -Dspring-boot.run.profiles=dev -Dspring-boot.run.arguments=--server.port=8081 spring-boot:run
```

If you need a fresh build before running:

```bash
./mvnw clean package -DskipTests
./mvnw -Dspring-boot.run.profiles=dev -Dspring-boot.run.arguments=--server.port=8081 spring-boot:run
```

### Update agent via id - example adding agent card url

For example, if you add an aisp-mini-swe-agent with the following details (without agent card url and deployment details):
```bash
curl -X POST http://localhost:8080/api/agents \
  -H 'Content-Type: application/json' \
  -d '{
    "name": "metamodel-generation-collaboration-agent",
    "description": "Collaboration agent who should handle a query for generating a metamodel.",
    "version": "1.0.0",
    "role": "collaboration agent",
    "objective": "coordinate solution agent and supervisor agent to generate and verify metamodel based on user description."
  }'
```


Now, to update that agent, find agent's ID using name: aisp-mini-swe-agent

```bash
curl -X GET 'http://localhost:8080/api/agents/search/name?name=aisp-mini-swe-agent' \
  -H 'accept: */*'
```

Using the ID found above, update agent's fields (for example, adding a2aAgentCardUrl)

```bash
curl -X PUT http://localhost:8080/api/agents/{AGENT-ID-FROM-ABOVE} \
  -H 'Content-Type: application/json' \
  -d '{
    "name": "aisp-mini-swe-agent",
    "description": "Agent who solves generic software engineering issues using a bash-only tool, including write Python script, explain code, solve coding challenges.",
    "version": "1.0.0",
    "role": "generic software engineering solver",
    "objective": "solves generic software engineering issues",
    "a2aAgentCardUrl": "http://localhost:20000/.well-known/agent-card.json",
    "deployment": {
      "mode": "ENDPOINT"
    }
  }'
```

### Clean up vector store and re-index agents

If vector search returns unexpected results (e.g., `topK` doesn't match the number of results, or scores seem wrong), the vector store may have stale/duplicate entries. Follow these steps to clean up and re-index:

**Step 1: Connect to the pgvector PostgreSQL database**

```bash
docker-compose -f docker-compose.langfuse.yml --env-file .env.langfuse exec pgvector psql -U mosaico -d mosaico_db
```

**Step 2: Inspect the vector_store table (optional)**

```sql
-- Check how many vector entries exist
SELECT COUNT(*) FROM vector_store;

-- View agent-related vectors
SELECT id, metadata FROM vector_store WHERE metadata->>'entityType' = 'Agent';
```

**Step 3: Delete all agent vectors**

```sql
-- Remove all agent vectors from the vector store
DELETE FROM vector_store WHERE metadata->>'entityType' = 'Agent';

-- Verify deletion
SELECT COUNT(*) FROM vector_store WHERE metadata->>'entityType' = 'Agent';
```

**Step 4: Exit PostgreSQL**

```sql
\q
```

**Step 5: Re-add your agents via the API**

Delete existing agents (optional, if you want a fresh start):

```bash
# Get all agents
curl -X GET 'http://localhost:8080/api/agents' -H 'accept: */*'

# Delete each agent by ID
curl -X DELETE 'http://localhost:8080/api/agents/{AGENT-ID}'
```

Re-add agents using the POST endpoint (this will automatically create new vector embeddings):

```bash
# Add aisp-mini-swe-agent
curl -X POST http://localhost:8080/api/agents \
  -H 'Content-Type: application/json' \
  -d '{
    "name": "aisp-mini-swe-agent",
    "description": "Agent who solves generic software engineering issues using a bash-only tool, including write Python script, explain code, solve coding challenges.",
    "version": "1.0.0",
    "role": "generic software engineering solver",
    "objective": "solves generic software engineering issues",
    "a2aAgentCardUrl": "http://localhost:20000/.well-known/agent-card.json",
    "deployment": {
      "mode": "ENDPOINT"
    }
  }'

# Add metamodel-generation-collaboration-agent
curl -X POST http://localhost:8080/api/agents \
  -H 'Content-Type: application/json' \
  -d '{
    "name": "metamodel-generation-collaboration-agent",
    "description": "Collaboration agent who should handle a query for generating a metamodel.",
    "version": "1.0.0",
    "role": "collaboration agent",
    "objective": "coordinate solution agent and supervisor agent to generate and verify metamodel based on user description.",
    "a2aAgentCardUrl": "http://localhost:12000/.well-known/agent-card.json",
    "deployment": {
      "mode": "ENDPOINT"
    }
  }'
```

**Step 6: Verify vector search works correctly**

Test using the MCP Inspector or curl:

```bash
# Via REST API - search with topK=2 should return 2 agents
curl -X POST 'http://localhost:8080/api/agents/search/semantic' \
  -H 'Content-Type: application/json' \
  -d '{
    "query": "agent",
    "topK": 2
  }'
```