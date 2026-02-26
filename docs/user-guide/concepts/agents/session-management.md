# Session Management

Session management in Strands Agents provides a robust mechanism for persisting agent state and conversation history across multiple interactions. This enables agents to maintain context and continuity even when the application restarts or when deployed in distributed environments.

## Overview

A session represents all of stateful information that is needed by agents and multi-agent systems to function, including:

**Single Agent Sessions**:

- Conversation history (messages)
- Agent state (key-value storage)
- Other stateful information (like [Conversation Manager](./state.md#conversation-manager))

**Multi-Agent Sessions**:

- Orchestrator state and configuration
- Individual agent states and result within the orchestrator
- Cross-agent shared state and context
- Execution flow and node transition history

Strands provides built-in session persistence capabilities that automatically capture and restore this information, allowing agents and multi-agent systems  to seamlessly continue conversations where they left off.

Beyond the built-in options, [third-party session managers](#third-party-session-managers) provide additional storage and memory capabilities.

!!! warning
    **(Python only)** You cannot use a single agent with session manager in a multi-agent system. This will throw an exception. Each agent in a multi-agent system must be created without a session manager, and only the orchestrator should have the session manager. Additionally, multi-agent session managers only track the current state of the Graph/Swarm execution and do not persist individual agent conversation histories.

## Basic Usage

### Single Agent Sessions

Simply create an agent with a session manager and use it:

=== "Python"
    
    ```python
    from strands import Agent
    from strands.session.file_session_manager import FileSessionManager
    
    # Create a session manager with a unique session ID
    session_manager = FileSessionManager(session_id="test-session")
    
    # Create an agent with the session manager
    agent = Agent(session_manager=session_manager)
    
    # Use the agent - all messages and state are automatically persisted
    agent("Hello!")  # This conversation is persisted
    ```

=== "TypeScript"

    ```typescript
    import { Agent, SessionManager } from '@strands-agents/sdk'

    // Create a session manager with a unique session ID
    const sessionManager = new SessionManager({ sessionId: 'test-session' })

    // Create an agent with the session manager
    const agent = new Agent({ sessionManager })

    // Use the agent - all messages and state are automatically persisted
    await agent.invoke('Hello!')  // This conversation is persisted
    ```

    

The conversation, and associated state, is persisted to the underlying filesystem.

### Multi-Agent Sessions

Multi-agent systems(Graph/Swarm) can also use session management to persist their state:

=== "Python"

    ```python
    from strands.multiagent import Graph
    from strands.session.file_session_manager import FileSessionManager

    # Create agents
    agent1 = Agent(name="researcher")
    agent2 = Agent(name="writer")

    # Create a session manager for the graph
    session_manager = FileSessionManager(session_id="multi-agent-session")

    # Create graph with session management
    graph = Graph(
        agents={"researcher": agent1, "writer": agent2},
        session_manager=session_manager
    )

    # Use the graph - all orchestrator state is persisted
    result = graph("Research and write about AI")
    ```

{{ ts_not_supported_code("Multi-agent session management is not yet supported in the TypeScript SDK.") }}

## Built-in Session Managers

=== "Python"

    Strands offers two built-in session managers for persisting agent sessions:

    1. [**FileSessionManager**](../../../api-reference/python/session/file_session_manager.md#strands.session.file_session_manager.FileSessionManager): Stores sessions in the local filesystem
    2. [**S3SessionManager**](../../../api-reference/python/session/s3_session_manager.md#strands.session.s3_session_manager.S3SessionManager): Stores sessions in Amazon S3 buckets

=== "TypeScript"

    The TypeScript SDK uses a single `SessionManager` class with pluggable storage backends:

    1. **FileStorage** (default): Stores snapshots in the local filesystem
    2. **S3Storage**: Stores snapshots in an Amazon S3 bucket

### FileSessionManager / FileStorage

=== "Python"

    The [`FileSessionManager`](../../../api-reference/python/session/file_session_manager.md#strands.session.file_session_manager.FileSessionManager) provides a simple way to persist both single agent and multi-agent sessions to the local filesystem:

    ```python
    from strands import Agent
    from strands.session.file_session_manager import FileSessionManager

    # Create a session manager with a unique session ID
    session_manager = FileSessionManager(
        session_id="user-123",
        storage_dir="/path/to/sessions"  # Optional, defaults to a temp directory
    )

    # Create an agent with the session manager
    agent = Agent(session_manager=session_manager)

    # Use the agent normally - state and messages will be persisted automatically
    agent("Hello, I'm a new user!")

    # Multi-agent usage
    multi_session_manager = FileSessionManager(
        session_id="orchestrator-456",
        storage_dir="/path/to/sessions"
    )
    graph = Graph(
        agents={"agent1": agent1, "agent2": agent2},
        session_manager=multi_session_manager
    )
    ```

=== "TypeScript"

    `FileStorage` persists snapshots to the local filesystem. It is the default storage backend when no `storage` option is provided:

    ```typescript
    import { Agent, SessionManager, FileStorage } from '@strands-agents/sdk'

    const sessionManager = new SessionManager({
      sessionId: 'user-123',
      // Optional: defaults to OS temp directory
      storage: { snapshot: new FileStorage('/path/to/sessions') },
    })

    const agent = new Agent({ sessionManager })

    await agent.invoke("Hello, I'm a new user!")
    ```

#### File Storage Structure

=== "Python"

    When using [`FileSessionManager`](../../../api-reference/python/session/file_session_manager.md#strands.session.file_session_manager.FileSessionManager), sessions are stored in the following directory structure:

    ```
    /<sessions_dir>/
    └── session_<session_id>/
        ├── session.json                # Session metadata
        ├── agents/                     # Single agent storage
        │   └── agent_<agent_id>/
        │       ├── agent.json          # Agent metadata and state
        │       └── messages/
        │           ├── message_<message_id>.json
        │           └── message_<message_id>.json
        └── multi_agents/               # Multi-agent  storage
            └── multi_agent_<orchestrator_id>/
                └── multi_agent.json    # Orchestrator state and configuration
    ```

=== "TypeScript"

    When using `FileStorage`, snapshots are stored in the following directory structure:

    ```
    /<base_dir>/
    └── <session_id>/
        └── scopes/
            └── agent/
                └── <scope_id> (aka:agent id)/
                    └── snapshots/
                        ├── manifest.json           # Snapshot ID counter
                        ├── snapshot_latest.json    # Latest snapshot (mutable)
                        └── immutable_history/
                            ├── snapshot_00001.json
                            ├── snapshot_00002.json
                            └── ...
    ```

### S3SessionManager / S3Storage

=== "Python"

    For cloud-based persistence, especially in distributed environments, use the [`S3SessionManager`](../../../api-reference/python/session/s3_session_manager.md#strands.session.s3_session_manager.S3SessionManager):

    ```python
    from strands import Agent
    from strands.session.s3_session_manager import S3SessionManager
    import boto3

    # Optional: Create a custom boto3 session
    boto_session = boto3.Session(region_name="us-west-2")

    # Create a session manager that stores data in S3
    session_manager = S3SessionManager(
        session_id="user-456",
        bucket="my-agent-sessions",
        prefix="production/",  # Optional key prefix
        boto_session=boto_session,  # Optional boto3 session
        region_name="us-west-2"  # Optional AWS region (if boto_session not provided)
    )

    # Create an agent with the session manager
    agent = Agent(session_manager=session_manager)

    # Use the agent normally - state and messages will be persisted to S3
    agent("Tell me about AWS S3")

    # Use with multi-agent orchestrator
    swarm = Swarm(
        agents=[agent1, agent2, agent3],
        session_manager=session_manager
    )

    result = swarm("Coordinate the task across agents")
    ```

=== "TypeScript"

    `S3Storage` persists snapshots to an Amazon S3 bucket. The bucket must already exist — `S3Storage` does not create it automatically:

    ```typescript
    import { Agent, SessionManager, S3Storage } from '@strands-agents/sdk'

    const sessionManager = new SessionManager({
      sessionId: 'user-456',
      storage: {
        snapshot: new S3Storage({
          bucket: 'my-agent-sessions',
          prefix: 'production',   // Optional key prefix
          region: 'us-west-2',    // Optional, defaults to us-east-1
        }),
      },
    })

    const agent = new Agent({ sessionManager })

    await agent.invoke('Tell me about AWS S3')
    ```

    You can also provide a pre-configured `S3Client` instead of a region:

    ```typescript
    import { S3Client } from '@aws-sdk/client-s3'
    import { S3Storage } from '@strands-agents/sdk'

    const s3Client = new S3Client({ region: 'us-west-2' })

    const storage = new S3Storage({ bucket: 'my-agent-sessions', s3Client })
    ```

#### S3 Storage Structure

=== "Python"

    Just like in the [`FileSessionManager`](../../../api-reference/python/session/file_session_manager.md#strands.session.file_session_manager.FileSessionManager), sessions are stored with the following structure in the s3 bucket:

    ```
    <s3_key_prefix>/
    └── session_<session_id>/
        ├── session.json                # Session metadata
        ├── agents/                     # Single agent storage
        │   └── agent_<agent_id>/
        │       ├── agent.json          # Agent metadata and state
        │       └── messages/
        │           ├── message_<message_id>.json
        │           └── message_<message_id>.json
        └── multi_agents/               # Multi-agent storage
            └── multi_agent_<orchestrator_id>/
                └── multi_agent.json    # Orchestrator state and configuration
    ```

=== "TypeScript"

    Snapshots are stored in S3 with the following key structure:

    ```
    <prefix>/
    └── <session_id>/
        └── scopes/
            └── agent/
                └── <agent_id>/
                    └── snapshots/
                        ├── manifest.json
                        ├── snapshot_latest.json
                        └── immutable_history/
                            ├── snapshot_00001.json
                            ├── snapshot_00002.json
                            └── ...
    ```

#### Required S3 Permissions

To use the [`S3SessionManager`](../../../api-reference/python/session/s3_session_manager.md#strands.session.s3_session_manager.S3SessionManager), your AWS credentials must have the following S3 permissions:

- `s3:PutObject` - To create and update session data
- `s3:GetObject` - To retrieve session data
- `s3:DeleteObject` - To delete session data
- `s3:ListBucket` - To list objects in the bucket

Here's a sample IAM policy that grants these permissions for a specific bucket:

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "s3:PutObject",
                "s3:GetObject",
                "s3:DeleteObject"
            ],
            "Resource": "arn:aws:s3:::my-agent-sessions/*"
        },
        {
            "Effect": "Allow",
            "Action": "s3:ListBucket",
            "Resource": "arn:aws:s3:::my-agent-sessions"
        }
    ]
}
```

## How Session Management Works

=== "Python"

    The session management system in Strands Agents works through a combination of events, repositories, and data models:

    ### 1. Session Persistence Triggers

    Session persistence is automatically triggered by several key events in the agent and multi-agent lifecycle:

    **Single Agent Events**

    - **Agent Initialization**: When an agent is created with a session manager, it automatically restores any existing state and messages from the session.
    - **Message Addition**: When a new message is added to the conversation, it's automatically persisted to the session.
    - **Agent Invocation**: After each agent invocation, the agent state is synchronized with the session to capture any updates.
    - **Message Redaction**: When sensitive information needs to be redacted, the session manager can replace the original message with a redacted version while maintaining conversation flow.

    **Multi-Agent Events:**

    - **Multi-Agent Initialization**:  When an orchestrator is created with a session manager, it automatically restores state from the session.
    - **Node Execution**: After each node invocation, synchronizes orchestrator state after node transitions
    - **Multi-Agent Invocation**: After multiagent finished, captures final orchestrator state after execution

    !!! warning "After initializing the agent, direct modifications to `agent.messages` will not be persisted. Utilize the [Conversation Manager](./conversation-management.md) to help manage context of the agent in a way that can be persisted."

=== "TypeScript"

    The TypeScript SDK uses a **snapshot-based** model. A snapshot captures the full agent state at a point in time — messages, state, and system prompt — and can be restored later.

    ### Snapshot Types

    There are two kinds of snapshots:

    - **`snapshot_latest`**: A single mutable snapshot overwritten on every save. Used for crash recovery and resuming the most recent session.
    - **History snapshots**: Numbered snapshots (`snapshot_00001.json`, `snapshot_00002.json`, …) written to `immutable_history/`. Snapshot IDs start at `1`; ID `0` is reserved for `snapshot_latest`. Within a single linear session these snapshots are never overwritten, but restoring from a past snapshot via `loadSnapshotId` and continuing will overwrite any history beyond that point — effectively branching the timeline.

    ### Persistence Triggers

    `SessionManager` hooks into the agent lifecycle via three events:

    - **`InitializedEvent`**: Restores a snapshot into the agent on startup. If `loadSnapshotId` is set, the specific immutable snapshot is loaded and the manifest is advanced past it so future snapshots don't overwrite it. Otherwise, `snapshot_latest` is loaded if it exists.
    - **`MessageAddedEvent`**: Saves `snapshot_latest` after each message when `saveLatestOn: 'message'` (the default).
    - **`AfterInvocationEvent`**: Saves `snapshot_latest` after each invocation when `saveLatestOn: 'invocation'`, and evaluates the `snapshotTrigger` callback to decide whether to also write an immutable snapshot.

    ### `saveLatestOn` Strategy

    | Value | When `snapshot_latest` is saved |
    |---|---|
    | `'message'` (default) | After every message added to the conversation |
    | `'invocation'` | After every agent invocation completes |
    | `'never'` | Only when a `snapshotTrigger` fires |

    ### History Snapshots with `snapshotTrigger`

    The `snapshotTrigger` callback is evaluated after every invocation. When it returns `true`, a history snapshot is written and `snapshot_latest` is also updated atomically:

    ```typescript
    import { Agent, SessionManager } from '@strands-agents/sdk'

    const sessionManager = new SessionManager({
      sessionId: 'my-session',
      // Snapshot every 5 turns
      snapshotTrigger: ({ turnCount }) => turnCount % 5 === 0,
    })
    ```

    The callback receives a `SnapshotTriggerParams` object:

    | Field | Type | Description |
    |---|---|---|
    | `turnCount` | `number` | Number of invocations since session started |
    | `lastSnapshotAt` | `number \| undefined` | Timestamp (ms) of last immutable snapshot, `undefined` if none yet |
    | `agentData` | `{ messages, state }` | Current agent messages and state |

    ```typescript
    // Snapshot every 60 seconds
    snapshotTrigger: ({ lastSnapshotAt }) =>
      lastSnapshotAt !== undefined && Date.now() - lastSnapshotAt > 60_000

    // Snapshot when conversation exceeds 20 messages
    snapshotTrigger: ({ agentData }) => agentData.messages.length > 20
    ```

    ### Restoring from a Specific Snapshot

    To resume from a specific immutable snapshot, pass `loadSnapshotId` to `SessionManager`. The manifest is automatically advanced to `loadSnapshotId + 1`, so new snapshots continue from that point. Any immutable snapshots with IDs greater than `loadSnapshotId` will be overwritten by subsequent saves — this is a destructive branch, not a copy.

    !!! warning "Restoring from a snapshot and continuing the session will overwrite immutable snapshots that existed beyond that point."

    ```typescript
    import { Agent, SessionManager, S3Storage } from '@strands-agents/sdk'

    const storage = new S3Storage({ bucket: 'my-agent-sessions', region: 'us-east-1' })

    // Resume from snapshot 3 — new snapshots will start at ID 4
    const sessionManager = new SessionManager({
      sessionId: 'my-session',
      storage: { snapshot: storage },
      loadSnapshotId: '3',
    })

    const agent = new Agent({ sessionManager })
    await agent.invoke('Continue from where we left off.')
    ```

## Third-Party Session Managers

The following third-party session managers extend Strands with additional storage and memory capabilities:

| Session Manager | Provider | Description | Documentation |
|-----------------|----------|-------------|---------------|
| AgentCoreMemorySessionManager | Amazon | Advanced memory with intelligent retrieval using Amazon Bedrock AgentCore Memory. Supports both short-term memory (STM) and long-term memory (LTM) with strategies for user preferences, facts, and session summaries. | [View Documentation](../../../community/session-managers/agentcore-memory.md) |
| **Contribute Your Own** | Community | Have you built a session manager? Share it with the community! | [Learn How](../../../community/community-packages.md) |

## Custom Storage Backends (TypeScript)

In the TypeScript SDK, you can implement a custom storage backend by implementing the `SnapshotStorage` interface:

```typescript
import type { SnapshotStorage, SnapshotLocation } from '@strands-agents/sdk'
import type { Snapshot, SnapshotManifest } from '@strands-agents/sdk'

class RedisStorage implements SnapshotStorage {
  async saveSnapshot(params: {
    location: SnapshotLocation
    snapshotId: string
    isLatest: boolean
    snapshot: Snapshot
  }): Promise<void> {
    // Write to Redis
  }

  async loadSnapshot(params: { location: SnapshotLocation; snapshotId?: string }): Promise<Snapshot | null> {
    // Read from Redis
  }

  async listSnapshotIds(params: { location: SnapshotLocation }): Promise<string[]> {
    // List immutable snapshot IDs
  }

  async loadManifest(params: { location: SnapshotLocation }): Promise<SnapshotManifest> {
    // Load or return default manifest
  }

  async saveManifest(params: { location: SnapshotLocation; manifest: SnapshotManifest }): Promise<void> {
    // Persist manifest
  }
}

const sessionManager = new SessionManager({
  sessionId: 'my-session',
  storage: { snapshot: new RedisStorage() },
})
```

## Custom Session Repositories (Python)

For advanced use cases, you can implement your own session storage backend by creating a custom session repository:

```python
from typing import Optional
from strands import Agent
from strands.session.repository_session_manager import RepositorySessionManager
from strands.session.session_repository import SessionRepository
from strands.types.session import Session, SessionAgent, SessionMessage

class CustomSessionRepository(SessionRepository):
    """Custom session repository implementation."""
    
    def __init__(self):
        """Initialize with your custom storage backend."""
        # Initialize your storage backend (e.g., database connection)
        self.db = YourDatabaseClient()
    
    def create_session(self, session: Session) -> Session:
        """Create a new session."""
        self.db.sessions.insert(asdict(session))
        return session
    
    def read_session(self, session_id: str) -> Optional[Session]:
        """Read a session by ID."""
        data = self.db.sessions.find_one({"session_id": session_id})
        if data:
            return Session.from_dict(data)
        return None
    
    # Implement other required methods...
    # create_agent, read_agent, update_agent
    # create_message, read_message, update_message, list_messages

# Use your custom repository with RepositorySessionManager
custom_repo = CustomSessionRepository()
session_manager = RepositorySessionManager(
    session_id="user-789",
    session_repository=custom_repo
)

agent = Agent(session_manager=session_manager)
```

This approach allows you to store session data in any backend system while leveraging the built-in session management logic.

## Session Persistence Best Practices

- **Use Unique Session IDs**: Generate unique session IDs for each user or conversation context to prevent data overlap.

- **Session Cleanup**: Implement a strategy for cleaning up old or inactive sessions. Consider adding TTL (Time To Live) for sessions in production environments.

- **Understand Persistence Triggers**: Remember that changes to agent state or messages are only persisted during specific lifecycle events.

- **Concurrent Access**: Session managers are not thread-safe; use appropriate locking for concurrent access.

- **(TypeScript) Pre-create S3 buckets**: `S3Storage` does not create the bucket automatically. Ensure the bucket exists before using it.

- **(TypeScript) Choose `saveLatestOn` deliberately**: `'message'` (default) gives the finest-grained crash recovery but generates the most writes. Use `'invocation'` or `'never'` with a `snapshotTrigger` to reduce storage I/O for high-throughput agents.
