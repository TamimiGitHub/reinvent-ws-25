# Workshop: Using Solace Agent Mesh with Real-Time FAA Data

## Overview

In this hands-on workshop, you will explore how to use Solace Agent Mesh to analyze real-time FAA (Federal Aviation Administration) flight data streams. This workshop demonstrates how to transform complex aviation data into actionable insights using AI-powered agents and natural language queries.

By the end of this workshop, you will have built a multi-agent system that can:
- Interact with real-time flight data from multiple FAA systems
- Analyze flight plan adherence
- Generate automated landing reports
- Provide operational insights to flight planners, operators, and controllers

You will use: 
- AWS resources: EC2 Instances, DocumentDB, Bedrock LLM, AgentCore
- Agentic Frameworks: MCP, A2A, Strands, Solace Agent Mesh
- Solace Platform: Event Broker

## Understanding the Problem

### Scenario

You are engineers working for the FAA. Your mission is to use AI to enhance the experience of flight planners, operators, and air traffic controllers. These users are domain experts who understand aviation operations but may not be familiar with the structure, schema, or meaning of the FAA's numerous data sources.


### The Challenge

The primary challenge is bridging the gap between domain expertise and data expertise. Most users know their operational domain intimately but lack knowledge of:
- Database schemas and data structures
- Query languages and syntax
- The relationships between different data sources
- How to extract meaningful insights from raw data
- How to interact with other data sources outside the FAA ecosystem

### Your Task

Create an Agentic information discovery system that allows users to "chat with their data" using natural language. This agent-based approach democratizes data access and enables anyone to extract insights without requiring technical database knowledge.

![image](../placeholder.png)
_Figure 1: Agent Mesh architecture overview_

## The FAA Data Landscape

### Understanding Event-Driven Architecture

The FAA utilizes a publish/subscribe messaging architecture for real-time flight information distribution, powered by the Solace Event Mesh. This architecture provides several key advantages:

- **Selective consumption**: Subscribers can consume only the data streams relevant to their operations
- **Dynamic data flow**: New subscribers can be added without impacting existing systems
- **Scalability**: The event mesh handles high-volume, real-time data distribution efficiently
- **Decoupling**: Publishers and subscribers operate independently

![image](../placeholder.png)
_Figure 1: FAA usecae overview_


### The Need for Historical Context

While real-time event streams provide the latest information, AI-powered analysis often requires historical context to identify patterns, trends, and anomalies. For this workshop, flight data is continuously landed into a DocumentDB instance that maintains approximately 10 minutes of rolling historical messages.

_Note: The data landing infrastructure has already been configured in your workshop environment._

![image](../placeholder.png)
_Figure 2: Event mesh and data landing architecture_

## Part 1: The Data Streams - FDPS, STDDS, and RAG

### Introduction to FAA Systems

In this workshop, we'll work with two critical FAA data systems:

#### FDPS (Flight Data Processing System)

FDPS is the FAA's primary system for managing flight plan data and tracking aircraft through the National Airspace System (NAS). It processes flight plans, monitors aircraft positions, and coordinates handoffs between air traffic control facilities.

#### STDDS (Surface Data Distribution System)

STDDS provides real-time information about aircraft movements on airport surfaces, including taxiways, runways, and ramps. This system is crucial for ground traffic management and preventing runway incursions.

### Understanding System Documentation with RAG

In a real FAA project, each system comes with extensive CONOPS (Concept of Operations) documentation—often exceeding 200 pages per system. Rather than manually reading through these documents (which we want ask you to do!) we'll use Agent Mesh's RAG (Retrieval-Augmented Generation) capabilities to extract relevant information.

### Exercise: Create a RAG agent

In this exercise, you'll set up a Retrieval Augmented Generation (RAG) agent that can ingest, process, and answer questions about FAA system documentation. This agent will enable you to extract insights from complex technical documents without having to read hundreds of pages manually.

#### Step 1: Add the RAG Plugin

2. Run the following command to add the RAG plugin:

```sh
sam plugin add faa-docs-agent --plugin sam-rag
```

3. This will create a new agent configuration file at `configs/agents/faa-docs-agent.yaml`

**Expected Outcome:** You should see confirmation that the plugin was added successfully and the configuration file was created.

![image](../placeholder.png)
_Figure X: Adding the RAG plugin_

#### Step 2: Configure the RAG Agent

Open `configs/agents/faa-docs-agent.yaml` in your editor and understand the following sections:

**Scanner Configuration:**
This section tells the agent where to find documents and which file types to process.

```yaml
scanner:
  batch: true
  use_memory_storage: true
  source:
    type: filesystem
    directories:
      - "./faa_documents"  # Directory where you'll store FAA documentation
    filters:
      file_formats:
        - ".txt"
        - ".pdf"
        - ".md"
```

**Preprocessor Configuration:**
This section defines how the text is cleaned and normalized before processing.

```yaml
preprocessor:
  default_preprocessor:
    type: enhanced
    params:
      lowercase: true
      normalize_whitespace: true
      remove_urls: false
```

**Splitter Configuration:**
This section determines how documents are split into manageable chunks for embedding.

```yaml
splitter:
  default_splitter:
    type: recursive_character
    params:
      chunk_size: 2048
      chunk_overlap: 400
  splitters:
    markdown:
      type: markdown
      params:
        chunk_size: 2048
        chunk_overlap: 400
    pdf:
      type: token
      params:
        chunk_size: 500
        chunk_overlap: 100
```

**Embedding Configuration:**
This section configures the model used to create vector embeddings from text chunks.

```yaml
embedding:
  embedder_type: "openai"
  embedder_params:
    model: "${OPENAI_EMBEDDING_MODEL}"
    api_key: "${OPENAI_API_KEY}"
    api_base: "${OPENAI_API_ENDPOINT}"
  normalize_embeddings: true
```

**Vector Database Configuration:**
This section configures the connection to your vector database where embeddings are stored.

```yaml
vector_db:
  db_type: "qdrant"
  db_params:
    url: "${QDRANT_URL}"
    api_key: "${QDRANT_API_KEY}"
    collection_name: "faa-documentation"
    embedding_dimension: 1536
```

**LLM Configuration:**
This section configures the language model used to generate answers based on retrieved context.

```yaml
llm:
  load_balancer:
    - model_name: "gpt-4o"
      litellm_params:
        model: "openai/${OPENAI_MODEL_NAME}"
        api_key: "${OPENAI_API_KEY}"
        api_base: "${OPENAI_API_ENDPOINT}"
```

**Retrieval Configuration:**
This section defines how many document chunks are retrieved to answer a query.

```yaml
retrieval:
  top_k: 7
```

**Expected Outcome:** A complete configuration file that defines all aspects of the RAG pipeline.

![image](../placeholder.png)
_Figure X: RAG agent configuration_

#### Step 3: Set Up Environment Variables

Update your `.env` file with the following variables. Your workshop instructor will provide the specific values for your environment.

```bash

```

**Expected Outcome:** A properly configured environment file with all necessary credentials and endpoints.

#### Step 4: Create Document Directory

1. Create a directory for your FAA documentation:

```sh
mkdir -p faa_documents
```

2. This directory will be used in the next exercise to store the FDPS and STDDS CONOPS documents

**Expected Outcome:** A new directory where you'll store the FAA documentation files.

![image](../placeholder.png)
_Figure X: Creating the document directory_

#### Step 5: Run the RAG Agent

1. Start your RAG agent with the following command:

```sh
sam run configs/agents/faa-docs-agent.yaml
```

2. The agent will initialize and connect to your vector database
3. It will be ready to scan and ingest documents in the next exercise

**Expected Outcome:** The agent starts successfully and displays initialization logs.

![image](../placeholder.png)
_Figure X: Running the RAG agent_

#### Step 6: Verify Agent Setup

1. Check the terminal output for any errors
2. Verify that the agent has successfully connected to:
   - The Solace broker
   - The vector database
3. If you see any connection errors, double-check your environment variables and configuration

**Expected Outcome:** All connections are established successfully, and the agent is ready to process documents.

![image](../placeholder.png)
_Figure X: Successful agent initialization_

Now that your RAG agent is set up, you're ready to proceed to the next exercise where you'll upload and analyze FAA system documentation. This agent will help you quickly extract insights from complex technical documents without having to read them in their entirety.

### Exercise: Analyzing System Documentation

#### Step 1: Upload CONOPS Documents

1. Navigate to the Agent Mesh interface
1. Upload the FDPS CONOPS document to Agent Mesh
1. Upload the STDDS CONOPS document to Agent Mesh
1. Put the following promots
```
Upload the following documents to my vector database
```

![image](../placeholder.png)
_Figure 3: Document upload interface_

#### Step 2: Query the RAG Agent

Use the RAG agent to analyze the uploaded documentation by running the following prompts:

```
Tell me about the FDPS and STDDS systems and data.
```

_Wait for the response, review the summary, then proceed with the next prompt:_

```
Tell me what types of outcomes I could expect by analyzing data from these systems.
```

#### Expected Outcomes

The RAG agent will provide:
- High-level overviews of each system's purpose and functionality
- Descriptions of the data structures and key fields
- Insights into potential analysis opportunities
- Relationships between the two systems

_This approach is significantly more efficient than reading hundreds of pages of technical documentation manually._

![image](../placeholder.png)
_Figure 4: RAG agent response example_

## Part 2: Setting Up the Data Layer

### Understanding Your Workshop Environment

Your workshop environment includes a pre-configured DocumentDB instance that serves as the data layer for this exercise. This instance contains the primary collections that store real-time flight data.

### Collection Overview

#### FDPSPosition Collection

This collection contains position reports and flight plan data from the FDPS system, including:
- Aircraft identification
- Current position (latitude, longitude, altitude)
- Flight plan information
- Route adherence data
- Timestamps and status indicators

#### STDDSPosition Collection

This collection contains surface movement data from the STDDS system, including:
- Aircraft identification
- Ground position coordinates
- Taxiway and runway assignments
- Ground speed and heading
- Surface movement status

### Data Ingestion Architecture

The collections are continuously populated by a micro-integration that:

1. **Subscribe to Solace Event Mesh**: Each micro-integration subscribes to specific topic patterns for FDPS or STDDS data
2. **Receive real-time events**: As flight data is published, the micro-integrations receive relevant messages
3. **Transform and validate**: Data is validated and transformed as needed for database storage
4. **Write to DocumentDB**: Processed messages are written to the appropriate collection
5. **Maintain rolling window**: Old data is automatically purged to maintain the 10-minute historical window

![image](../placeholder.png)
_Figure 5: Micro-integration architecture diagram_

_These micro-integrations are already running in your environment, continuously populating the database with live FAA data._

![image](../placeholder.png)
_Figure 6: Data flow from Event Mesh to DocumentDB_

## Part 3: Creating DocumentDB Agents in Agent Mesh

Now that you understand the data layer, it's time to connect Agent Mesh to your DocumentDB collections. You'll create dedicated agents for each data source to enable parallel processing and maintain a clean architectural separation.

### Step 1: Create the FDPS DocumentDB Agent

### Step 2: Create the STDDS MongoDB Agent

### Step 3: Test the Agent Connections

#### Start the Agent Mesh Interface

1. Ensure Agent Mesh is running in your environment
2. Open a web browser
3. Navigate to the Agent Mesh interface URL (format: `http://<your-workshop-url>:8000`)

![image](../placeholder.png)
_Figure 10: Agent Mesh main interface_

#### Verify Agent Availability

1. Navigate to the "Agents" tab or section
2. Confirm that you see both newly created agents:
   - FDPS Agent
   - STDDS Agent
3. Verify that both agents show as "Available" or "Online"

![image](../placeholder.png)
_Figure 11: Agents list showing active agents_

#### Test the FDPS Agent

In the Agent Mesh chat or query interface, run the following prompt:

```
Get me an example document from the FDPS database.
```

**Expected Response:**

The agent should return a JSON document showing the structure of FDPS data, including fields such as:
- Aircraft identifier (flight number, tail number)
- Position data (latitude, longitude, altitude)
- Speed and heading
- Flight plan information
- Status indicators
- Timestamp

![image](../placeholder.png)
_Figure 12: Example FDPS document response_

#### Test the STDDS Agent

Run the following prompt:

```
Get me an example document from the STDDS database.
```

**Expected Response:**

The agent should return a JSON document showing the structure of STDDS data, including fields such as:
- Aircraft identifier
- Ground position coordinates
- Surface location (runway, taxiway, ramp)
- Ground speed and direction
- Surface status
- Timestamp

![image](../placeholder.png)
_Figure 13: Example STDDS document response_

#### Review the Data Structures

Examine the returned documents carefully to understand:
- Available fields and their data types
- Field naming conventions
- Data formats (especially for timestamps and coordinates)
- Relationships between different fields

_This knowledge will be essential for formulating effective natural language queries in the next section._

## Part 4: Asking Questions in Natural Language

Now that your agents are connected and tested, you can begin querying the data using natural language. Agent Mesh will interpret your questions, determine which agent(s) to consult, construct appropriate database queries, and present the results in a user-friendly format.

### Exercise: Querying Inbound Flights

#### Query for LAS Arrivals

In the Agent Mesh interface, enter the following prompt:

```
Provide me a summary of flights inbound into LAS.
```

_Note: LAS is the IATA airport code for Las Vegas Harry Reid International Airport_

![image](../placeholder.png)
_Figure 14: Inbound flights query interface_

#### Understanding the Response

Agent Mesh will:
1. Identify that this query requires FDPS data (flight positions and routes)
2. Route the request to the FDPS Agent
3. Construct a database query to find aircraft with destination LAS
4. Filter for inbound flights (flights currently en route, not yet landed)
5. Aggregate and summarize the results
6. Present a human-readable summary

**Expected Summary Elements:**
- Number of inbound flights
- Flight identifiers and airlines
- Current positions and estimated times of arrival
- Altitude and speed information
- Origin airports

![image](../placeholder.png)
_Figure 15: Example summary of inbound flights_

### Exercise: Querying Ground Operations

#### Query for Ground Traffic

Enter the following prompt:

```
Provide me a summary of flights on the ground and taxiing in LAS.
```

#### Understanding the Response

Agent Mesh will:
1. Recognize this requires STDDS data (surface movements)
2. Route the request to the STDDS Agent
3. Query for aircraft at LAS with ground/taxiing status
4. Summarize surface operations

**Expected Summary Elements:**
- Number of aircraft on the ground
- Aircraft currently taxiing vs. parked
- Taxiway and runway occupancy
- Departure vs. arrival operations
- Ground movement patterns

![image](../placeholder.png)
_Figure 16: Example summary of ground operations_

### Extracting Operational Insights

The natural language interface enables non-technical users to:
- Monitor airport congestion in real-time
- Identify potential bottlenecks in ground operations
- Track inbound traffic for resource planning
- Generate operational reports without writing SQL or code

_Try experimenting with your own queries to explore different aspects of the data, such as specific airlines, aircraft types, or time-based patterns._

## Part 5: Adding Flight Plan Intelligence

To perform more sophisticated analysis, such as comparing actual flight paths to planned routes, you need access to flight plan data. In this section, you'll add another DocumentDB agent to access this information.

### Understanding Flight Plan Data

Flight plans contain:
- Planned route waypoints and altitudes
- Estimated times for each waypoint
- Alternate airports and contingency routes
- Fuel calculations and aircraft performance data
- Filed vs. actual departure times

### Step 1: Create the Flight Plan DocumentDB Agent

#### Configure the Agent

Following the same process used in Part 3, create a new DocumentDB agent:

### Step 2: Test Flight Plan Access

Run a test query to verify access to flight plan data:

```
Get me an example flight plan document from the database.
```

Review the returned document to understand the structure of flight plan data.

![image](../placeholder.png)
_Figure 18: Example flight plan document_

### Step 3: Analyze Flight Plan Adherence

Now that you have access to both actual position data and flight plans, you can perform adherence analysis.

#### Query for a Specific Flight

Replace `XXYYYY` with an actual flight number from your data (you can find flight numbers from the earlier queries), then run:

```
How well has flight XXYYYY adhered to its flight plan?
```

![image](../placeholder.png)
_Figure 19: Flight plan adherence query_

#### Understanding the Analysis

Agent Mesh will:
1. Retrieve the flight plan for flight XXYYYY from the Flight Plan Agent
2. Retrieve actual position data from the FDPS Agent
3. Compare actual positions and times against planned waypoints and schedule
4. Calculate deviations in:
   - Route (lateral deviation from planned path)
   - Altitude (vertical deviation from planned altitude)
   - Timing (early/late compared to estimates)
5. Provide a summary of adherence quality

![image](../placeholder.png)
_Figure 20: Flight plan adherence analysis results_

### Use Cases for Flight Plan Intelligence

With flight plan analysis capabilities, users can:
- Identify flights experiencing significant deviations
- Assess the impact of weather or airspace restrictions
- Evaluate airline operational efficiency
- Support post-flight analysis and investigations
- Optimize future flight planning based on historical adherence

_This multi-agent coordination demonstrates the power of Agent Mesh to orchestrate complex analyses across multiple data sources seamlessly._

## Part 6: Event-Based Agentic AI

So far, you've been querying data on-demand. In this section, you'll configure Agent Mesh to automatically generate reports in response to real-time events—specifically, when flights land.

### Understanding Event-Driven Agentic AI

The FAA's FDPS system publishes status updates for every flight. When a flight lands, its status changes to `DROPPED`. By subscribing to these events, you can trigger automated workflows without manual intervention.

### The Event Gateway

The Agent Mesh Event Gateway is a component that:
- Subscribes to topics on the Solace Event Mesh
- Receives events matching subscription patterns
- Triggers configured agents or workflows when events arrive
- Can publish results back to the event mesh

![image](../placeholder.png)
_Figure 21: Event Gateway architecture_

### Step 1: Configure the Event Gateway

**Topic Subscription:**
```
faa/fdps/status/DROPPED/>
```

_This topic pattern subscribes to all FDPS status messages where the status is DROPPED_

**Event Filter (optional):**

![image](../placeholder.png)
_Figure 23: Subscription configuration_

### Step 2: Create the Landing Report Workflow

#### Define the Workflow

When a DROPPED event is received, the Event Gateway should:

1. **Extract flight information** from the event payload (flight number, timestamp, airport)
1. **Starts Stimulus to Agent Mesh** and the orchestrator agent receives the prompt
1. **Invoke the FDPS Agent** to retrieve the full flight history
1. **Invoke the Flight Plan Agent** to retrieve the original flight plan
1. **Generate a landing report** that includes:
   - Flight identification and basic details
   - Actual vs. planned landing time
   - Route adherence summary
   - Flight duration and performance metrics
   - Any notable deviations or events
1. **Publish the report** back to the event mesh for consumption by other systems

#### Review a Generated Report

Subscribe to the report topic or access the generated report to verify it contains:
- Flight identification
- Landing timestamp and airport
- Planned vs. actual landing time comparison
- Route adherence summary
- Performance metrics
- Any anomalies or noteworthy events

![image](../placeholder.png)
_Figure 27: Example automated landing report_

### Benefits of Event-Driven Reporting

This automated approach provides:
- **Immediate insights**: Reports generated within seconds of landing
- **Consistency**: Every landing produces a standardized report
- **Scalability**: Handles hundreds of landings per hour without manual effort
- **Integration**: Reports available on the event mesh for any consuming system
- **Reduced workload**: Eliminates manual report creation and distribution

_Consider what other events in your enterprise could trigger automated AI-driven analysis and reporting._

## Final Takeaways

### What You've Accomplished

Congratulations! In this workshop, you have:

1. **Utilized Agent Mesh for real-time FAA data analysis**: You've connected to live aviation data streams and extracted meaningful insights using natural language queries

2. **Built a multi-agent architecture**: You created specialized DocumentDB agents for FDPS, STDDS, and flight plan data, demonstrating how Agent Mesh orchestrates multiple data sources

3. **Integrated RAG-based summarization**: You used RAG to analyze technical documentation, extracting relevant information without reading hundreds of pages

4. **Automated event-driven reporting**: You configured the Event Gateway to trigger agentic stimuli automatically in response to real-time events, demonstrating the power of event-driven Agentic AI

![image](../placeholder.png)
_Figure 28: Complete workshop architecture_

### Key Concepts Reviewed

#### Agent-Based Architecture

You've seen how individual agents, each with a specific purpose and data source, can be combined to answer complex questions that span multiple systems.

#### Natural Language Data Access

Non-technical users can now access FAA data without knowing database schemas, query languages, or system architectures—democratizing data access across the organization.

#### Event-Driven AI

By connecting AI agents to real-time event streams, you've created a system that proactively generates insights rather than waiting for users to ask questions.

#### The Power of Integration

Solace Agent Mesh bridges AI capabilities with enterprise event-driven architecture, creating intelligent, responsive systems that enhance decision-making.

### Next Steps

To continue your Agent Mesh journey:

1. **Explore additional agent types**: Experiment with REST API agents, SQL database agents, or custom agents for your specific systems

2. **Build more complex workflows**: Create multi-step workflows that chain together multiple agents and decision points

3. **Integrate with your systems**: Connect Agent Mesh to your organization's actual data sources and event streams

4. **Train your team**: Share what you've learned and identify use cases within your organization

5. **Join the community**: Connect with other Agent Mesh users to share patterns, solutions, and best practices

