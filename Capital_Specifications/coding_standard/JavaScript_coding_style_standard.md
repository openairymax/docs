Copyright (c) 2025-2026 SPHARX Ltd. All Rights Reserved.
"From data intelligence emerges."

# Airymax JavaScript 编码规范

**最新**: 2026-06-09
**状态**: 维护中
**路径**: OpenAirymax/Docs/Capital_Specifications/coding_standard/JavaScript_coding_style_standard.md
---

## 一、概述

### 1.1 编制目的

本规范为 Airymax 项目中的 JavaScript/TypeScript 代码提供统一的编码标准。基于项目架构设计原则的五维正交系统，本规范聚焦于工程观维度（E-1至E-4），为开发者提供可操作的代码实现指南。

### 1.2 理论基础

本规范基于 Airymax 架构设计原则的五维正交系统：

- **《工程控制论》**（原则 S-1, E-2）：通过错误处理、日志、健康检查构建反馈闭环
- **《论系统工程》**（原则 S-2）：模块化、接口驱动、边界清晰
- **Thinkdual 双思考系统**（原则 C-1）：TypeScript 提供编译时检查（t2 慢思考），JavaScript 提供运行时灵活（t1 快思考）

**双思考系统在 JavaScript/TypeScript 中的体现**:

| 场景 | t1 快思考（快速） | t2 慢思考（严谨） |
|------|-----------------|-----------------|
| 类型系统 | JavaScript 动态类型 | TypeScript 静态类型 |
| 错误处理 | 运行时 try-catch | 编译时类型检查 |
| 开发体验 | 快速原型开发 | 完整类型定义 |
| 性能优化 | JIT 热路径优化 | AOT 编译优化 |

### 1.3 适用范围

| 环境 | 标准 | 运行环境 |
|------|------|----------|
| Node.js | ES2022+ / TypeScript 5.0+ | Node.js 18+ |
| 浏览器 | ES2022 / TypeScript 5.0+ | ES2022 兼容浏览器 |
| SDK | ES2022 / TypeScript 5.0+ | Node.js 18+ |

---

## 二、文件组织

### 2.1 文件命名

- **TypeScript 文件**：`.ts` 扩展名，采用 `kebab-case`
- **JavaScript 文件**：`.js` 扩展名，采用 `kebab-case`
- **类型定义文件**：`.d.ts` 扩展名
- **React 组件**：`.tsx`/`.jsx` 扩展名

```
src/
├── agents/
│   ├── task-scheduler.ts
│   ├── memory-manager.ts
│   └── cognition-engine.ts
├── types/
│   ├── agent.types.ts
│   └── task.types.ts
└── utils/
    ├── error-handler.ts
    └── logger.ts
```

### 2.2 模块结构

```typescript
/**
 * @fileoverview 任务调度器模块
 *
 * 提供任务调度核心功能，包括任务提交、状态管理、
 * 优先级队列和依赖解析。
 *
 * @module agentos/scheduler
 * @author SPHARX Ltd. - Airymax Team
 * @version 1.6.0
 */

// 1. 导入语句
import { EventEmitter } from 'events';
import { validateTaskPlan } from '../utils/validator';
import { Logger } from '../utils/logger';

// 2. 类型定义
export interface SchedulerConfig {
    name: string;
    maxWorkers: number;
    defaultTimeout: number;
}

export type SchedulerStatus = 'idle' | 'running' | 'closed';

// 3. 常量定义
const DEFAULT_CONFIG: Required<SchedulerConfig> = {
    name: 'default',
    maxWorkers: 4,
    defaultTimeout: 30000
};

// 4. 类/函数定义
export class TaskScheduler {
    private manager: SchedulerConfig;
    private status: SchedulerStatus = 'idle';
    
    constructor(manager: SchedulerConfig) {
        this.manager = { ...DEFAULT_CONFIG, ...manager };
    }
    
    // 方法实现...
}

// 5. 导出
export default TaskScheduler;
```

---

## 三、命名规范

### 3.1 命名风格

| 类型 | 风格 | 示例 |
|------|------|------|
| 变量/函数 | camelCase | `taskId`, `submitTask()`, `getTaskStatus()` |
| 类/接口/类型 | PascalCase | `class TaskScheduler`, `interface TaskConfig` |
| 常量 | UPPER_SNAKE_CASE | `MAX_RETRY_COUNT`, `DEFAULT_TIMEOUT` |
| 枚举成员 | camelCase | `TaskStatus.Idle`, `Priority.Normal` |
| 文件名 | kebab-case | `task-scheduler.ts`, `memory-manager.js` |
| 目录名 | kebab-case | `task-scheduler/`, `memory-agentos/manager/` |

### 3.2 命名示例

```typescript
// 正确的命名
const MAX_TASK_NAME_LENGTH = 256;
const taskRegistry = new Map<string, Task>();
const taskId = 'task-001';

enum TaskPriority {
    Low = 1,
    Normal = 5,
    High = 8,
    Critical = 10
}

interface TaskConfig {
    name: string;
    priority: TaskPriority;
}

// 错误的命名
const maxTaskNameLength = 256;  // 常量应该 UPPER_SNAKE_CASE
const TaskRegistry = new Map();  // 变量应该 camelCase
class task_scheduler {}  // 类名应该 PascalCase
```

---

## 四、类型设计

### 4.1 类型别名

```typescript
// 基本类型别名
type TaskId = string;
type Timestamp = number;
type ByteBuffer = Uint8Array;

// 函数类型
type TaskHandler<TInput, TOutput> = (input: TInput) => Promise<TOutput>;
type ErrorHandler = (error: Error) => void;

// 回调类型
type CompletionCallback = (result: TaskResult) => void;
type ProgressCallback = (progress: number) => void;
```

### 4.2 接口定义

```typescript
/**
 * 任务计划接口
 */
export interface TaskPlan {
    /** 任务唯一标识符 */
    readonly id: TaskId;
    
    /** 任务名称 */
    name: string;
    
    /** 执行步骤列表 */
    steps: readonly TaskStep[];
    
    /** 执行策略 */
    strategy: 'sequential' | 'parallel' | 'adaptive';
    
    /** 最大重试次数 */
    maxRetries: number;
    
    /** 创建时间戳 */
    readonly createdAt: Timestamp;
}

/**
 * 任务步骤接口
 */
export interface TaskStep {
    /** 步骤 ID */
    readonly id: string;
    
    /** 步骤名称 */
    name: string;
    
    /** 步骤处理器 */
    handler: TaskHandler<unknown, unknown>;
    
    /** 依赖的步骤 ID */
    dependencies?: readonly string[];
    
    /** 超时时间（毫秒） */
    timeout?: number;
}
```

### 4.3 枚举

```typescript
/**
 * 任务状态枚举
 *
 * ⚠️ 注意：`const enum` 在 `isolatedModules: true`（Babel/esbuild/swc 等转译器默认模式）
 * 下存在跨模块问题——每个文件独立编译时无法内联其他模块的 const enum 值，
 * 导致运行时引用为 undefined。推荐改用普通 enum 或 union type：
 *
 *   // 推荐：普通 enum（无跨模块问题）
 *   export enum TaskStatus { Idle = 'idle', ... }
 *
 *   // 或：字符串 union type（零运行时开销）
 *   export type TaskStatus = 'idle' | 'pending' | 'running' | 'completed' | 'failed' | 'cancelled';
 */
export const enum TaskStatus {
    /** 空闲状态 */
    Idle = 'idle',
    
    /** 等待状态 */
    Pending = 'pending',
    
    /** 运行状态 */
    Running = 'running',
    
    /** 完成状态 */
    Completed = 'completed',
    
    /** 失败状态 */
    Failed = 'failed',
    
    /** 取消状态 */
    Cancelled = 'cancelled'
}

/**
 * 任务优先级枚举
 */
export const enum TaskPriority {
    Low = 1,
    Normal = 5,
    High = 8,
    Critical = 10
}
```

---

## 五、函数设计

### 5.1 函数签名规范

```typescript
/**
 * 提交任务计划
 *
 * 将任务计划加入调度队列，返回任务 ID 用于后续查询。
 * 调度器会根据优先级和依赖关系决定执行顺序。
 *
 * @param plan - 任务计划对象
 * @param options - 提交选项
 * @returns 任务 ID
 *
 * @throws {ValidationError} 当计划格式无效时
 * @throws {SchedulerClosedError} 当调度器已关闭时
 *
 * @example
 * ```typescript
 * const taskId = await scheduler.submit({
 *     name: 'data-processing',
 *     steps: [...],
 *     strategy: 'parallel'
 * });
 * console.log(`Task submitted: ${taskId}`);
 * ```
 */
async submit(
    plan: TaskPlan,
    options?: SubmitOptions
): Promise<string> {
    // 实现...
}
```

### 5.2 参数验证

```typescript
/**
 * 验证任务计划
 */
export function validateTaskPlan(plan: unknown): ValidationResult {
    // 1. 基本类型检查
    if (plan === null || plan === undefined) {
        return { valid: false, error: 'Plan cannot be null or undefined' };
    }
    
    // 2. 类型断言
    if (typeof plan !== 'object') {
        return { valid: false, error: 'Plan must be an object' };
    }
    
    const taskPlan = plan as Partial<TaskPlan>;
    
    // 3. 必填字段验证
    if (!taskPlan.id || typeof taskPlan.id !== 'string') {
        return { valid: false, error: 'Plan must have a valid id' };
    }
    
    // 4. 长度验证
    if (taskPlan.id.length > MAX_TASK_ID_LENGTH) {
        return { valid: false, error: `Task ID too long: ${taskPlan.id.length}` };
    }
    
    // 5. 枚举值验证
    if (taskPlan.strategy && !['sequential', 'parallel', 'adaptive'].includes(taskPlan.strategy)) {
        return { valid: false, error: `Invalid strategy: ${taskPlan.strategy}` };
    }
    
    return { valid: true };
}
```

### 5.3 返回值约定

```typescript
/**
 * 使用 Result 类型处理错误
 */
export type Result<T, E = Error> =
    | { success: true; value: T }
    | { success: false; error: E };

/**
 * 异步操作结果
 */
export interface AsyncResult<T> {
    /** 是否成功 */
    readonly success: boolean;
    /** 成功时的值 */
    readonly value?: T;
    /** 失败时的错误 */
    readonly error?: Error;
}

/**
 * 任务提交结果
 */
export interface SubmitResult {
    /** 任务 ID */
    taskId: string;
    /** 提交时间戳 */
    submittedAt: Timestamp;
    /** 预计开始时间 */
    estimatedStartTime?: Timestamp;
}
```

---

## 六、类设计

### 6.1 类结构模板

```typescript
/**
 * 任务调度器
 *
 * 负责管理任务的生命周期和执行。调度器采用双思考系统架构，
 * t1 快思考 处理简单任务，t2 慢思考 处理复杂任务。
 *
 * @example
 * ```typescript
 * const scheduler = new TaskScheduler({
 *     name: 'main',
 *     maxWorkers: 4
 * });
 *
 * const taskId = await scheduler.submit(plan);
 * const result = await scheduler.wait(taskId);
 * ```
 */
export class TaskScheduler extends EventEmitter {
    /** 调度器配置 */
    private readonly manager: Readonly<Required<SchedulerConfig>>;
    
    /** 当前状态 */
    private status: SchedulerStatus = 'idle';
    
    /** 任务注册表 */
    private readonly tasks: Map<TaskId, TaskContext> = new Map();
    
    /** 工作线程池 */
    private readonly workerPool: WorkerPool;
    
    /**
     * 构造函数
     */
    public constructor(manager: SchedulerConfig) {
        super();
        this.manager = Object.freeze({ ...DEFAULT_CONFIG, ...manager });
        this.workerPool = new WorkerPool(this.manager.maxWorkers);
        
        // 设置事件处理器...
    }
    
    /**
     * 提交任务
     */
    public async submit(plan: TaskPlan): Promise<string> {
        // 实现...
    }
    
    /**
     * 等待任务完成
     */
    public async wait(taskId: string, timeout?: number): Promise<TaskResult> {
        // 实现...
    }
    
    /**
     * 关闭调度器
     */
    public async shutdown(): Promise<void> {
        // 实现...
    }
}
```

### 6.2 私有成员命名

```typescript
class TaskExecutor {
    // 使用下划线前缀标记私有成员
    private _isRunning: boolean = false;
    private _taskQueue: Task[] = [];
    
    // 公开访问器
    public get isRunning(): boolean {
        return this._isRunning;
    }
    
    // 或者使用 # 前缀（ES2022 私有字段）
    #cancellationToken: CancellationToken | null = null;
    
    public execute(task: Task): Promise<TaskResult> {
        // 实现...
    }
}
```

---

## 七、错误处理

### 7.1 错误类定义

```typescript
/**
 * Airymax 错误基类
 */
export class AgentOSError extends Error {
    public readonly code: string;
    public readonly timestamp: number;
    
    constructor(message: string, code: string) {
        super(message);
        this.name = this.constructor.name;
        this.code = code;
        this.timestamp = Date.now();
        Error.captureStackTrace(this, this.constructor);
    }
}

/**
 * 调度器错误
 */
export class SchedulerError extends AgentOSError {
    public constructor(message: string) {
        super(message, 'SCHEDULER_ERROR');
    }
}

/**
 * 调度器已关闭错误
 */
export class SchedulerClosedError extends SchedulerError {
    public constructor() {
        super('Scheduler is closed');
        this.name = 'SchedulerClosedError';
    }
}

/**
 * 任务不存在错误
 */
export class TaskNotFoundError extends SchedulerError {
    public constructor(taskId: string) {
        super(`Task not found: ${taskId}`);
        this.name = 'TaskNotFoundError';
    }
}

/**
 * 验证错误
 */
export class ValidationError extends AgentOSError {
    public constructor(message: string) {
        super(message, 'VALIDATION_ERROR');
    }
}
```

### 7.2 错误处理模式

```typescript
/**
 * 使用 Result 类型处理错误
 */
async function submitTask(
    plan: TaskPlan
): Promise<Result<string, SchedulerError>> {
    try {
        // 验证计划
        const validation = validateTaskPlan(plan);
        if (!validation.valid) {
            return { success: false, error: new ValidationError(validation.error!) };
        }
        
        // 提交任务
        const taskId = await scheduler.submit(plan);
        return { success: true, value: taskId };
        
    } catch (error) {
        if (error instanceof SchedulerError) {
            return { success: false, error };
        }
        return { success: false, error: new SchedulerError(`Unexpected error: ${error}`) };
    }
}

/**
 * 使用 try-catch
 */
async function processTask(taskId: string): Promise<TaskResult> {
    try {
        const task = await taskRegistry.get(taskId);
        if (!task) {
            throw new TaskNotFoundError(taskId);
        }
        
        return await task.execute();
        
    } catch (error) {
        if (error instanceof AgentOSError) {
            logger.error(`Task ${taskId} failed: ${error.message}`, { code: error.code });
            throw error;
        }
        logger.error(`Unexpected error processing task ${taskId}`, error);
        throw new SchedulerError('Internal error');
    }
}
```

---

## 八、异步编程

### 8.1 Promise 使用

```typescript
/**
 * 异步任务执行
 */
async function executeTask(task: Task): Promise<TaskResult> {
    const startTime = Date.now();
    
    try {
        const result = await task.execute();
        return {
            taskId: task.id,
            status: 'completed',
            result,
            duration: Date.now() - startTime
        };
    } catch (error) {
        return {
            taskId: task.id,
            status: 'failed',
            error: error instanceof Error ? error.message : String(error),
            duration: Date.now() - startTime
        };
    }
}

/**
 * 并行执行多个任务
 */
async function executeParallel(tasks: Task[]): Promise<TaskResult[]> {
    return Promise.all(tasks.map(executeTask));
}

/**
 * 带超时的 Promise
 */
function withTimeout<T>(
    promise: Promise<T>,
    timeoutMs: number,
    errorMessage: string = 'Operation timed out'
): Promise<T> {
    return Promise.race([
        promise,
        new Promise<T>((_, reject) =>
            setTimeout(() => reject(new Error(errorMessage)), timeoutMs)
        )
    ]);
}
```

### 8.2 async/await 模式

```typescript
/**
 * 顺序执行任务
 */
async function executeSequential(tasks: Task[]): Promise<TaskResult[]> {
    const results: TaskResult[] = [];
    
    for (const task of tasks) {
        const result = await executeTask(task);
        results.push(result);
        
        // 如果任务失败，可以选择停止
        if (result.status === 'failed' && task.critical) {
            break;
        }
    }
    
    return results;
}

/**
 * 并行执行并限制并发数
 *
 * ⚠️ 修复说明：原实现中 `executing.findIndex(p => p === promise)` 存在 bug——
 * 当 Promise 已 resolve 时，`Promise.race` 返回的是最先完成的 Promise，
 * 但 `findIndex` 查找的是当前迭代的 `promise`（可能尚未 push 到数组），
 * 导致找到的索引为 -1，splice 删除错误元素。正确做法是追踪已完成 Promise 自身。
 */
async function executeWithConcurrency(
    tasks: Task[],
    concurrency: number
): Promise<TaskResult[]> {
    const results: TaskResult[] = [];
    const executing: Promise<void>[] = [];

    for (const task of tasks) {
        const p = executeTask(task).then(result => {
            results.push(result);
            // 从 executing 中移除自身
            const idx = executing.indexOf(p);
            if (idx !== -1) {
                executing.splice(idx, 1);
            }
        });

        executing.push(p);

        if (executing.length >= concurrency) {
            await Promise.race(executing);
        }
    }

    await Promise.all(executing);
    return results;
}
```

---

## 九、模块与导入

### 9.1 导入语句顺序

```typescript
// 1. Node.js 内置模块
import { EventEmitter } from 'events';
import { readFile, writeFile } from 'fs/promises';
import { join } from 'path';

// 2. 外部包
import express, { Request, Response } from 'express';
import { z } from 'zod';
import { Logger } from 'winston';

// 3. 项目内部模块（相对于当前文件）
import { TaskScheduler } from './task-scheduler';
import { TaskPlan, TaskResult } from '../types/task.types';
import { ValidationError } from '../errors';
import { DEFAULT_CONFIG } from '../constants';

// 4. 类型导入（始终使用 type 导入）
import type { manager, Options } from '../types';
import type { Logger } from 'winston';
```

### 9.2 导出模式

```typescript
// named export（命名导出）
export const MAX_RETRY_COUNT = 3;
export interface TaskConfig { ... }
export function submitTask() { ... }

// re-export（重新导出）
export { TaskScheduler } from './task-scheduler';
export type { TaskPlan, TaskResult } from '../types';

// default export（默认导出）
export default class TaskScheduler { ... }

// Barrel 导出（index.ts）
export { TaskScheduler } from './task-scheduler';
export { MemoryManager } from './memory-manager';
export { CognitionEngine } from './cognition-engine';
```

---

## 十、注释规范

### 10.1 JSDoc 注释

```typescript
/**
 * @fileoverview 任务调度器模块
 *
 * 提供任务调度核心功能，包括任务提交、状态管理、
 * 优先级队列和依赖解析。调度器采用双思考系统架构：
 * - t1 快思考：快速路径，处理简单任务
 * - t2 慢思考：深度路径，处理复杂任务
 *
 * @module agentos/scheduler
 */

/**
 * 提交任务计划
 *
 * 将任务计划加入调度队列，返回任务 ID 用于后续查询。
 * 调度器会根据优先级和依赖关系决定执行顺序。
 *
 * @param plan - 任务计划对象，非空
 * @param options - 提交选项，可选
 * @returns Promise 解析为任务 ID
 *
 * @throws {ValidationError} 当计划格式无效
 * @throws {SchedulerClosedError} 当调度器已关闭
 * @throws {QueueFullError} 当队列已满
 *
 * @example
 * ```typescript
 * const taskId = await scheduler.submit({
 *     name: 'data-processing',
 *     steps: [
 *         { id: 'step1', name: 'Load Data', handler: loadData },
 *         { id: 'step2', name: 'Process', handler: processData, dependencies: ['step1'] }
 *     ],
 *     strategy: 'sequential'
 * });
 * ```
 *
 * @see {@link TaskScheduler.wait} 获取任务结果
 * @see {@link TaskScheduler.cancel} 取消任务
 */
async submit(
    plan: TaskPlan,
    options?: SubmitOptions
): Promise<string> {
    // 实现...
}
```

### 10.2 TSDoc 类型注释

```typescript
/**
 * 任务计划
 *
 * @template T - 任务输入数据类型
 * @template R - 任务输出数据类型
 */
export interface TaskPlan<T = unknown, R = unknown> {
    /** 任务唯一标识符 */
    readonly id: string;
    
    /** 任务名称，用于日志和监控 */
    name: string;
    
    /** 执行步骤列表 */
    steps: readonly TaskStep[];
    
    /** 输入数据 */
    input: T;
    
    /** 执行策略 */
    strategy: 'sequential' | 'parallel' | 'adaptive';
}
```

---

## 十一、类型安全

### 11.1 严格模式

```json
// tsconfig.json
{
    "compilerOptions": {
        "strict": true,
        "noImplicitAny": true,
        "strictNullChecks": true,
        "strictFunctionTypes": true,
        "strictBindCallApply": true,
        "strictPropertyInitialization": true,
        "noImplicitThis": true,
        "useUnknownInCatchVariables": true,
        "alwaysStrict": true
    }
}
```

### 11.2 类型守卫

```typescript
/**
 * 类型守卫：检查是否为 Airymax 错误
 */
function isAgentOSError(error: unknown): error is AgentOSError {
    return error instanceof AgentOSError;
}

/**
 * 类型守卫：检查任务计划
 */
function isValidTaskPlan(plan: unknown): plan is TaskPlan {
    if (plan === null || typeof plan !== 'object') {
        return false;
    }
    const p = plan as Record<string, unknown>;
    return (
        typeof p.id === 'string' &&
        typeof p.name === 'string' &&
        Array.isArray(p.steps)
    );
}

/**
 * 使用类型守卫
 */
async function handleError(error: unknown): Promise<void> {
    if (isAgentOSError(error)) {
        logger.error(`Airymax error: ${error.code} - ${error.message}`);
        // error.code 是确定存在的
    } else if (error instanceof Error) {
        logger.error(`Unexpected error: ${error.message}`);
    }
}
```

---

## 十二、测试集成

### 12.1 Jest 测试示例

```typescript
/**
 * @fileoverview 任务调度器测试
 */

import { describe, it, expect, beforeEach, afterEach, jest } from '@jest/globals';
import { TaskScheduler } from '../task-scheduler';
import { TaskPlan, TaskStatus } from '../types';

describe('TaskScheduler', () => {
    let scheduler: TaskScheduler;
    
    beforeEach(() => {
        scheduler = new TaskScheduler({
            name: 'test',
            maxWorkers: 2
        });
    });
    
    afterEach(async () => {
        await scheduler.shutdown();
    });
    
    describe('submit', () => {
        it('should submit a task and return task id', async () => {
            const plan: TaskPlan = {
                id: 'task-001',
                name: 'test-task',
                steps: [],
                strategy: 'sequential'
            };
            
            const taskId = await scheduler.submit(plan);
            
            expect(taskId).toBe('task-001');
        });
        
        it('should throw ValidationError for invalid plan', async () => {
            const invalidPlan = { name: 'test' } as TaskPlan;
            
            await expect(scheduler.submit(invalidPlan))
                .rejects.toThrow('id');
        });
    });
    
    describe('wait', () => {
        it('should return task result after completion', async () => {
            const plan: TaskPlan = {
                id: 'task-002',
                name: 'completed-task',
                steps: [],
                strategy: 'sequential'
            };
            
            await scheduler.submit(plan);
            const result = await scheduler.wait('task-002', 5000);
            
            expect(result.status).toBe(TaskStatus.Completed);
        });
    });
});
```

---

## 十三-A、TypeScript/JavaScript 禁止模式清单（BAN-300~305）

所有 TypeScript/JavaScript PR 必须通过以下禁止模式检查：

| 编号 | 禁止模式 | 检测方式 | 替代方案 |
|------|---------|----------|---------|
| BAN-300 | 禁止使用 `any` 类型 | `tsconfig strict: true` + `@typescript-eslint/no-explicit-any` | 使用具体类型、泛型或 `unknown` |
| BAN-301 | 禁止 `@ts-ignore` / `@ts-expect-error` | `@typescript-eslint/ban-ts-comment` | 修复类型错误，必要时使用类型守卫 |
| BAN-302 | 禁止 `console.log` | `eslint no-console` / `ruff T201` | 使用 `logger.ts` 封装的日志模块 |
| BAN-303 | 禁止空 catch 块 | `eslint no-empty` + `no-catch-shadow` | 至少记录 `logger.warn()` 或 `logger.error()` |
| BAN-304 | 禁止硬编码 URL/端口 | 自定义 ESLint 规则 / 代码审查 | 使用配置文件或环境变量 |
| BAN-305 | 禁止 `eval()` / `Function()` 构造器 | `eslint no-eval` / `no-new-func` | 使用安全的 JSON.parse 或模板引擎 |

**示例**：

```typescript
// ❌ BAN-300: 使用 any
function process(data: any) { ... }  // 禁止！

// ✅ 正确：使用具体类型或泛型
function process<T>(data: T): Result<T> { ... }

// ❌ BAN-301: 使用 @ts-ignore
// @ts-ignore
const result = someFunction();  // 禁止！

// ✅ 正确：修复类型或使用类型守卫
if (isValidData(data)) {
    const result = someFunction(data);
}

// ❌ BAN-302: 使用 console.log
console.log('Task completed');  // 禁止！

// ✅ 正确：使用 logger
import { logger } from '../utils/logger';
logger.info('Task completed', { taskId });

// ❌ BAN-303: 空 catch 块
try { ... } catch (e) { }  // 禁止！

// ✅ 正确：记录异常
try { ... } catch (e) {
    logger.error('Operation failed', { error: e });
}

// ❌ BAN-304: 硬编码 URL
const API_URL = 'http://localhost:8080/api';  // 禁止！

// ✅ 正确：使用配置
const API_URL = config.get('api.url');
```

---

## 十三-B、C 内核通信规范

### 13-B.1 通信架构

TypeScript/JavaScript 层与 C 内核（atoms 层）通过以下方式通信：

```
┌─────────────────────┐     WebSocket/HTTP      ┌──────────────────┐
│  Desktop / Web SDK  │ ◄──────────────────────► │  daemon 服务层    │
│  (TypeScript)       │     JSON-RPC 2.0         │  (C++ daemon)    │
└─────────────────────┘                          └────────┬─────────┘
                                                          │ IPC / syscalls
                                                 ┌────────▼─────────┐
                                                 │  atoms 内核层     │
                                                 │  (C)             │
                                                 └──────────────────┘
```

### 13-B.2 WebSocket/HTTP API 规范

daemon 层对外暴露 WebSocket（实时通信）和 HTTP REST（管理操作）两种接口：

```typescript
/**
 * daemon 通信客户端
 */
export class DaemonClient {
    private ws?: WebSocket;
    private readonly baseUrl: string;
    private requestId = 0;

    constructor(config: DaemonClientConfig) {
        this.baseUrl = config.daemonUrl;
    }

    /**
     * 通过 HTTP REST 发送管理请求
     */
    async request(method: 'GET' | 'POST' | 'PUT' | 'DELETE',
                  path: string, body?: unknown): Promise<unknown> {
        const response = await fetch(`${this.baseUrl}${path}`, {
            method,
            headers: { 'Content-Type': 'application/json' },
            body: body ? JSON.stringify(body) : undefined,
        });
        if (!response.ok) {
            throw new DaemonError(
                `HTTP ${response.status}`, this.mapHttpStatus(response.status)
            );
        }
        return response.json();
    }

    /**
     * 通过 WebSocket 发送实时指令（JSON-RPC 2.0）
     */
    async rpc(method: string, params?: unknown): Promise<unknown> {
        const id = ++this.requestId;
        const message: JsonRpcRequest = {
            jsonrpc: '2.0',
            id,
            method,
            params: params ?? {},
        };
        return this.sendWs(message);
    }
}
```

### 13-B.3 错误码映射

AGENTOS_E* 错误码到 HTTP 状态码的映射：

| AGENTOS 错误码 | HTTP 状态码 | 说明 |
|---------------|------------|------|
| `AGENTOS_OK` (0) | 200 | 成功 |
| `AGENTOS_ERR_INVALID_PARAM` (-2) | 400 | 参数无效 |
| `AGENTOS_ERR_OUT_OF_MEMORY` (-4) | 503 | 资源不足 |
| `AGENTOS_ERR_TIMEOUT` (-8) | 408 / 504 | 请求超时 |
| `AGENTOS_ERR_BUSY` (-17) | 429 | 服务繁忙 |
| `AGENTOS_ERR_NOT_FOUND` | 404 | 资源不存在 |
| `AGENTOS_ERR_PERMISSION_DENIED` | 403 | 权限不足 |
| 其他负值 | 500 | 内部错误 |

### 13-B.4 JSON-RPC 2.0 服务层规范

所有 daemon 服务层的 RPC 接口必须遵循 JSON-RPC 2.0 规范：

```typescript
/**
 * JSON-RPC 2.0 请求/响应类型
 */
export interface JsonRpcRequest {
    jsonrpc: '2.0';
    id: number | string;
    method: string;
    params?: Record<string, unknown> | unknown[];
}

export interface JsonRpcResponse<T = unknown> {
    jsonrpc: '2.0';
    id: number | string;
    result?: T;
    error?: {
        code: number;
        message: string;
        data?: unknown;
    };
}

/**
 * JSON-RPC 2.0 错误码规范
 *
 * -32700: Parse error（JSON 解析失败）
 * -32600: Invalid Request（请求格式无效）
 * -32601: Method not found（方法不存在）
 * -32602: Invalid params（参数无效）
 * -32603: Internal error（内部错误）
 * -32000 ~ -32099: Server error（服务端自定义错误，映射 AGENTOS_E*）
 */
```

---

## 十四、Airymax 模块 JavaScript/TypeScript 编码示例

### 14.1 daemon（守护层）TypeScript 实现
Backs模块作为系统用户态服务，需要高可靠性和可观测性：

#### 14.1.1 IPC 通信服务（映射原则：E-3 通信基础设施）
```typescript
/**
 * IPC通信服务 - 体现系统观（S-3）和工程观（E-2, E-4）原则
 * 
 * 实现与Atoms模块的高性能进程间通信。
 * 集成OpenTelemetry可观测性和消息加密。
 * 
 * @see ipc.md 中的通信协议规范
 */
@Injectable()
export class IpcService implements OnModuleInit, OnModuleDestroy {
  private readonly logger = new Logger(IpcService.name);
  private readonly metrics = new MetricsCollector('ipc_service');
  private readonly crypto = new MessageCrypto();
  private channel?: IpcChannel;
  
  /**
   * 发送安全消息 - 体现防御深度（D-3）原则
   * 
   * 多层安全验证：
   * 1. 消息签名验证
   * 2. 发送者身份验证
   * 3. 消息加密
   * 4. 完整性保护
   */
  async sendSecure<TRequest, TResponse>(
    request: SecureRequest<TRequest>,
    options: SendOptions = {}
  ): Promise<SecureResponse<TResponse>> {
    // 层次1：请求签名
    const signature = await this.crypto.sign(request.payload);
    const signedRequest = { ...request, signature };
    
    // 层次2：消息加密
    const encrypted = await this.crypto.encrypt(
      JSON.stringify(signedRequest),
      this.sessionKey
    );
    
    // 层次3：身份验证
    const token = await this.auth.getCurrentToken();
    if (!token.valid) {
      throw new SecurityException('Authentication token expired');
    }
    
    // 层次4：发送并验证响应
    const rawResponse = await this.channel!.send(encrypted, options);
    const response = this.parseResponse<TResponse>(rawResponse);
    
    // 验证响应完整性
    const isValid = await this.crypto.verify(
      response.signature,
      response.payload
    );
    if (!isValid) {
      this.metrics.increment('integrity_check_failed');
      throw new SecurityException('Response integrity check failed');
    }
    
    return response;
  }
  
  private parseResponse<T>(raw: RawResponse): SecureResponse<T> {
    // 类型安全解析，使用Zod验证模式
    return ipcResponseSchema.parse(raw) as SecureResponse<T>;
  }
}
```

#### 14.1.2 任务监控用户态服务（映射原则：E-2 可维护性）
```typescript
/**
 * 任务监控用户态服务 - 体现工程观（E-2）和设计美学（A-1）原则
 * 
 * 监控Airymax任务执行状态，提供实时指标和告警。
 * 基于事件驱动的架构，支持插件化扩展。
 */
@Controller('tasks')
export class TaskMonitorDaemon {
  private readonly tasks = new Map<string, TaskContext>();
  private readonly eventEmitter = new EventEmitter();
  
  /**
   * 订阅任务状态变更 - 体现反馈闭环（S-1）原则
   */
  @SubscribeMessage('task:status')
  async handleTaskStatus(
    @MessageBody() data: TaskStatusUpdate
  ): Promise<MonitoringResponse> {
    const { taskId, status, timestamp, metrics } = data;
    
    // 更新任务状态
    const context = this.tasks.get(taskId);
    if (context) {
      context.status = status;
      context.lastUpdate = timestamp;
      context.metrics = metrics;
      
      // 触发状态变更事件
      this.eventEmitter.emit('task:status:changed', {
        taskId,
        oldStatus: context.previousStatus,
        newStatus: status,
        timestamp
      });
      
      // 检查告警条件
      await this.checkAlerts(context);
    }
    
    return { success: true, monitored: true };
  }
  
  /**
   * 检查告警条件 - 体现t2 慢思考（深度分析）原则
   */
  private async checkAlerts(context: TaskContext): Promise<void> {
    const alerts: Alert[] = [];
    
    // 性能告警：CPU使用率超过阈值
    if (context.metrics.cpuUsage > context.thresholds.cpu) {
      alerts.push({
        type: 'performance',
        severity: 'warning',
        message: `CPU usage high: ${context.metrics.cpuUsage}%`,
        taskId: context.taskId,
        timestamp: Date.now()
      });
    }
    
    // 错误率告警
    if (context.metrics.errorRate > context.thresholds.errorRate) {
      alerts.push({
        type: 'error',
        severity: 'critical',
        message: `Error rate high: ${context.metrics.errorRate}%`,
        taskId: context.taskId,
        timestamp: Date.now()
      });
    }
    
    // 发送告警
    if (alerts.length > 0) {
      await this.alertService.sendAlerts(alerts);
    }
  }
}
```

### 14.2 cupolas（安全穹顶）TypeScript 实现
cupolas模块实现零信任安全模型，需要严格的安全保障：

#### 14.2.1 安全策略引擎（映射原则：D-1 最小权限）
```typescript
/**
 * 安全策略引擎 - 体现安全工程（D-1至D-4）原则
 * 
 * 基于YAML规则引擎实现细粒度访问控制。
 * 支持动态策略更新和形式化验证。
 * 
 * @see Security_design_standard.md 中的零信任架构
 */
export class SecurityPolicyEngine {
  private readonly policies = new Map<string, SecurityPolicy>();
  private readonly validators = new Map<string, PolicyValidator>();
  private readonly auditLogger = new AuditLogger();
  
  /**
   * 评估访问请求 - 体现防御深度（D-3）原则
   */
  async evaluate(request: AccessRequest): Promise<AccessDecision> {
    // 层次1：主体验证
    const subjectValid = await this.validateSubject(request.subject);
    if (!subjectValid) {
      await this.auditLogger.log('subject_validation_failed', request);
      return AccessDecision.deny('Invalid subject');
    }
    
    // 层次2：资源权限检查
    const hasPermission = await this.checkPermission(
      request.subject,
      request.resource,
      request.action
    );
    if (!hasPermission) {
      await this.auditLogger.log('permission_denied', request);
      return AccessDecision.deny('Insufficient permission');
    }
    
    // 层次3：上下文策略评估
    const contextValid = await this.evaluateContext(request.context);
    if (!contextValid) {
      await this.auditLogger.log('context_violation', request);
      return AccessDecision.deny('Context policy violation');
    }
    
    // 层次4：风险评估
    const risk = await this.assessRisk(request);
    if (risk.level === RiskLevel.HIGH) {
      await this.auditLogger.log('high_risk_denied', { ...request, risk });
      return AccessDecision.deny('High risk access');
    }
    
    // 授予访问权限
    await this.auditLogger.log('access_granted', { ...request, risk });
    return AccessDecision.grant({
      riskScore: risk.score,
      sessionTimeout: risk.recommendedTimeout
    });
  }
  
  /**
   * 动态更新策略 - 体现工程观（E-2）原则
   */
  async updatePolicy(policyId: string, policy: SecurityPolicy): Promise<void> {
    // 验证策略格式
    const validation = await this.validatePolicy(policy);
    if (!validation.valid) {
      throw new PolicyValidationError(validation.errors);
    }
    
    // 形式化验证（可选）
    if (policy.formalVerification) {
      const formalResult = await this.formalVerify(policy);
      if (!formalResult.verified) {
        throw new FormalVerificationError(formalResult.violations);
      }
    }
    
    // 更新策略
    this.policies.set(policyId, policy);
    
    // 审计日志
    await this.auditLogger.log('policy_updated', {
      policyId,
      timestamp: Date.now(),
      updatedBy: this.currentUser
    });
  }
}
```

### 14.3 commons（公共库层）TypeScript 实现
Common模块提供跨层基础设施，强调通用性和性能：

#### 14.3.1 向量数据库客户端（映射原则：E-1 基础设施）
```typescript
/**
 * 向量数据库客户端 - 体现工程观（E-1, E-3）和认知观（C-3）原则
 * 
 * 封装FAISS和HNSW向量索引，支持持久化存储和拓扑分析。
 * 集成HDBSCAN聚类算法，自动发现数据中的拓扑结构。
 * 
 * @see memoryrovol.md 中的记忆进化算法
 */
export class VectorDbClient {
  private readonly index: VectorIndex;
  private readonly metadataStore: MetadataStore;
  private readonly cache = new LRUCache<string, Vector>(1000);
  
  /**
   * 相似性搜索 - 体现性能优化和资源确定性
   */
  async search(
    query: Vector,
    options: SearchOptions = {}
  ): Promise<SearchResult[]> {
    const startTime = performance.now();
    
    try {
      // 缓存查询
      const cacheKey = this.hashVector(query);
      const cached = this.cache.get(cacheKey);
      if (cached && !options.forceRefresh) {
        this.metrics.increment('cache_hit');
        return cached;
      }
      
      // 执行搜索
      const results = await this.index.search(query, options);
      
      // 批量获取元数据（减少随机访问）
      const ids = results.map(r => r.id);
      const metadata = await this.metadataStore.batchGet(ids);
      
      // 构造结果
      const formatted = results.map((result, index) => ({
        id: result.id,
        score: result.score,
        metadata: metadata[index],
        vector: result.vector
      }));
      
      // 更新缓存
      this.cache.set(cacheKey, formatted);
      
      // 记录性能指标
      const duration = performance.now() - startTime;
      this.metrics.record('search_duration', duration);
      
      return formatted;
      
    } catch (error) {
      this.metrics.increment('search_error');
      this.logger.error('Search failed', { error, query: query.slice(0, 10) });
      throw new VectorDbError('Search failed', { cause: error });
    }
  }
  
  /**
   * 批量插入向量 - 体现批量处理优化
   */
  async insertBatch(
    vectors: Vector[],
    metadata: Metadata[] = []
  ): Promise<string[]> {
    // 预分配结果数组
    const ids = new Array<string>(vectors.length);
    
    // 批量插入索引
    const indexIds = await this.index.insertBatch(vectors);
    
    // 批量存储元数据
    if (metadata.length > 0) {
      await this.metadataStore.batchSet(
        indexIds.map((id, index) => ({
          id,
          metadata: metadata[index] || {}
        }))
      );
    }
    
    // 更新缓存
    vectors.forEach((vector, index) => {
      const cacheKey = this.hashVector(vector);
      this.cache.delete(cacheKey);
    });
    
    return indexIds;
  }
}
```

### 14.4 前端SDK TypeScript 实现
Airymax前端SDK需要与后端架构保持一致性：

#### 14.4.1 统一状态管理（映射原则：S-2 模块化设计）
```typescript
/**
 * 统一状态管理器 - 体现系统观（S-2）和工程观（E-2）原则
 * 
 * 基于Redux模式的状态管理，支持时间旅行调试和持久化。
 * 集成与Backs模块的实时同步。
 */
export class UnifiedStateManager {
  private store: Store<AppState>;
  private readonly middleware: Middleware[];
  private readonly syncService: StateSyncService;
  
  constructor(manager: StateConfig) {
    // 创建Redux store
    this.store = createStore(
      rootReducer,
      manager.initialState,
      applyMiddleware(...this.middleware)
    );
    
    // 集成同步服务
    this.syncService = new StateSyncService({
      backendUrl: manager.backendUrl,
      syncInterval: manager.syncInterval
    });
    
    // 设置状态持久化
    if (manager.persist) {
      this.setupPersistence(manager.persistenceKey);
    }
    
    // 启动自动同步
    if (manager.autoSync) {
      this.startAutoSync();
    }
  }
  
  /**
   * 分派动作 - 体现类型安全和错误恢复
   */
  dispatch<T extends Action>(action: T): T {
    try {
      // 验证动作格式
      this.validateAction(action);
      
      // 分派到store
      const result = this.store.dispatch(action);
      
      // 异步同步到后端
      if (this.shouldSync(action)) {
        this.syncService.sync(action).catch(error => {
          this.logger.warn('Sync failed', { action, error });
        });
      }
      
      return result;
      
    } catch (error) {
      this.logger.error('Dispatch failed', { action, error });
      throw new StateError('Dispatch failed', { cause: error });
    }
  }
  
  /**
   * 订阅状态变更 - 体现响应式编程模式
   */
  subscribe<T>(
    selector: (state: AppState) => T,
    listener: (value: T, previousValue: T) => void
  ): () => void {
    let previousValue = selector(this.store.getState());
    
    return this.store.subscribe(() => {
      const currentValue = selector(this.store.getState());
      if (!Object.is(previousValue, currentValue)) {
        listener(currentValue, previousValue);
        previousValue = currentValue;
      }
    });
  }
}
```

---
## 十五、参考文献

1. **Airymax 架构设计原则**: [ARCHITECTURAL_PRINCIPLES.md](../../Capital_Architecture/ARCHITECTURAL_PRINCIPLES.md)
2. **Google TypeScript Style Guide**: https://google.github.io/styleguide/tsguide.html
3. **TypeScript Documentation**: https://www.typescriptlang.org/docs/
4. **Airbnb JavaScript Style Guide**: https://github.com/airbnb/javascript
5. **Airymax 核心架构文档**:
   - [coreloopthree.md](../../Capital_Architecture/coreloopthree.md)
   - [memoryrovol.md](../../Capital_Architecture/memoryrovol.md)
   - [microcorert.md](../../Capital_Architecture/microcorert.md)
   - [ipc.md](../../Capital_Architecture/kernel/ipc.md)
   - [syscall.md](../../Capital_Architecture/syscall.md)
   - [logging.md](../../Capital_Architecture/services/logging.md)

---

## 附录：跨文档规范引用

本规范与以下 Airymax 工程规范一致，所有 JavaScript/TypeScript 代码须同时遵循：

| 规范集 | 说明 | 来源文档 |
|--------|------|---------|
| **BAN-01~13** | 13 项禁止模式（桩函数/假数据/空返回等） | [C编码规范 §18](./C_coding_style_standard.md) |
| **SDK-01~05** | 4 SDK 编译验证规范 | [工程规范化标准手册 v10.5](../../../Docs-closed/工程规范化标准手册08.md) |
| **标准术语** | 8 个架构组件标准名称 | [TERMINOLOGY.md](../../Capital_Specifications/TERMINOLOGY.md) |

**关键术语映射**（需在代码和文档中使用标准名称）：
- `daemon` → **用户态服务层**（禁止使用"守护进程"）
- `coreloopthree` → **认知循环运行时**
- `memoryrovol` → **记忆卷载**
- `triple_coordinator` → **认知层双思考功能**

---

© 2025-2026 SPHARX Ltd. All Rights Reserved.  
"From data intelligence emerges."