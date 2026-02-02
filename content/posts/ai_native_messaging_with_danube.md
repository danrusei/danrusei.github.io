---
title: "AI-Native Messaging: Managing Danube with Natural Language"
date: 2026-02-01
draft: false
categories: ["Danube Messaging"]
author: "Dan Rusei"

autoCollapseToc: true
contentCopyright: MIT

---

🌊 [Danube Messaging](https://github.com/danube-messaging/danube/tree/main) is a modern cloud-native messaging platform built in Rust, designed for high-performance pub/sub messaging with intelligent cluster management. It provides durable message streaming, schema registry, and automatic load balancing, all while being easy to operate, especially when paired with 🤖 AI assistants.

## Meet danube-admin: Three Tools in One Binary

The [danube-admin](https://github.com/danube-messaging/danube/tree/main/danube-admin) binary is the Swiss Army knife for Danube cluster management. It combines three powerful interfaces in a single executable:

### 1. >_ CLI Mode

Traditional command-line interface for scripts, automation, and quick operations:

```bash
danube-admin brokers list
danube-admin topics create /default/events --partitions 4
danube-admin schemas register user-events --schema-type json_schema --file schema.json
```

### 2.🌐 Web Server Mode

HTTP/REST API for the [Danube Admin UI](https://github.com/danube-messaging/danube-admin-ui), a modern React-based web interface for visual cluster management and monitoring:

```bash
danube-admin serve --mode ui ----broker-endpoint broker1:50051 --listen-addr 0.0.0.0:8080
```

### 3. 🔌 MCP Mode

🤖 AI-native interface that connects your AI assistant directly to Danube for natural language cluster management:

```bash
danube-admin serve --mode mcp
```

This unified approach means you download one binary and get instant access to Danube from your terminal, applications, *and ✨ AI assistants*.

## What Danube MCP Provides

The Danube MCP server exposes **40+ specialized tools** and **several guided workflow prompts** that make cluster management feel like a conversation.

### Tools (Automated Actions)

**🏗️ Cluster Management:**
Ask the AI to discover which brokers are running and their roles, check who the cluster leader is, or analyze load distribution across brokers. When you notice imbalance, trigger rebalancing to redistribute topics evenly.

Preparing for maintenance? Unload all topics from a broker before shutting it down, then activate it again when ready. Create and manage namespaces to organize the topics by team or environment.

**📨 Topic Operations:**  
Create topics with the preferred partitioning strategy and delivery guarantees, just describing the need. Configure schema validation for topics to enforce data contracts.

Check detailed topic configurations including which broker owns them and what subscriptions are active. Delete topics or remove old subscriptions with a simple request.

**📋 Schema Registry:**  
Register new schemas or evolve existing ones while the AI ensures compatibility with previous versions. Ask to see all schema versions for a subject to understand its evolution history.

**🔧 Diagnostics & Observability**  
Get a comprehensive health check that analyzes broker status, load distribution, and identifies issues automatically. Request optimization recommendations based on current cluster state—the AI will prioritize actions and explain why they matter.

Query real-time metrics from Prometheus for the entire cluster, specific brokers, or individual topics—see message rates, throughput, consumer counts, and latency percentiles. Access logs from brokers whether they're running in Docker containers, Kubernetes pods, or as local processes. The AI can correlate metrics and logs to give you the full picture when troubleshooting.

### Prompts (Guided Workflows)

Prompts are pre-built multi-step workflows that guide the AI through complex operations:

**Operations:**

- `setup_new_topic` - Best-practice topic creation with configuration guidance
- `manage_schema_evolution` - Safe schema updates with compatibility checks
- `prepare_cluster_rebalance` - Dry-run validation and execution workflow

**Troubleshooting:**

- `diagnose_broker_issues` - Broker health deep-dive
- `analyze_topic_performance` - Performance bottleneck analysis
- `cluster_health_check` - Complete health audit

## Demo: ✨ Managing Danube with AI

Here's a real conversation showing how natural cluster management becomes with AI. No commands to memorize, no flags to look up—just describe what you need.

### Checking Cluster Status

**ME:** "List the brokers from Danube cluster"

**AI:**

```bash
The Danube cluster has 2 brokers:

Broker ID: 13784082042618672561
Status: active
Role: Cluster_Follower
Address: http://broker2:6650

Broker ID: 10284038671600611281
Status: active
Role: Cluster_Leader
Address: http://broker1:6650

Both brokers are active with one serving as the cluster leader.
```

### Identifying Load Imbalance

**ME:** "What is the distribution across the brokers?"

**AI:**

```bash
The cluster is severely imbalanced:

Broker 13784082042618672561 (Follower): 2 topics - UNDERLOADED
Broker 10284038671600611281 (Leader): 6 topics - OVERLOADED

Balance Metrics:
- Coefficient of Variation: 0.5000 (severe imbalance)
- Mean Load: 4.00 topics per broker
- Load Range: 2-6 topics (3x difference)

Recommendation: Run trigger_rebalance to redistribute topics evenly.
```

### Rebalancing with Confidence

**ME:** "Yes please rebalance the brokers"

**AI:**

```bash
Rebalancing completed successfully!

Topic Movement:
✓ Moved /default/pattern_4-part-1 from overloaded leader to underloaded follower

Result: 1 move initiated

Let me verify the new balance...

✅ Rebalancing successful! The cluster is now balanced:

New Distribution:
- Broker 13784082042618672561: 3 topics ✓
- Broker 10284038671600611281: 5 topics ✓

Improved Metrics:
- Status: ✓ Balanced (was severely imbalanced)
- CV: 0.2500 (was 0.5000)
- Load Range: 3-5 topics (was 2-6)
```

### Creating Topics with Intelligence

**ME:** "I would like to create a topic, what info do I need to provide?"

**AI:**

```bash
To create a topic, you need:

Required:
- Topic name: /namespace_name/topic_name

Optional:
- Partitions: 0 (non-partitioned, default) or higher for parallelism
- Schema subject: For schema validation
- Delivery strategy: "reliable" (durable) or "non_reliable" (fast)

Available namespaces: default, system

Would you like me to create a specific topic?
```

**ME:** "Great, create topic /default/my_ai_topic, reliable with 4 partitions and no schema"

**AI:**

```bash
✅ Topic created successfully!

Topic Details:
- Name: /default/my_ai_topic
- Partitions: 4 (parallel processing enabled)
- Delivery Strategy: reliable (WAL + cloud storage)
- Schema: None

The topic is ready for use with reliable persistence!
```

### Cluster Health Monitoring

**ME:** "Give me some statistics about the brokers"

**AI:**

```bash
📊 Danube Cluster Statistics

Cluster Overview:
- Brokers: 2 active
- Topics: 8 total
- Consumers: 12 active
- Subscriptions: 7 active
- Balance: 0.2500 CV (well balanced)

Throughput (1 min avg):
- Messages In: 0.00 msg/s
- Messages Out: 0.00 msg/s
- Currently idle, 12 consumers ready

Broker 10284038671600611281 (Leader):
- Topics Owned: 8
- RPC Total: 32 operations
- Consumers: 12

Summary: Cluster is healthy and well-balanced, ready for traffic.
```

### Using Guided Workflows

**ME:** `/danube-admin/cluster_health_check`

**AI:**

```bash
🏥 Danube Cluster Health Check Report

Step 1: Cluster Overview ✅
- 2 active brokers (leader + follower)
- 8 topics, 12 consumers, 7 subscriptions

Step 2: Broker Health ✅
- All brokers online: 2/2 active
- Leader elected: Broker 10284038671600611281
- No error states detected

Step 3: Load Distribution ✅
- Balance Status: Well Balanced (CV: 0.0000)
- Distribution: 6 topics per broker (perfect)

Step 4: Automated Diagnostics ✅
- Connectivity: All brokers reachable
- Configuration: Proper setup
- Namespaces: 2 available

Step 5: Recommendations
💡 Medium Priority: Consider 3+ brokers for production HA

🟢 Overall Health: HEALTHY

Top Actions:
✅ No immediate actions required
📊 Monitor when producers start generating traffic
```

Notice how the AI doesn't just execute commands—it *understands* the results, provides context, suggests next steps, and validates outcomes automatically.

## Getting Started

### 1. Run Danube

The fastest way to start is using Docker Compose. Download the required files:

```bash
# Create a directory for Danube
mkdir danube-quickstart && cd danube-quickstart

# Download docker-compose configuration
curl -O https://raw.githubusercontent.com/danube-messaging/danube/main/docker/quickstart/docker-compose.yml
curl -O https://raw.githubusercontent.com/danube-messaging/danube/main/docker/danube_broker.yml
curl -O https://raw.githubusercontent.com/danube-messaging/danube/main/docker/prometheus.yml

# Start the cluster
docker-compose up -d
```

This starts a 2-broker cluster with ETCD and Prometheus. For more deployment options, see the [Docker deployment guide](https://github.com/danube-messaging/danube/tree/main/docker).

### 2. Get danube-admin

Download the latest release for your platform from [GitHub Releases](https://github.com/danube-messaging/danube/releases):

**Linux:**

```bash
curl -L -o danube-admin https://github.com/danube-messaging/danube/releases/latest/download/danube-admin-linux
chmod +x danube-admin
```

**macOS (Apple Silicon):**

```bash
curl -L -o danube-admin https://github.com/danube-messaging/danube/releases/latest/download/danube-admin-macos
chmod +x danube-admin
```

**Windows:**  
Download `danube-admin-windows.exe` from the releases page.

**Docker (alternative):**

```bash
docker run --rm ghcr.io/danube-messaging/danube-admin:latest danube-admin --help
```

### 3. Configure AI MCP in the IDE

#### Windsurf

Windsurf MCP config (`~/.codeium/windsurf/mcp_config.json`):

```json
{
  "mcpServers": {
    "danube-admin": {
      "command": "sh",
      "args": [
        "-c",
        "/path/to/danube-admin serve --mode mcp --broker-endpoint http://127.0.0.1:50051"
      ],
      "env": {
        "PATH": "/usr/local/bin:/usr/bin:/bin",
        "NO_COLOR": "1",
        "RUST_LOG": "error"
      }
    }
  }
}
```

#### VSCode (with Continue extension or similar)

VSCode MCP config:

```json
{
  "servers": {
    "danube-admin": {
      "type": "stdio",
      "command": "/path/to/danube-admin",
      "args": [
        "serve",
        "--mode",
        "mcp",
        "--broker-endpoint",
        "http://127.0.0.1:50051"
      ],
      "env": {
        "NO_COLOR": "1",
        "RUST_LOG": "error"
      }
    }
  }
}
```

#### Claude Desktop

Add to `~/Library/Application Support/Claude/claude_desktop_config.json` (macOS) or `%APPDATA%\Claude\claude_desktop_config.json` (Windows):

```json
{
  "mcpServers": {
    "danube-admin": {
      "command": "/path/to/danube-admin",
      "args": ["serve", "--mode", "mcp", "--broker-endpoint", "http://localhost:50051"],
      "env": {
        "NO_COLOR": "1",
        "RUST_LOG": "error"
      }
    }
  }
}
```

**Replace `/path/to/danube-admin`** with your actual binary path:

- Development: `./target/debug/danube-admin` or `./target/release/danube-admin`
- Installed: `/usr/local/bin/danube-admin` or wherever you placed it

### 4. Optional: Advanced Configuration

For log access and metrics, create `mcp-config.yml`:

```yaml
broker_endpoint: http://127.0.0.1:50051
prometheus_url: http://localhost:9090  # If running Prometheus

# For Docker deployments
deployment:
  type: docker
  docker:
    container_mappings:
      - id: "broker1"
        container: "danube-broker1"
      - id: "broker2"
        container: "danube-broker2"
```

Then update the MCP config to include `--config /path/to/mcp-config.yml` in the args.

### 5. Start Using AI

Restart your AI IDE and start asking questions:

- "List all brokers in my Danube cluster"
- "Create a new topic for user events with 4 partitions"
- "Check if the cluster is balanced"
- "Show me metrics for the /default/events topic"

Or use guided workflows from the prompt picker:

- Select "Setup New Topic" for step-by-step topic creation
- Select "Cluster Health Check" for comprehensive diagnostics
- Select "Manage Schema Evolution" for safe schema updates

## Why This Matters

Traditional messaging platforms require deep expertise: memorizing CLI flags, understanding complex metrics, manually correlating logs and performance data. With Danube's AI-native approach:

✅ **Natural language replaces commands** - Describe what you need, not how to do it  
✅ **Context-aware assistance** - AI understands cluster state and suggests optimizations  
✅ **Multi-step workflows** - Complex operations become guided conversations  
✅ **Proactive monitoring** - AI spots issues before you ask  
✅ **Zero learning curve** - New team members productive from day one  

---

**Show your support:** Star the [Danube repository](https://github.com/danube-messaging/danube) or contribute to the project on GitHub!
