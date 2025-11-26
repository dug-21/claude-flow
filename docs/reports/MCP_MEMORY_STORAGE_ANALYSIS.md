# MCP Memory Storage Analysis Report

**Date:** 2025-11-26
**Analysis Scope:** All memory-related MCP tools and their storage backends
**Project:** claude-flow

---

## Executive Summary

This report documents all memory-related MCP endpoints in claude-flow, tracing each tool to its underlying storage implementation. The analysis reveals **three distinct memory storage systems** with varying levels of consistency between MCP tools and CLI commands.

### Key Findings

1. **Multiple Storage Backends:** The codebase uses at least 3 different storage backends for memory operations
2. **MCP-CLI Consistency:** Some MCP tools share storage with CLI commands, others do not
3. **Storage Locations:** Memory data is stored in 4+ different file paths depending on the backend
4. **Fallback Strategy:** SQLite is preferred but falls back to JSON/in-memory when unavailable

---

## 1. MCP Memory Tools Inventory

### 1.1 Primary Memory Tools (mcp-server.js)

Located in `/workspaces/claude-flow/src/mcp/mcp-server.js`

#### `memory_usage`
- **MCP Tool Name:** `memory_usage`
- **Line:** 225-239
- **Description:** Store/retrieve persistent memory with TTL and namespacing
- **Actions:** `store`, `retrieve`, `list`, `delete`, `search`
- **Handler:** `handleMemoryUsage()` (line 2340)

#### `memory_search`
- **MCP Tool Name:** `memory_search`
- **Line:** 240-252
- **Description:** Search memory with patterns
- **Parameters:** `pattern`, `namespace`, `limit`
- **Handler:** Routes through `memory_usage` handler

---

## 2. Storage Backend Architecture

### 2.1 MCP Tools Storage Path

```
MCP Tool: memory_usage
    ↓
Handler: handleMemoryUsage() [mcp-server.js:2340]
    ↓
Storage: this.memoryStore [mcp-server.js:68]
    ↓
Import: memoryStore from 'fallback-store.js' [mcp-server.js:13]
    ↓
Backend: FallbackMemoryStore (singleton)
```

**Code Reference:**
```javascript
// src/mcp/mcp-server.js:13
import { memoryStore } from '../memory/fallback-store.js';

// src/mcp/mcp-server.js:68
this.memoryStore = memoryStore; // Singleton instance

// src/mcp/mcp-server.js:2340-2376
async handleMemoryUsage(args) {
  if (!this.memoryStore) {
    return { success: false, error: 'Shared memory system not initialized' };
  }

  switch (args.action) {
    case 'store':
      const storeResult = await this.memoryStore.store(args.key, args.value, {
        namespace: args.namespace || 'default',
        ttl: args.ttl,
        metadata: { sessionId: this.sessionId, storedBy: 'mcp-server', type: 'knowledge' }
      });
      return {
        storage_type: this.memoryStore.isUsingFallback() ? 'in-memory' : 'sqlite',
        // ...
      };
  }
}
```

### 2.2 FallbackMemoryStore Implementation

**File:** `/workspaces/claude-flow/src/memory/fallback-store.js`

**Singleton Export (Line 131):**
```javascript
// Export a singleton instance for MCP server
export const memoryStore = new FallbackMemoryStore();
```

**Initialization Logic (Lines 19-64):**
```javascript
async initialize() {
  // 1. Check if SQLite is available
  const sqliteAvailable = await isSQLiteAvailable();

  if (!sqliteAvailable) {
    // Use in-memory store directly
    this.fallbackStore = new InMemoryStore(this.options);
    await this.fallbackStore.initialize();
    this.useFallback = true;
    return;
  }

  // 2. Try to initialize SQLite store
  try {
    this.primaryStore = new SqliteMemoryStore(this.options);
    await this.primaryStore.initialize();
    this.useFallback = false;
  } catch (error) {
    // 3. Fall back to in-memory store
    this.fallbackStore = new InMemoryStore(this.options);
    await this.fallbackStore.initialize();
    this.useFallback = true;
  }
}
```

**Active Store Selection (Line 82-84):**
```javascript
get activeStore() {
  return this.useFallback ? this.fallbackStore : this.primaryStore;
}
```

### 2.3 SQLite Backend (Primary)

**File:** `/workspaces/claude-flow/src/memory/sqlite-store.js`

**Storage Location (Lines 34-38):**
```javascript
_getMemoryDirectory() {
  // Always use .swarm directory in the project root
  // This ensures consistency whether running locally or via npx
  return getSwarmDir(); // Returns: /workspaces/claude-flow/.swarm/
}
```

**Database Path:**
```
/workspaces/claude-flow/.swarm/memory.db
```

**Table Schema (Lines 84-99):**
```sql
CREATE TABLE IF NOT EXISTS memory_entries (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  key TEXT NOT NULL,
  value TEXT NOT NULL,
  namespace TEXT NOT NULL DEFAULT 'default',
  metadata TEXT,
  created_at INTEGER DEFAULT (strftime('%s', 'now')),
  updated_at INTEGER DEFAULT (strftime('%s', 'now')),
  accessed_at INTEGER DEFAULT (strftime('%s', 'now')),
  access_count INTEGER DEFAULT 0,
  ttl INTEGER,
  expires_at INTEGER,
  UNIQUE(key, namespace)
);
```

### 2.4 In-Memory Backend (Fallback)

**File:** `/workspaces/claude-flow/src/memory/in-memory-store.js`

**Storage Type:** JavaScript Map (volatile, non-persistent)

**Used When:**
- SQLite native module fails to load (common in npx environments)
- Running on Windows without proper build tools
- SQLite initialization fails for any reason

---

## 3. CLI Memory Commands Storage Path

### 3.1 CLI Commands Architecture

**File:** `/workspaces/claude-flow/src/cli/commands/memory.ts`

**Command Structure (Lines 284-577):**
```typescript
export const memoryCommand = new Command()
  .name('memory')
  .description('Manage persistent memory with AgentDB integration')

memoryCommand
  .command('store')
  .action(async (key: string, value: string, options: any) => {
    const memory = new UnifiedMemoryManager(); // Creates NEW instance
    await memory.store(key, value, options.namespace);
  });
```

### 3.2 UnifiedMemoryManager (CLI-Specific)

**File:** `/workspaces/claude-flow/src/cli/commands/memory.ts` (Lines 35-176)

**Backend Selection (Lines 40-64):**
```typescript
async getBackend(): Promise<MemoryBackend> {
  if (this.backend === 'sqlite' && !this.sqliteManager) {
    try {
      // Try ReasoningBank/AgentDB SQLite backend
      const { initializeReasoningBank, storeMemory, queryMemories, listMemories, getStatus } =
        await import('../../reasoningbank/reasoningbank-adapter.js');

      await initializeReasoningBank();
      this.sqliteManager = { storeMemory, queryMemories, listMemories, getStatus };
      console.log('🗄️  Using SQLite backend (.swarm/memory.db)');
      return 'sqlite';
    } catch (error) {
      console.log('⚠️  SQLite unavailable, falling back to JSON');
      this.backend = 'json';
    }
  }

  if (this.backend === 'json' && !this.jsonManager) {
    this.jsonManager = new SimpleMemoryManager();
    console.log('📄 Using JSON backend (./memory/memory-store.json)');
  }

  return this.backend;
}
```

### 3.3 CLI JSON Backend

**File:** `/workspaces/claude-flow/src/cli/commands/memory.ts` (Lines 178-282)

**Storage Location (Line 179):**
```typescript
private filePath = './memory/memory-store.json';
```

**Database Path:**
```
/workspaces/claude-flow/memory/memory-store.json
```

### 3.4 CLI SQLite Backend (ReasoningBank)

**File:** `/workspaces/claude-flow/src/reasoningbank/reasoningbank-adapter.js`

**Backend:** agentic-flow@1.5.13 Node.js backend with SQLite

**Storage Strategy (Lines 97-148):**
```javascript
export async function storeMemory(key, value, options = {}) {
  // Map our memory model to ReasoningBank pattern model
  const memory = {
    id: memoryId,
    type: 'reasoning_memory',
    pattern_data: {
      title: key,
      content: value,
      domain: options.namespace || 'default',
      agent: options.agent || 'memory-agent',
      // ...
    },
    confidence: options.confidence || 0.8,
    usage_count: 0
  };

  // Store memory using Node.js backend
  ReasoningBank.db.upsertMemory(memory);

  // Generate and store embedding for semantic search
  const embedding = await ReasoningBank.computeEmbedding(value);
  ReasoningBank.db.upsertEmbedding({
    id: memoryId,
    model: 'text-embedding-3-small',
    dims: embedding.length,
    vector: embedding
  });
}
```

**Database Path:** (Managed by agentic-flow library)
- Likely: `.swarm/reasoningbank.db` or similar

---

## 4. Legacy/Additional Memory Tools

### 4.1 Claude-Flow Tools (claude-flow-tools.ts)

**File:** `/workspaces/claude-flow/src/mcp/claude-flow-tools.ts`

**Memory Tools (Lines 68-73):**
- `memory/query` (Line 554) - Query agent memory with filters
- `memory/store` (Line 636) - Store a new memory entry
- `memory/delete` (Line 706) - Delete a memory entry
- `memory/export` (Line 738) - Export memory to file
- `memory/import` (Line 794) - Import memory from file

**Storage Path:**
```
MCP Tool: memory/query, memory/store, etc.
    ↓
Handler: Uses context.orchestrator [claude-flow-tools.ts:605-634]
    ↓
Backend: Orchestrator's memory system (separate from memoryStore)
```

**Code Reference:**
```typescript
// Lines 605-634
handler: async (input: any, context?: ClaudeFlowToolContext) => {
  if (!context?.orchestrator) {
    throw new Error('Orchestrator not available');
  }

  const query = {
    agentId: input.agentId,
    sessionId: input.sessionId,
    type: input.type,
    // ...
  };

  const entries = await context.orchestrator.queryMemory(query);
  // ...
}
```

**Storage Backend:** Unknown (depends on orchestrator implementation)

### 4.2 Swarm Tools (swarm-tools.ts)

**File:** `/workspaces/claude-flow/src/mcp/swarm-tools.ts`

**Legacy Tools (Lines 731-763):**
- `memory_store` (Line 732) - Legacy export
- `memory_retrieve` (Line 751) - Legacy export

**Status:** These are **legacy tool definitions** only. The actual handlers are not implemented in this file.

---

## 5. Storage Location Summary

### 5.1 File Paths by Backend

| Backend | File Path | Used By | Persistence |
|---------|-----------|---------|-------------|
| **MCP SQLite** | `/workspaces/claude-flow/.swarm/memory.db` | MCP `memory_usage` tool | ✅ Persistent |
| **MCP In-Memory** | (RAM only) | MCP `memory_usage` fallback | ❌ Volatile |
| **CLI JSON** | `/workspaces/claude-flow/memory/memory-store.json` | CLI `memory store` command | ✅ Persistent |
| **CLI SQLite (ReasoningBank)** | `.swarm/reasoningbank.db` (likely) | CLI `memory store` (SQLite mode) | ✅ Persistent |
| **Orchestrator Memory** | Unknown | `memory/query` MCP tool | Unknown |
| **SharedMemory** | `/workspaces/claude-flow/.swarm/memory.db` (SharedMemory class) | Various internal | ✅ Persistent |
| **Hive-Mind Memory** | `/workspaces/claude-flow/.hive-mind/memory.db` | Hive-mind features | ✅ Persistent |

**Current State (2025-11-26):**
```bash
$ ls -lh /workspaces/claude-flow/.swarm/
total 136K
-rw-r--r-- 1 vscode vscode 132K Nov 26 18:16 memory.db
```

### 5.2 Storage Consistency Matrix

| Operation | MCP Tool Storage | CLI Command Storage | Consistent? |
|-----------|------------------|---------------------|-------------|
| **Store** | `.swarm/memory.db` (SQLite) OR in-memory | `./memory/memory-store.json` (JSON) OR `.swarm/reasoningbank.db` (SQLite) | ❌ Different paths |
| **Retrieve** | `.swarm/memory.db` (SQLite) OR in-memory | `./memory/memory-store.json` (JSON) OR `.swarm/reasoningbank.db` (SQLite) | ❌ Different paths |
| **List** | `.swarm/memory.db` (SQLite) OR in-memory | `./memory/memory-store.json` (JSON) OR `.swarm/reasoningbank.db` (SQLite) | ❌ Different paths |
| **Search** | `.swarm/memory.db` (SQLite) OR in-memory | N/A (uses query) | ❌ Not available in CLI |

---

## 6. Code References

### 6.1 MCP Memory Tool Registration

**File:** `/workspaces/claude-flow/src/mcp/mcp-server.js`

**Tool Definition (Lines 225-252):**
```javascript
memory_usage: {
  name: 'memory_usage',
  description: 'Store/retrieve persistent memory with TTL and namespacing',
  inputSchema: {
    type: 'object',
    properties: {
      action: { type: 'string', enum: ['store', 'retrieve', 'list', 'delete', 'search'] },
      key: { type: 'string' },
      value: { type: 'string' },
      namespace: { type: 'string', default: 'default' },
      ttl: { type: 'number' },
    },
    required: ['action'],
  },
},
```

**Handler Routing (Lines 1617-1618):**
```javascript
case 'memory_usage':
  return await this.handleMemoryUsage(args);
```

**Handler Implementation (Lines 2340-2450):**
```javascript
async handleMemoryUsage(args) {
  if (!this.memoryStore) {
    return {
      success: false,
      error: 'Shared memory system not initialized',
      timestamp: new Date().toISOString(),
    };
  }

  try {
    switch (args.action) {
      case 'store':
        const storeResult = await this.memoryStore.store(args.key, args.value, {
          namespace: args.namespace || 'default',
          ttl: args.ttl,
          metadata: {
            sessionId: this.sessionId,
            storedBy: 'mcp-server',
            type: 'knowledge',
          },
        });

        return {
          success: true,
          action: 'store',
          key: args.key,
          namespace: args.namespace || 'default',
          stored: true,
          size: storeResult.size || args.value.length,
          id: storeResult.id,
          storage_type: this.memoryStore.isUsingFallback() ? 'in-memory' : 'sqlite',
          timestamp: new Date().toISOString(),
        };

      case 'retrieve':
        const value = await this.memoryStore.retrieve(args.key, {
          namespace: args.namespace || 'default',
        });

        return {
          success: true,
          action: 'retrieve',
          key: args.key,
          value: value,
          found: value !== null,
          namespace: args.namespace || 'default',
          storage_type: this.memoryStore.isUsingFallback() ? 'in-memory' : 'sqlite',
          timestamp: new Date().toISOString(),
        };

      case 'list':
        const entries = await this.memoryStore.list({
          namespace: args.namespace || 'default',
          limit: 100,
        });

        return {
          success: true,
          action: 'list',
          namespace: args.namespace || 'default',
          entries: entries,
          count: entries.length,
          storage_type: this.memoryStore.isUsingFallback() ? 'in-memory' : 'sqlite',
          timestamp: new Date().toISOString(),
        };

      case 'delete':
        const deleted = await this.memoryStore.delete(args.key, {
          namespace: args.namespace || 'default',
        });

        return {
          success: true,
          action: 'delete',
          key: args.key,
          namespace: args.namespace || 'default',
          deleted: deleted,
          storage_type: this.memoryStore.isUsingFallback() ? 'in-memory' : 'sqlite',
          timestamp: new Date().toISOString(),
        };

      case 'search':
        const results = await this.memoryStore.search(args.value || '', {
          namespace: args.namespace || 'default',
          limit: args.limit || 10,
        });

        return {
          success: true,
          action: 'search',
          pattern: args.value,
          namespace: args.namespace || 'default',
          results: results,
          count: results.length,
          storage_type: this.memoryStore.isUsingFallback() ? 'in-memory' : 'sqlite',
          timestamp: new Date().toISOString(),
        };

      default:
        return {
          success: false,
          error: `Unknown action: ${args.action}`,
          timestamp: new Date().toISOString(),
        };
    }
  } catch (error) {
    console.error('[memory_usage] Error:', error);
    return {
      success: false,
      error: error.message,
      timestamp: new Date().toISOString(),
    };
  }
}
```

### 6.2 CLI Memory Command Implementation

**File:** `/workspaces/claude-flow/src/cli/commands/memory.ts`

**Store Command (Lines 292-311):**
```typescript
memoryCommand
  .command('store')
  .description('Store information in memory (uses SQLite by default)')
  .arguments('<key> <value>')
  .option('-n, --namespace <namespace>', 'Target namespace', 'default')
  .action(async (key: string, value: string, options: any) => {
    try {
      const memory = new UnifiedMemoryManager(); // NEW instance per command
      const result = await memory.store(key, value, options.namespace);
      console.log(chalk.green('✅ Stored successfully'));
      console.log(`📝 Key: ${key}`);
      console.log(`📦 Namespace: ${options.namespace}`);
      console.log(`💾 Size: ${new TextEncoder().encode(value).length} bytes`);
      if (result.id) {
        console.log(chalk.gray(`🆔 ID: ${result.id}`));
      }
    } catch (error) {
      console.error(chalk.red('❌ Failed to store:'), (error as Error).message);
    }
  });
```

**Query Command (Lines 314-351):**
```typescript
memoryCommand
  .command('query')
  .description('Search memory entries (semantic search with SQLite)')
  .arguments('<search>')
  .option('-n, --namespace <namespace>', 'Filter by namespace')
  .option('-l, --limit <limit>', 'Limit results', '10')
  .action(async (search: string, options: any) => {
    try {
      const memory = new UnifiedMemoryManager();
      const results = await memory.query(search, options.namespace, parseInt(options.limit));

      if (results.length === 0) {
        console.log(chalk.yellow('⚠️  No results found'));
        return;
      }

      console.log(chalk.green(`✅ Found ${results.length} results:\n`));

      for (const entry of results) {
        console.log(chalk.blue(`📌 ${entry.key}`));
        console.log(`   Namespace: ${entry.namespace}`);
        console.log(`   Value: ${entry.value.substring(0, 100)}${entry.value.length > 100 ? '...' : ''}`);
        const timestamp = entry.created_at || entry.timestamp;
        if (timestamp) {
          const date = typeof timestamp === 'number' ? new Date(timestamp) : new Date(timestamp);
          console.log(`   Stored: ${date.toLocaleString()}`);
        }
        if (entry.confidence) {
          console.log(chalk.gray(`   Confidence: ${(entry.confidence * 100).toFixed(0)}%`));
        }
        console.log('');
      }
    } catch (error) {
      console.error(chalk.red('❌ Failed to query:'), (error as Error).message);
    }
  });
```

---

## 7. Consistency Analysis

### 7.1 MCP vs CLI Storage Consistency

**Problem:** MCP tools and CLI commands use **different storage backends**:

- **MCP Tools:** Use `FallbackMemoryStore` singleton
  - Primary: `.swarm/memory.db` (SQLite with `memory_entries` table)
  - Fallback: In-memory (volatile)

- **CLI Commands:** Use `UnifiedMemoryManager` (new instance per command)
  - Primary: ReasoningBank adapter (SQLite with `reasoning_memory` pattern)
  - Fallback: `./memory/memory-store.json` (JSON)

**Impact:**
- ❌ Data stored via MCP tools is **not visible** to CLI commands
- ❌ Data stored via CLI commands is **not visible** to MCP tools
- ❌ No shared state between interfaces

**Example:**
```bash
# Store via MCP tool
mcp__claude-flow__memory_usage({ action: "store", key: "test", value: "data" })
# Stores to: .swarm/memory.db (memory_entries table)

# Query via CLI
npx claude-flow memory query "test"
# Queries: ./memory/memory-store.json OR reasoningbank adapter
# Result: ❌ NOT FOUND (different storage)
```

### 7.2 Table Schema Differences

**MCP SQLite Schema (sqlite-store.js):**
```sql
CREATE TABLE IF NOT EXISTS memory_entries (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  key TEXT NOT NULL,
  value TEXT NOT NULL,
  namespace TEXT NOT NULL DEFAULT 'default',
  metadata TEXT,
  created_at INTEGER,
  updated_at INTEGER,
  accessed_at INTEGER,
  access_count INTEGER,
  ttl INTEGER,
  expires_at INTEGER,
  UNIQUE(key, namespace)
);
```

**CLI ReasoningBank Schema (reasoningbank-adapter.js):**
```javascript
const memory = {
  id: memoryId,
  type: 'reasoning_memory',
  pattern_data: {
    title: key,              // Key mapped to title
    content: value,          // Value mapped to content
    domain: namespace,       // Namespace mapped to domain
    agent: 'memory-agent',
    task_type: 'fact',
    // ...
  },
  confidence: 0.8,
  usage_count: 0
};
```

**Differences:**
- ❌ Different table structures
- ❌ Different field mappings
- ❌ Different query mechanisms
- ✅ Similar namespace concept

---

## 8. Recommendations

### 8.1 Critical Issues

1. **Storage Inconsistency**
   - **Issue:** MCP and CLI use different databases
   - **Impact:** User confusion, data silos, poor UX
   - **Priority:** HIGH
   - **Recommendation:** Unify storage backend

2. **Multiple Memory Systems**
   - **Issue:** 3+ memory implementations (FallbackMemoryStore, UnifiedMemoryManager, SharedMemory)
   - **Impact:** Code complexity, maintenance burden
   - **Priority:** MEDIUM
   - **Recommendation:** Consolidate to single implementation

3. **Schema Incompatibility**
   - **Issue:** Different table structures for same conceptual data
   - **Impact:** Cannot migrate data between systems
   - **Priority:** MEDIUM
   - **Recommendation:** Standardize schema

### 8.2 Proposed Solutions

#### Option 1: Unified Storage Adapter (Recommended)

Create a single memory adapter that both MCP and CLI use:

```typescript
// src/memory/unified-adapter.ts
export class UnifiedMemoryAdapter {
  constructor() {
    // Use SAME storage as MCP server
    this.store = new FallbackMemoryStore();
  }

  async store(key, value, namespace) {
    return this.store.store(key, value, { namespace });
  }

  async query(search, namespace) {
    return this.store.search(search, { namespace });
  }
}

// CLI commands use unified adapter
const memory = new UnifiedMemoryAdapter();

// MCP server already uses FallbackMemoryStore singleton
```

**Benefits:**
- ✅ Single source of truth
- ✅ Data visible across MCP and CLI
- ✅ Minimal code changes
- ✅ Backward compatible

#### Option 2: Bridge Pattern

Create a bridge between existing systems:

```typescript
// src/memory/storage-bridge.ts
export class StorageBridge {
  async syncMCPToCLI() {
    // Copy data from .swarm/memory.db to CLI backend
  }

  async syncCLIToMCP() {
    // Copy data from CLI backend to .swarm/memory.db
  }
}
```

**Benefits:**
- ✅ Preserves both systems
- ✅ Gradual migration
- ❌ Complex synchronization logic

#### Option 3: Deprecate and Replace

1. Deprecate `UnifiedMemoryManager` in CLI
2. Use `FallbackMemoryStore` everywhere
3. Update ReasoningBank to read from `memory_entries` table

**Benefits:**
- ✅ Clean architecture
- ✅ Single storage system
- ❌ Breaking change for CLI users

### 8.3 Implementation Plan

**Phase 1: Audit** (1 day)
- [ ] Document all memory operations in codebase
- [ ] Identify all storage access points
- [ ] Create migration path

**Phase 2: Unification** (2-3 days)
- [ ] Create `UnifiedMemoryAdapter` class
- [ ] Update CLI commands to use `FallbackMemoryStore`
- [ ] Add compatibility layer for existing data

**Phase 3: Testing** (1-2 days)
- [ ] Unit tests for unified adapter
- [ ] Integration tests for MCP + CLI consistency
- [ ] Migration tests for existing data

**Phase 4: Documentation** (1 day)
- [ ] Update memory command docs
- [ ] Add migration guide
- [ ] Update MCP tool descriptions

**Total Estimate:** 5-7 days

---

## 9. Appendix

### 9.1 File References

**MCP Server:**
- `/workspaces/claude-flow/src/mcp/mcp-server.js` - Main MCP server implementation
- `/workspaces/claude-flow/src/mcp/claude-flow-tools.ts` - Advanced memory tools
- `/workspaces/claude-flow/src/mcp/swarm-tools.ts` - Legacy memory tool exports

**Memory Backends:**
- `/workspaces/claude-flow/src/memory/fallback-store.js` - MCP memory singleton
- `/workspaces/claude-flow/src/memory/sqlite-store.js` - SQLite implementation
- `/workspaces/claude-flow/src/memory/in-memory-store.js` - Volatile fallback
- `/workspaces/claude-flow/src/memory/shared-memory.js` - Alternative SQLite implementation
- `/workspaces/claude-flow/src/memory/index.js` - Memory module exports

**CLI Commands:**
- `/workspaces/claude-flow/src/cli/commands/memory.ts` - CLI memory commands
- `/workspaces/claude-flow/src/reasoningbank/reasoningbank-adapter.js` - ReasoningBank integration

**Utilities:**
- `/workspaces/claude-flow/src/memory/sqlite-wrapper.js` - SQLite availability detection
- `/workspaces/claude-flow/src/utils/project-root.js` - Path utilities (getSwarmDir)

### 9.2 Environment Factors

**NPX Limitations:**
- better-sqlite3 native module often fails in npx environments
- Falls back to in-memory store (data loss on exit)
- CLI falls back to JSON storage

**Local Installation:**
- better-sqlite3 usually works correctly
- Persistent SQLite storage in `.swarm/memory.db`
- CLI can use ReasoningBank with embeddings

**Windows:**
- May require build tools for better-sqlite3
- Fallback mechanisms more frequently triggered

---

## 10. Conclusion

The claude-flow memory system uses **multiple storage backends** with **inconsistent data access** between MCP tools and CLI commands. While each system works independently, they create user confusion and data silos.

**Key Takeaways:**

1. ✅ **MCP Memory Tools Work:** Store data persistently in `.swarm/memory.db`
2. ✅ **CLI Memory Commands Work:** Store data in `./memory/memory-store.json` or ReasoningBank
3. ❌ **Not Compatible:** Data stored in one system is invisible to the other
4. 🔧 **Fixable:** Unification possible with ~5-7 days of work

**Recommended Next Steps:**

1. Implement `UnifiedMemoryAdapter` (Option 1)
2. Update CLI commands to use `FallbackMemoryStore`
3. Add data migration utilities
4. Update documentation

This will provide a **consistent memory experience** across all interfaces while preserving backward compatibility.

---

**Report Generated:** 2025-11-26
**Analyst:** Claude (Sonnet 4.5)
**Project Version:** alpha/devcontainer branch
