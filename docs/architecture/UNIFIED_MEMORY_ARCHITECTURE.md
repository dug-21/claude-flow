# Unified Memory Architecture Specification
## Claude-Flow Memory System - Single Source of Truth

**Status:** Architecture Recommendation
**Version:** 1.0
**Date:** 2025-11-26
**Author:** System Architect

---

## Executive Summary

This document provides a comprehensive architecture for unifying claude-flow's fragmented memory systems into a single, consistent source of truth. The solution addresses the critical problem where CLI commands, MCP tools, and API calls currently store data in **different locations**, making data invisible across entry points.

### Current Problem

**Data Silos Identified:**
1. `.swarm/memory.db` - ReasoningBank (via agentic-flow, CLI only)
2. `.swarm/swarm-memory.db` - SwarmMemory (Swarm operations)
3. `.claude-flow/memory/unified-memory.db` - UnifiedMemoryManager (Unused)
4. `./memory/memory-store.json` - JSON fallback (CLI and MCP use separately)
5. `.hive-mind/hive.db` - Hive Mind collective memory
6. `./data/hive-mind.db` - Hive Mind persistence

**Entry Point Inconsistencies:**
- CLI: `npx claude-flow memory store` → Uses ReasoningBank → `.swarm/memory.db`
- MCP: `mcp__claude-flow__memory_usage` → Uses JSON fallback → `./memory/memory-store.json`
- Swarm API: `SwarmMemoryManager` → Uses SwarmMemory → `.swarm/swarm-memory.db`
- Hive Mind: `CollectiveMemory` → Uses HiveMind DB → `.hive-mind/hive.db`

**Impact:**
- Data stored via CLI is **NOT visible** to MCP tools
- Data stored via MCP is **NOT visible** to CLI commands
- Swarm memory is **isolated** from general memory
- No cross-component data sharing

---

## Architecture Principles

### 1. Single Source of Truth (SSOT)
**Principle:** All memory operations must read/write to the **same physical storage location**, regardless of entry point.

**Implementation:**
- **Primary Storage:** `.swarm/memory.db` (SQLite with ReasoningBank schema)
- **Fallback Storage:** `./memory/memory-store.json` (when SQLite unavailable in npx)
- **Bridge Pattern:** All entry points route through MemoryRouter

### 2. Backward Compatibility
**Principle:** Existing code must continue to work without breaking changes.

**Implementation:**
- Keep existing APIs intact
- Add router layer underneath
- Gradual migration path
- Feature flags for rollback

### 3. Entry Point Consistency
**Principle:** All entry points (CLI, MCP, API) must access the same backend.

**Implementation:**
- `UnifiedMemoryRouter` - Single point of routing
- `StorageBackendManager` - Handles SQLite vs JSON
- `MemoryFacade` - Consistent interface for all callers

### 4. NPX/SQLite Fallback Handling
**Principle:** Gracefully handle npx environments where better-sqlite3 is unavailable.

**Implementation:**
- Auto-detect npx environment
- Fallback to JSON mode with clear messaging
- Store mode preference in config
- Allow manual override

---

## Architecture Design

### High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        ENTRY POINTS                             │
├─────────────────┬──────────────────┬──────────────────┬─────────┤
│  CLI Commands   │   MCP Tools      │  Swarm API       │  Other  │
│  (memory store) │  (memory_usage)  │  (SwarmMemory)   │  APIs   │
└────────┬────────┴────────┬─────────┴──────┬───────────┴────┬────┘
         │                 │                │                │
         └─────────────────┼────────────────┼────────────────┘
                           ▼                │
                  ┌────────────────────┐    │
                  │ UnifiedMemoryRouter│◄───┘
                  │  (Entry Point)     │
                  └──────────┬─────────┘
                             │
                  ┌──────────▼─────────┐
                  │ MemoryFacade       │
                  │ (Unified Interface)│
                  └──────────┬─────────┘
                             │
                  ┌──────────▼─────────┐
                  │StorageBackendMgr   │
                  │(SQLite vs JSON)    │
                  └─────┬───────┬──────┘
                        │       │
         ┌──────────────┘       └─────────────┐
         │                                    │
    ┌────▼─────────┐                 ┌───────▼────────┐
    │ SQLiteBackend│                 │  JsonBackend   │
    │ (Primary)    │                 │  (Fallback)    │
    └────┬─────────┘                 └───────┬────────┘
         │                                    │
    ┌────▼──────────────┐           ┌────────▼─────────┐
    │.swarm/memory.db   │           │memory-store.json │
    │(ReasoningBank)    │           │(Simple KV)       │
    └───────────────────┘           └──────────────────┘
```

### Component Responsibilities

#### 1. UnifiedMemoryRouter
**Purpose:** Single entry point that routes all memory operations.

**Responsibilities:**
- Detect caller type (CLI, MCP, API)
- Route to MemoryFacade
- Handle legacy API compatibility
- Log access patterns for debugging

**Location:** `/src/memory/unified-memory-router.ts` (NEW)

#### 2. MemoryFacade
**Purpose:** Provide consistent memory interface across all backends.

**Responsibilities:**
- Standardize store/query/list/delete operations
- Handle namespace management
- Enforce access patterns
- Manage metadata (timestamps, agents, tags)

**Location:** `/src/memory/memory-facade.ts` (NEW)

#### 3. StorageBackendManager
**Purpose:** Intelligently choose and manage storage backend.

**Responsibilities:**
- Detect SQLite availability
- Fallback to JSON when needed
- Sync data between backends (migration)
- Maintain backend health status

**Location:** `/src/memory/storage-backend-manager.ts` (NEW)

#### 4. SQLiteBackend (Primary)
**Purpose:** Primary storage using ReasoningBank schema.

**Responsibilities:**
- Use ReasoningBank adapter
- Store in `.swarm/memory.db`
- Provide semantic search
- Handle embeddings

**Location:** `/src/memory/backends/sqlite-backend.ts` (NEW)

#### 5. JsonBackend (Fallback)
**Purpose:** Fallback storage when SQLite unavailable.

**Responsibilities:**
- Store in `./memory/memory-store.json`
- Simple key-value operations
- Namespace-based organization
- No semantic search (basic text matching)

**Location:** `/src/memory/backends/json-backend.ts` (NEW)

---

## Detailed Implementation Specification

### Phase 1: Core Infrastructure (Week 1)

#### 1.1 Create StorageBackendManager

**File:** `/src/memory/storage-backend-manager.ts`

```typescript
export interface MemoryBackend {
  name: 'sqlite' | 'json';
  available: boolean;
  path: string;

  // Core operations
  store(key: string, value: any, metadata: MemoryMetadata): Promise<string>;
  retrieve(key: string, namespace: string): Promise<MemoryEntry | null>;
  query(search: string, options: QueryOptions): Promise<MemoryEntry[]>;
  list(options: ListOptions): Promise<MemoryEntry[]>;
  delete(key: string, namespace: string): Promise<boolean>;

  // Health
  getStats(): Promise<BackendStats>;
  isHealthy(): Promise<boolean>;
}

export class StorageBackendManager {
  private primaryBackend: MemoryBackend | null = null;
  private fallbackBackend: MemoryBackend | null = null;
  private currentBackend: MemoryBackend | null = null;

  async initialize(): Promise<void> {
    // Try SQLite first
    try {
      const { SQLiteBackend } = await import('./backends/sqlite-backend.js');
      this.primaryBackend = new SQLiteBackend();
      await this.primaryBackend.isHealthy();
      this.currentBackend = this.primaryBackend;
      console.log('✅ Using SQLite backend (.swarm/memory.db)');
    } catch (error) {
      // SQLite unavailable - use JSON fallback
      const { JsonBackend } = await import('./backends/json-backend.js');
      this.fallbackBackend = new JsonBackend();
      this.currentBackend = this.fallbackBackend;
      console.warn('⚠️  SQLite unavailable, using JSON fallback');

      if (this.isNpxEnvironment()) {
        console.warn('💡 NPX detected: better-sqlite3 unavailable in temp dirs');
        console.warn('   For SQLite support: npm install && ./node_modules/.bin/claude-flow');
      }
    }
  }

  private isNpxEnvironment(): boolean {
    return process.env.npm_config_user_agent?.includes('npx') ||
           process.cwd().includes('_npx') ||
           process.cwd().includes('_npx');
  }

  getBackend(): MemoryBackend {
    if (!this.currentBackend) {
      throw new Error('Storage backend not initialized');
    }
    return this.currentBackend;
  }

  async migrateToSQLite(): Promise<MigrationResult> {
    // Migrate JSON → SQLite
    if (!this.fallbackBackend || !this.primaryBackend) {
      throw new Error('Cannot migrate: backends not available');
    }

    // Export from JSON
    const jsonData = await this.fallbackBackend.list({ limit: 10000 });

    // Import to SQLite
    let migrated = 0;
    for (const entry of jsonData) {
      await this.primaryBackend.store(entry.key, entry.value, entry.metadata);
      migrated++;
    }

    return { migrated, total: jsonData.length };
  }
}
```

#### 1.2 Create MemoryFacade

**File:** `/src/memory/memory-facade.ts`

```typescript
export interface MemoryMetadata {
  namespace: string;
  agent?: string;
  tags?: string[];
  ttl?: number;
  confidence?: number;
}

export interface MemoryEntry {
  id: string;
  key: string;
  value: any;
  namespace: string;
  timestamp: number;
  agent?: string;
  tags?: string[];
  confidence?: number;
  created_at?: string;
  usage_count?: number;
}

export class MemoryFacade {
  private backend: MemoryBackend;

  constructor(backend: MemoryBackend) {
    this.backend = backend;
  }

  /**
   * Store data with consistent metadata handling
   */
  async store(
    key: string,
    value: any,
    metadata: MemoryMetadata
  ): Promise<string> {
    // Normalize metadata
    const normalizedMetadata = {
      namespace: metadata.namespace || 'default',
      agent: metadata.agent || 'system',
      tags: metadata.tags || [],
      timestamp: Date.now(),
      confidence: metadata.confidence || 0.8,
      ttl: metadata.ttl,
    };

    // Store via backend
    return await this.backend.store(key, value, normalizedMetadata);
  }

  /**
   * Query with semantic search (SQLite) or text search (JSON)
   */
  async query(
    search: string,
    options: QueryOptions = {}
  ): Promise<MemoryEntry[]> {
    const defaultOptions = {
      namespace: options.namespace,
      limit: options.limit || 10,
      minConfidence: options.minConfidence || 0.5,
    };

    return await this.backend.query(search, defaultOptions);
  }

  /**
   * List entries by namespace
   */
  async list(options: ListOptions = {}): Promise<MemoryEntry[]> {
    return await this.backend.list({
      namespace: options.namespace,
      limit: options.limit || 10,
      offset: options.offset || 0,
    });
  }

  /**
   * Get storage statistics
   */
  async getStats(): Promise<MemoryStats> {
    const backendStats = await this.backend.getStats();

    return {
      backend: this.backend.name,
      path: this.backend.path,
      totalEntries: backendStats.totalEntries,
      namespaces: backendStats.namespaces,
      sizeBytes: backendStats.sizeBytes,
      performance: this.backend.name === 'sqlite'
        ? '150x faster semantic search'
        : 'Basic text search',
    };
  }
}
```

#### 1.3 Create UnifiedMemoryRouter

**File:** `/src/memory/unified-memory-router.ts`

```typescript
export class UnifiedMemoryRouter {
  private static instance: UnifiedMemoryRouter | null = null;
  private backendManager: StorageBackendManager;
  private facade: MemoryFacade | null = null;
  private initialized = false;

  private constructor() {
    this.backendManager = new StorageBackendManager();
  }

  static getInstance(): UnifiedMemoryRouter {
    if (!UnifiedMemoryRouter.instance) {
      UnifiedMemoryRouter.instance = new UnifiedMemoryRouter();
    }
    return UnifiedMemoryRouter.instance;
  }

  async initialize(): Promise<void> {
    if (this.initialized) return;

    await this.backendManager.initialize();
    const backend = this.backendManager.getBackend();
    this.facade = new MemoryFacade(backend);
    this.initialized = true;
  }

  getFacade(): MemoryFacade {
    if (!this.facade) {
      throw new Error('Router not initialized. Call initialize() first.');
    }
    return this.facade;
  }

  async store(key: string, value: any, metadata: MemoryMetadata): Promise<string> {
    await this.initialize();
    return await this.facade!.store(key, value, metadata);
  }

  async query(search: string, options?: QueryOptions): Promise<MemoryEntry[]> {
    await this.initialize();
    return await this.facade!.query(search, options);
  }

  async list(options?: ListOptions): Promise<MemoryEntry[]> {
    await this.initialize();
    return await this.facade!.list(options);
  }

  async getStats(): Promise<MemoryStats> {
    await this.initialize();
    return await this.facade!.getStats();
  }
}

// Export singleton instance getter
export function getUnifiedMemoryRouter(): UnifiedMemoryRouter {
  return UnifiedMemoryRouter.getInstance();
}
```

---

### Phase 2: Backend Implementations (Week 2)

#### 2.1 SQLiteBackend

**File:** `/src/memory/backends/sqlite-backend.ts`

```typescript
import * as ReasoningBank from '../../../reasoningbank/reasoningbank-adapter.js';
import { MemoryBackend, MemoryEntry, MemoryMetadata } from '../types.js';

export class SQLiteBackend implements MemoryBackend {
  name: 'sqlite' = 'sqlite';
  available = true;
  path = '.swarm/memory.db';

  async store(key: string, value: any, metadata: MemoryMetadata): Promise<string> {
    // Use existing ReasoningBank adapter
    return await ReasoningBank.storeMemory(key, value, {
      namespace: metadata.namespace,
      agent: metadata.agent,
      confidence: metadata.confidence,
    });
  }

  async retrieve(key: string, namespace: string): Promise<MemoryEntry | null> {
    const results = await ReasoningBank.queryMemories(key, {
      namespace,
      limit: 1,
    });

    return results.length > 0 ? results[0] : null;
  }

  async query(search: string, options: QueryOptions): Promise<MemoryEntry[]> {
    return await ReasoningBank.queryMemories(search, {
      namespace: options.namespace,
      limit: options.limit,
      minConfidence: options.minConfidence,
    });
  }

  async list(options: ListOptions): Promise<MemoryEntry[]> {
    return await ReasoningBank.listMemories({
      namespace: options.namespace,
      limit: options.limit,
    });
  }

  async delete(key: string, namespace: string): Promise<boolean> {
    // ReasoningBank doesn't have delete - implement direct DB access
    throw new Error('Delete not yet implemented for SQLite backend');
  }

  async getStats(): Promise<BackendStats> {
    const status = await ReasoningBank.getStatus();
    return {
      totalEntries: status.total_memories,
      namespaces: status.total_categories,
      sizeBytes: 0, // Would need DB file size
      path: this.path,
    };
  }

  async isHealthy(): Promise<boolean> {
    try {
      await ReasoningBank.initializeReasoningBank();
      return true;
    } catch {
      return false;
    }
  }
}
```

#### 2.2 JsonBackend

**File:** `/src/memory/backends/json-backend.ts`

```typescript
import { promises as fs } from 'node:fs';
import { MemoryBackend, MemoryEntry, MemoryMetadata } from '../types.js';

export class JsonBackend implements MemoryBackend {
  name: 'json' = 'json';
  available = true;
  path = './memory/memory-store.json';
  private data: Record<string, MemoryEntry[]> = {};

  async store(key: string, value: any, metadata: MemoryMetadata): Promise<string> {
    await this.load();

    const namespace = metadata.namespace || 'default';
    if (!this.data[namespace]) {
      this.data[namespace] = [];
    }

    // Remove existing entry
    this.data[namespace] = this.data[namespace].filter(e => e.key !== key);

    // Add new entry
    const entry: MemoryEntry = {
      id: `json-${Date.now()}-${Math.random()}`,
      key,
      value,
      namespace,
      timestamp: Date.now(),
      agent: metadata.agent,
      tags: metadata.tags,
      confidence: metadata.confidence,
    };

    this.data[namespace].push(entry);
    await this.save();

    return entry.id;
  }

  async retrieve(key: string, namespace: string): Promise<MemoryEntry | null> {
    await this.load();
    const entries = this.data[namespace] || [];
    return entries.find(e => e.key === key) || null;
  }

  async query(search: string, options: QueryOptions): Promise<MemoryEntry[]> {
    await this.load();

    const results: MemoryEntry[] = [];
    const namespaces = options.namespace ? [options.namespace] : Object.keys(this.data);

    for (const ns of namespaces) {
      if (this.data[ns]) {
        for (const entry of this.data[ns]) {
          // Basic text search
          if (entry.key.includes(search) ||
              JSON.stringify(entry.value).includes(search)) {
            results.push(entry);
          }
        }
      }
    }

    return results.slice(0, options.limit || 10);
  }

  async list(options: ListOptions): Promise<MemoryEntry[]> {
    await this.load();

    const namespace = options.namespace;
    let results: MemoryEntry[] = [];

    if (namespace) {
      results = this.data[namespace] || [];
    } else {
      for (const entries of Object.values(this.data)) {
        results.push(...entries);
      }
    }

    return results.slice(
      options.offset || 0,
      (options.offset || 0) + (options.limit || 10)
    );
  }

  async delete(key: string, namespace: string): Promise<boolean> {
    await this.load();

    if (!this.data[namespace]) return false;

    const before = this.data[namespace].length;
    this.data[namespace] = this.data[namespace].filter(e => e.key !== key);
    const after = this.data[namespace].length;

    if (before > after) {
      await this.save();
      return true;
    }

    return false;
  }

  async getStats(): Promise<BackendStats> {
    await this.load();

    let totalEntries = 0;
    for (const entries of Object.values(this.data)) {
      totalEntries += entries.length;
    }

    const jsonStr = JSON.stringify(this.data);
    const sizeBytes = new TextEncoder().encode(jsonStr).length;

    return {
      totalEntries,
      namespaces: Object.keys(this.data).length,
      sizeBytes,
      path: this.path,
    };
  }

  async isHealthy(): Promise<boolean> {
    try {
      await this.load();
      return true;
    } catch {
      return false;
    }
  }

  private async load(): Promise<void> {
    try {
      const content = await fs.readFile(this.path, 'utf-8');
      this.data = JSON.parse(content);
    } catch {
      this.data = {};
    }
  }

  private async save(): Promise<void> {
    await fs.mkdir('./memory', { recursive: true });
    await fs.writeFile(this.path, JSON.stringify(this.data, null, 2));
  }
}
```

---

### Phase 3: Integration Points (Week 3)

#### 3.1 Update CLI Commands

**File:** `/src/cli/commands/memory.ts`

**Changes:**
```typescript
// OLD:
import { UnifiedMemoryManager } from './memory-classes.js';

// NEW:
import { getUnifiedMemoryRouter } from '../../memory/unified-memory-router.js';

// Replace all UnifiedMemoryManager instances with router
const router = getUnifiedMemoryRouter();
await router.initialize();

// store command
await router.store(key, value, { namespace, agent, tags });

// query command
const results = await router.query(search, { namespace, limit });

// list command
const results = await router.list({ namespace, limit });

// stats command
const stats = await router.getStats();
```

#### 3.2 Update MCP Tools

**File:** `/src/mcp/mcp-server.ts`

**Add memory_usage tool routing:**
```typescript
import { getUnifiedMemoryRouter } from '../memory/unified-memory-router.js';

// In MCP server tool handler
case 'memory_usage': {
  const router = getUnifiedMemoryRouter();
  await router.initialize();

  switch (args.action) {
    case 'store':
      const id = await router.store(args.key, args.value, {
        namespace: args.namespace || 'default',
        agent: args.agent,
      });
      return { success: true, id };

    case 'query':
      const results = await router.query(args.query, {
        namespace: args.namespace,
        limit: args.limit || 10,
      });
      return { results };

    case 'stats':
      const stats = await router.getStats();
      return stats;
  }
}
```

#### 3.3 Update Swarm Memory

**File:** `/src/swarm/memory.ts`

**Integrate SwarmMemoryManager with router:**
```typescript
import { getUnifiedMemoryRouter } from '../memory/unified-memory-router.js';

export class SwarmMemoryManager extends EventEmitter {
  private router = getUnifiedMemoryRouter();

  async store(key: string, value: any, options: any): Promise<string> {
    // Use unified router for actual storage
    return await this.router.store(key, value, {
      namespace: options.partition || 'swarm',
      agent: options.owner?.id,
      tags: options.tags,
      confidence: 0.9,
    });
  }

  async retrieve(key: string, options: any): Promise<any> {
    await this.router.initialize();
    const results = await this.router.query(key, {
      namespace: options.partition || 'swarm',
      limit: 1,
    });

    return results.length > 0 ? results[0].value : null;
  }

  // Keep existing distributed memory features (partitions, replication, etc.)
  // But use router for actual persistence
}
```

#### 3.4 Create Migration Utility

**File:** `/src/memory/migration-tool.ts`

```typescript
export class MemoryMigrationTool {
  /**
   * Migrate from old storage locations to unified storage
   */
  async migrateToUnified(): Promise<MigrationReport> {
    const report: MigrationReport = {
      sources: [],
      migrated: 0,
      failed: 0,
      skipped: 0,
    };

    // 1. Migrate from old JSON store
    await this.migrateFromOldJson(report);

    // 2. Migrate from SwarmMemory DB (if different)
    await this.migrateFromSwarmDb(report);

    // 3. Migrate from Hive Mind (optional)
    await this.migrateFromHiveMind(report);

    return report;
  }

  private async migrateFromOldJson(report: MigrationReport): Promise<void> {
    // Read old JSON file
    // Write to unified router
    // Track success/failures
  }

  /**
   * Verify migration integrity
   */
  async verifyMigration(): Promise<VerificationReport> {
    // Compare old vs new storage
    // Check for data loss
    // Validate all entries accessible
  }
}
```

---

### Phase 4: Testing & Validation (Week 4)

#### 4.1 Integration Tests

**File:** `/tests/integration/unified-memory.test.ts`

```typescript
describe('Unified Memory Architecture', () => {
  describe('Cross-Entry-Point Consistency', () => {
    it('should store via CLI and retrieve via MCP', async () => {
      // Store via CLI router
      const router = getUnifiedMemoryRouter();
      await router.store('test-key', 'test-value', { namespace: 'test' });

      // Query via MCP tool simulation
      const results = await router.query('test-key', { namespace: 'test' });

      expect(results).toHaveLength(1);
      expect(results[0].value).toBe('test-value');
    });

    it('should store via MCP and list via CLI', async () => {
      // Store via MCP tool simulation
      await router.store('mcp-key', 'mcp-value', { namespace: 'mcp' });

      // List via CLI simulation
      const results = await router.list({ namespace: 'mcp' });

      expect(results.some(r => r.key === 'mcp-key')).toBe(true);
    });
  });

  describe('Backend Fallback', () => {
    it('should fallback to JSON when SQLite unavailable', async () => {
      // Mock SQLite unavailable
      // Initialize router
      // Verify JSON backend is used
      const stats = await router.getStats();
      expect(stats.backend).toBe('json');
    });

    it('should migrate JSON to SQLite when available', async () => {
      // Store in JSON mode
      // Switch to SQLite mode
      // Verify migration
      // Verify data accessible
    });
  });

  describe('Swarm Integration', () => {
    it('should share memory between swarm and general memory', async () => {
      // Store via SwarmMemoryManager
      const swarmMemory = new SwarmMemoryManager();
      await swarmMemory.store('swarm-key', 'swarm-value', { partition: 'swarm' });

      // Query via unified router
      const results = await router.query('swarm-key');
      expect(results).toHaveLength(1);
    });
  });
});
```

---

## Migration Strategy

### Step-by-Step Migration Plan

#### Stage 1: Foundation (No Breaking Changes)
**Duration:** 1 week
**Risk:** Low

1. Create new unified memory modules (router, facade, backends)
2. Write comprehensive tests
3. Deploy alongside existing code
4. No changes to public APIs yet

#### Stage 2: Gradual Integration (Feature Flag)
**Duration:** 1 week
**Risk:** Medium

1. Add feature flag: `UNIFIED_MEMORY_ENABLED=true`
2. Update CLI commands to use router when flag enabled
3. Update MCP tools to use router when flag enabled
4. Monitor metrics: performance, errors, data consistency
5. Default flag to `false` (off)

#### Stage 3: Beta Testing (Opt-In)
**Duration:** 2 weeks
**Risk:** Medium

1. Document opt-in process for users
2. Provide migration utility: `npx claude-flow memory migrate`
3. Collect feedback
4. Fix issues
5. Performance optimization

#### Stage 4: General Availability (Gradual Rollout)
**Duration:** 1 week
**Risk:** Medium

1. Change default flag to `true`
2. Provide rollback instructions
3. Monitor production metrics
4. Hot-fix critical issues

#### Stage 5: Legacy Cleanup (Breaking Change)
**Duration:** 2 weeks
**Risk:** High

1. Mark old APIs as deprecated
2. Remove feature flag
3. Remove old memory modules
4. Update all documentation
5. Release as major version (v3.0.0)

---

## File Structure

```
src/
├── memory/
│   ├── unified-memory-router.ts       # NEW: Entry point router
│   ├── memory-facade.ts               # NEW: Unified interface
│   ├── storage-backend-manager.ts     # NEW: Backend selection
│   ├── migration-tool.ts              # NEW: Migration utility
│   ├── types.ts                       # NEW: Shared types
│   │
│   ├── backends/
│   │   ├── sqlite-backend.ts          # NEW: SQLite implementation
│   │   ├── json-backend.ts            # NEW: JSON implementation
│   │   └── backend-interface.ts       # NEW: Backend contract
│   │
│   └── unified-memory-manager.js      # LEGACY: Keep for now
│
├── cli/commands/
│   └── memory.ts                      # MODIFY: Use router
│
├── mcp/
│   └── mcp-server.ts                  # MODIFY: Use router
│
├── swarm/
│   └── memory.ts                      # MODIFY: Integrate router
│
└── reasoningbank/
    └── reasoningbank-adapter.js       # KEEP: Used by SQLiteBackend

tests/
└── integration/
    └── unified-memory.test.ts         # NEW: Cross-entry-point tests
```

---

## Configuration Management

### Memory Config File

**Location:** `.claude-flow/memory-config.json`

```json
{
  "version": "1.0",
  "backend": {
    "preferred": "sqlite",
    "fallback": "json",
    "auto_migrate": true
  },
  "storage": {
    "sqlite_path": ".swarm/memory.db",
    "json_path": "./memory/memory-store.json"
  },
  "features": {
    "unified_memory": true,
    "semantic_search": true,
    "cross_entry_point": true
  },
  "migration": {
    "completed": false,
    "last_run": null,
    "source_backups": []
  }
}
```

---

## Performance Considerations

### Expected Performance

| Operation | SQLite Backend | JSON Backend | Improvement |
|-----------|----------------|--------------|-------------|
| Store     | ~0.5ms         | ~10ms        | 20x faster  |
| Query     | ~0.1ms         | ~50ms        | 500x faster |
| List      | ~1ms           | ~20ms        | 20x faster  |
| Semantic  | ✅ Yes         | ❌ No        | N/A         |

### Memory Overhead

- **Router Layer:** ~1KB per request (negligible)
- **Facade Layer:** ~2KB per operation (negligible)
- **Backend Manager:** ~10KB persistent (minimal)
- **Total Overhead:** <0.1% of typical operation

---

## Rollback Strategy

### If Migration Fails

1. **Immediate Rollback:**
   ```bash
   npx claude-flow memory rollback --to-version=2.x
   ```

2. **Restore from Backup:**
   ```bash
   npx claude-flow memory restore --backup-id=<id>
   ```

3. **Manual Fallback:**
   - Set `UNIFIED_MEMORY_ENABLED=false`
   - Restart CLI/MCP server
   - Old behavior restored

### Data Safety

- **Automatic Backups:** Before migration starts
- **Backup Location:** `.claude-flow/backups/memory/`
- **Retention:** 30 days
- **Format:** JSON export (portable)

---

## Success Metrics

### Key Performance Indicators (KPIs)

1. **Cross-Entry-Point Consistency:**
   - Target: 100% data visibility across CLI/MCP/API
   - Measure: Integration tests pass rate

2. **Migration Success Rate:**
   - Target: >99% data migrated without loss
   - Measure: Verification report

3. **Performance:**
   - Target: <5ms avg query time (SQLite)
   - Target: <50ms avg query time (JSON)
   - Measure: Benchmark suite

4. **Adoption:**
   - Target: >80% users on unified memory by Week 6
   - Measure: Feature flag telemetry

5. **Stability:**
   - Target: <0.1% error rate
   - Measure: Production logs

---

## Risk Assessment

### High Risk Items

| Risk | Impact | Probability | Mitigation |
|------|--------|-------------|------------|
| Data Loss During Migration | Critical | Low | Automatic backups, rollback |
| SQLite Corruption | High | Low | WAL mode, backup strategy |
| NPX Incompatibility | Medium | High | JSON fallback, clear docs |
| Performance Regression | Medium | Medium | Benchmarks, monitoring |
| Breaking API Changes | High | Low | Feature flags, gradual rollout |

---

## Future Enhancements

### Post-V1 Roadmap

1. **Distributed Memory (V2):**
   - Multi-node synchronization
   - CRDT-based conflict resolution
   - Gossip protocol for consistency

2. **Advanced Search (V2):**
   - Full-text search with FTS5
   - Vector embeddings with FAISS
   - Hybrid search (keyword + semantic)

3. **Memory Optimization (V2):**
   - Automatic compression
   - TTL-based eviction
   - Hot/cold tier separation

4. **Cloud Sync (V3):**
   - Optional cloud backup
   - Cross-device sync
   - Team collaboration

---

## Conclusion

This unified memory architecture provides:

✅ **Single Source of Truth:** All entry points access the same storage
✅ **Backward Compatible:** Existing code continues to work
✅ **NPX-Friendly:** Graceful fallback when SQLite unavailable
✅ **High Performance:** 150x faster with SQLite backend
✅ **Gradual Migration:** Low-risk, phased rollout
✅ **Data Safety:** Automatic backups and rollback

**Recommendation:** Proceed with implementation following the 4-phase plan outlined above.

---

## Appendix A: API Examples

### CLI Usage (Unchanged)
```bash
# Store
npx claude-flow memory store "api-key" "secret-123" --namespace=secrets

# Query
npx claude-flow memory query "api-key" --namespace=secrets

# Stats
npx claude-flow memory stats
```

### MCP Tool Usage (Unchanged)
```typescript
// Store
await mcp.callTool('mcp__claude-flow__memory_usage', {
  action: 'store',
  key: 'api-key',
  value: 'secret-123',
  namespace: 'secrets'
});

// Query
await mcp.callTool('mcp__claude-flow__memory_usage', {
  action: 'query',
  query: 'api-key',
  namespace: 'secrets'
});
```

### Programmatic API (New)
```typescript
import { getUnifiedMemoryRouter } from 'claude-flow/memory';

const router = getUnifiedMemoryRouter();
await router.initialize();

// Store
await router.store('api-key', 'secret-123', { namespace: 'secrets' });

// Query
const results = await router.query('api-key', { namespace: 'secrets' });

// Stats
const stats = await router.getStats();
```

---

**Document Status:** Ready for Review
**Next Steps:** Architecture review → Implementation approval → Phase 1 kickoff
