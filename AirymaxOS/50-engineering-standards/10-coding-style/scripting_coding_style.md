Copyright (c) 2025-2026 SPHARX Ltd. All Rights Reserved.

# 脚本语言编码风格规范
> **文档定位**：Python/JavaScript/Java 脚本与用户态语言编码风格及安全编码规范合集\
> **文档版本**：0.1.1\
> **最后更新**： 2026-07-21\
> **上级文档**：[agentrt-linux（AirymaxOS）工程标准规范](README.md)

---

## Part I: Airymax Python 编码规范

### 一、概述

#### 1.1 编制目的

本规范为 Airymax 项目中的 Python 代码提供统一的编码标准。基于项目架构设计原则的五维正交系统，本规范聚焦于工程观维度（E-1至E-4），为开发者提供可操作的代码实现指南。

#### 1.2 理论基础

本规范基于 Airymax 架构设计原则的五维正交系统：

- **《工程控制论》**（原则 S-1, E-2）：通过错误处理、日志、健康检查构建反馈闭环
- **《论系统工程》**（原则 S-2）：模块化、接口驱动、边界清晰
- **Thinkdual 双思考系统**（原则 C-1）：Python 简洁语法（t1 快思考）与类型提示（t2 慢思考）

**双思考系统在 Python 中的体现**:

| 场景 | t1 快思考（快速） | t2 慢思考（严谨） |
|------|-----------------|-----------------|
| 类型系统 | 动态类型 | 类型提示 (typing) |
| 错误处理 | 运行时异常 | 静态类型检查 (mypy) |
| 开发体验 | 快速原型 | 完整类型定义 |
| 性能优化 | 解释执行 | Cython/Numba 编译 |

#### 1.3 适用范围

| 组件 | Python 版本 | 类型检查 |
|------|-------------|----------|
| agentrt/daemon/ | Python 3.11+ | mypy |
| SDK Python | Python 3.10+ | pyright |
| 工具脚本 | Python 3.9+ | - |

#### 1.4 与 Airymax 架构的关系

Python 代码在 Airymax 中主要应用于以下场景：

| 场景 | 位置 | 关联原则 | Python 特性 |
|------|------|---------|------------|
| 用户态服务层 | `agentrt/daemon/` | S-3, K-3 | asyncio, multiprocessing |
| 工具脚本 | `agentrt/toolkit/` | E-7 | 快速原型，脚本自动化 |
| SDK Python | `sdks/python/` | K-2, E-2 | 类型提示，数据类 |
| 机器学习 | `ml/` | C-1, C-4 | NumPy, PyTorch |

**层次纪律**（原则 S-2）:
- 用户态服务层 Python 代码必须通过 `syscalls.h` 的 C API 绑定与内核交互
- 禁止 Python 代码直接访问内核内部结构
- 所有跨语言边界必须进行参数验证和类型转换

---

### 二、文件组织

#### 2.1 文件命名

- **Python 文件**：`.py` 扩展名，采用 `snake_case`
- **包目录**：包含 `__init__.py`
- **测试文件**：`test_*.py` 或 `*_test.py`

```
src/
├── agents/
│   ├── __init__.py
│   ├── task_scheduler.py
│   ├── memory_manager.py
│   └── cognition_engine.py
├── types/
│   ├── __init__.py
│   ├── agent_types.py
│   └── task_types.py
└── utils/
    ├── __init__.py
    ├── error_handler.py
    └── logger.py
```

#### 2.2 模块结构

```python
"""
Airymax 任务调度模块。

提供任务调度核心功能，包括任务提交、状态管理、
优先级队列和依赖解析。调度器采用双思考系统架构：
- t1 快思考：快速路径，处理简单任务
- t2 慢思考：深度路径，处理复杂任务

Example:
    >>> from agentrt.scheduler import TaskScheduler
    >>> scheduler = TaskScheduler(max_workers=4)
    >>> task_id = scheduler.submit(plan)
    >>> result = scheduler.wait(task_id)

Author:
    SPHARX Ltd. - Airymax Team

Version:
    1.5.0
"""

# 1. 标准库导入
from __future__ import annotations
import logging
from typing import Optional, List, Dict, Any
from dataclasses import dataclass, field
from enum import Enum, auto

# 2. 第三方库导入
import pydantic
from pydantic import BaseModel, Field

# 3. 项目内部导入
from agentrt.types import TaskPlan, TaskResult
from agentrt.errors import SchedulerError
from agentrt.constants import DEFAULT_TIMEOUT

# 4. 公开导出
__all__ = ["TaskScheduler", "SchedulerConfig"]

# 5. 模块级变量和常量
logger = logging.getLogger(__name__)
DEFAULT_MAX_WORKERS = 4
```

---

### 三、命名规范

#### 3.1 命名风格

| 类型 | 风格 | 示例 |
|------|------|------|
| 模块/包名 | snake_case | `task_scheduler.py`, `agentrt` |
| 类名 | PascalCase | `class TaskScheduler`, `@dataclass TaskConfig` |
| 函数/方法 | snake_case | `submit_task()`, `get_task_status()` |
| 变量 | snake_case | `task_id`, `max_workers` |
| 常量 | UPPER_SNAKE_CASE | `MAX_RETRY_COUNT`, `DEFAULT_TIMEOUT` |
| 类型别名 | PascalCase | `TaskId = str`, `MetricsDict = Dict[str, float]` |
| 私有属性 | _leading_underscore | `_internal_state` |

#### 3.2 命名示例

```python
# 正确的命名
MAX_TASK_NAME_LENGTH = 256
task_registry: Dict[str, Task] = {}
task_id = "task-001"

class TaskScheduler:
    """任务调度器。"""
    
    def __init__(self, max_workers: int = 4) -> None:
        self.max_workers = max_workers
        self._internal_state: Optional[State] = None

# 枚举
class TaskStatus(Enum):
    IDLE = auto()
    PENDING = auto()
    RUNNING = auto()
    COMPLETED = auto()
    FAILED = auto()

# 类型别名
TaskId = str
Timestamp = float
ByteBuffer = bytes

# 错误的命名
TaskRegistry = {}  # 类名应该 PascalCase
maxTaskNameLength = 256  # 常量应该 UPPER_SNAKE_CASE
def SubmitTask():  # 函数名应该 snake_case
    pass
```

---

### 四、类型设计

#### 4.1 类型别名

```python
from typing import TypeAlias, Callable, Awaitable

# 基本类型别名
TaskId: TypeAlias = str
Timestamp: TypeAlias = float
ByteBuffer: TypeAlias = bytes

# 函数类型
TaskHandler: TypeAlias = Callable[[Any], Awaitable[Any]]
ErrorHandler: TypeAlias = Callable[[Exception], None]

# 复杂类型
MetricsDict: TypeAlias = Dict[str, float]
ConfigDict: TypeAlias = Dict[str, Any]
ResultTuple: TypeAlias = tuple[bool, Optional[str]]
```

#### 4.2 数据类

```python
from dataclasses import dataclass, field
from typing import Optional, List
from enum import Enum, auto

class TaskStrategy(Enum):
    """任务执行策略。"""
    SEQUENTIAL = auto()
    PARALLEL = auto()
    ADAPTIVE = auto()

@dataclass
class TaskConfig:
    """任务配置。
    
    Attributes:
        name: 任务名称
        timeout: 超时时间（秒）
        max_retries: 最大重试次数
        strategy: 执行策略
        tags: 任务标签
    """
    name: str
    timeout: float = 30.0
    max_retries: int = 3
    strategy: TaskStrategy = TaskStrategy.SEQUENTIAL
    tags: List[str] = field(default_factory=list)
    
    def __post_init__(self) -> None:
        """验证配置参数。"""
        if self.timeout <= 0:
            raise ValueError(f"timeout must be positive, got {self.timeout}")
        if self.max_retries < 0:
            raise ValueError(f"max_retries must be non-negative, got {self.max_retries}")

@dataclass
class TaskResult:
    """任务结果。
    
    Attributes:
        task_id: 任务 ID
        success: 是否成功
        output: 输出数据
        error: 错误信息
        duration: 执行时长（秒）
    """
    task_id: str
    success: bool
    output: Optional[bytes] = None
    error: Optional[str] = None
    duration: Optional[float] = None
```

#### 4.3 Pydantic 模型

```python
from pydantic import BaseModel, Field, field_validator
from typing import Optional, List

class TaskPlanSchema(BaseModel):
    """任务计划 schema。
    
    Attributes:
        id: 任务唯一标识符
        name: 任务名称
        steps: 执行步骤列表
        strategy: 执行策略
    """
    id: str = Field(..., min_length=1, max_length=64, description="任务 ID")
    name: str = Field(..., min_length=1, max_length=256, description="任务名称")
    steps: List[StepSchema] = Field(..., min_length=1, description="执行步骤")
    strategy: str = Field(default="sequential", pattern="^(sequential|parallel|adaptive)$")
    
    @field_validator("id")
    @classmethod
    def validate_id(cls, v: str) -> str:
        """验证任务 ID 格式。"""
        if not v.replace("-", "").replace("_", "").isalnum():
            raise ValueError(f"Invalid task ID format: {v}")
        return v
    
    model_config = {
        "json_schema_extra": {
            "example": {
                "id": "task-001",
                "name": "data-processing",
                "steps": [],
                "strategy": "parallel"
            }
        }
    }
```

---

### 五、函数设计

#### 5.1 函数签名规范

```python
from typing import Optional, List, Dict, Any
import logging

logger = logging.getLogger(__name__)

def submit_task(
    plan: TaskPlan,
    priority: int = 5,
    timeout: Optional[float] = None,
    tags: Optional[Dict[str, str]] = None,
) -> str:
    """提交任务计划。
    
    将任务计划加入调度队列，返回任务 ID 用于后续查询。
    调度器会根据优先级和依赖关系决定执行顺序。
    
    Args:
        plan: 任务计划对象
        priority: 任务优先级，1-10，默认 5
        timeout: 超时时间（秒），None 表示使用默认值
        tags: 任务标签字典，用于分类和过滤
        
    Returns:
        任务 ID 字符串
        
    Raises:
        ValueError: 当 priority 不在 1-10 范围内
        TypeError: 当 plan 不是 TaskPlan 类型
        SchedulerError: 当调度器已关闭或队列已满
        
    Example:
        >>> plan = TaskPlan(id="task-001", name="test")
        >>> task_id = submit_task(plan, priority=8)
        >>> print(task_id)
        'task-001'
        
    Note:
        - 提交后任务进入 PENDING 状态
        - 高优先级任务可能抢占低优先级任务
    """
    # 参数验证
    if not isinstance(priority, int) or priority < 1 or priority > 10:
        raise ValueError(f"priority must be in [1, 10], got {priority}")
    
    # 实现...
```

#### 5.2 异步函数

```python
import asyncio
from typing import Optional
from dataclasses import dataclass

async def wait_for_task(
    task_id: str,
    timeout: Optional[float] = None,
) -> TaskResult:
    """等待任务完成。
    
    异步等待任务执行完成或超时。
    
    Args:
        task_id: 任务 ID
        timeout: 超时时间（秒），None 表示无限等待
        
    Returns:
        任务结果
        
    Raises:
        asyncio.TimeoutError: 当等待超时
        TaskNotFoundError: 当任务不存在
    """
    if timeout is not None and timeout <= 0:
        raise ValueError(f"timeout must be positive, got {timeout}")
    
    start_time = asyncio.get_event_loop().time()
    
    while True:
        result = await _get_task_result(task_id)
        
        if result is not None:
            return result
        
        if timeout is not None:
            elapsed = asyncio.get_event_loop().time() - start_time
            if elapsed >= timeout:
                raise asyncio.TimeoutError(f"Task {task_id} timed out after {timeout}s")
        
        await asyncio.sleep(0.1)  # 轮询间隔
```

#### 5.3 返回类型约定

```python
from typing import TypeAlias, Generic, TypeVar, Union, Optional
from dataclasses import dataclass

# Result 类型
T = TypeVar("T")
E = TypeVar("E")

@dataclass
class Result(Generic[T, E]):
    """结果类型，支持成功/失败两种状态。"""
    success: bool
    value: Optional[T] = None
    error: Optional[E] = None
    
    @classmethod
    def ok(cls, value: T) -> "Result[T, None]":
        """创建成功结果。"""
        return cls(success=True, value=value)
    
    @classmethod
    def err(cls, error: E) -> "Result[None, E]":
        """创建错误结果。"""
        return cls(success=False, error=error)

# 使用示例
def process_task(task_id: str) -> Result[TaskResult, SchedulerError]:
    try:
        result = scheduler.execute(task_id)
        return Result.ok(result)
    except SchedulerError as e:
        return Result.err(e)
```

---

### 六、类设计

#### 6.1 类结构模板

```python
import logging
from typing import Optional, List, Dict, Any
from dataclasses import dataclass, field
from abc import ABC, abstractmethod

logger = logging.getLogger(__name__)

class SchedulerBase(ABC):
    """调度器基类。
    
    定义调度器的抽象接口。子类必须实现所有抽象方法。
    
    Attributes:
        name: 调度器名称
        max_workers: 最大工作线程数
    """
    
    def __init__(self, name: str, max_workers: int) -> None:
        """初始化调度器。
        
        Args:
            name: 调度器名称
            max_workers: 最大工作线程数，必须为正整数
            
        Raises:
            ValueError: 当 max_workers <= 0 时
        """
        if max_workers <= 0:
            raise ValueError(f"max_workers must be positive, got {max_workers}")
        
        self.name = name
        self.max_workers = max_workers
        self._tasks: Dict[str, Any] = {}
        logger.info(f"Scheduler {name} initialized with {max_workers} workers")
    
    @abstractmethod
    async def submit(self, plan: TaskPlan) -> str:
        """提交任务计划。"""
        pass
    
    @abstractmethod
    async def wait(self, task_id: str, timeout: Optional[float] = None) -> TaskResult:
        """等待任务完成。"""
        pass
    
    @abstractmethod
    async def cancel(self, task_id: str) -> bool:
        """取消任务。"""
        pass

class TaskScheduler(SchedulerBase):
    """任务调度器。
    
    负责管理任务的生命周期和执行。调度器采用双思考系统架构，
    t1 快思考 处理简单任务，t2 慢思考 处理复杂任务。
    
    Example:
        >>> scheduler = TaskScheduler(name="main", max_workers=4)
        >>> task_id = await scheduler.submit(plan)
        >>> result = await scheduler.wait(task_id)
    """
    
    def __init__(
        self,
        name: str = "default",
        max_workers: int = 4,
        enable_retry: bool = True,
    ) -> None:
        """初始化任务调度器。
        
        Args:
            name: 调度器名称
            max_workers: 最大工作线程数
            enable_retry: 是否启用自动重试
        """
        super().__init__(name, max_workers)
        self.enable_retry = enable_retry
        self._retry_policy = RetryPolicy()
    
    async def submit(self, plan: TaskPlan) -> str:
        """提交任务计划。"""
        task_id = plan.id
        self._tasks[task_id] = TaskContext(plan=plan, status=TaskStatus.PENDING)
        logger.debug(f"Task {task_id} submitted")
        return task_id
    
    async def wait(self, task_id: str, timeout: Optional[float] = None) -> TaskResult:
        """等待任务完成。"""
        # 实现...
        pass
    
    async def cancel(self, task_id: str) -> bool:
        """取消任务。"""
        if task_id not in self._tasks:
            return False
        self._tasks[task_id].status = TaskStatus.CANCELLED
        return True
```

#### 6.2 上下文管理器

```python
from contextlib import contextmanager
from typing import Generator
import logging

logger = logging.getLogger(__name__)

class ResourceManager:
    """资源管理器。
    
    提供资源获取和释放的上下文管理器支持。
    """
    
    def __init__(self, manager: ResourceConfig) -> None:
        self.manager = manager
        self._resource = None
    
    def __enter__(self) -> "ResourceManager":
        """获取资源。"""
        self._resource = self._acquire()
        logger.debug(f"Resource acquired: {self._resource}")
        return self
    
    def __exit__(self, exc_type, exc_val, exc_tb) -> None:
        """释放资源。"""
        if self._resource is not None:
            self._release(self._resource)
            logger.debug(f"Resource released")
    
    @contextmanager
    def transaction(self) -> Generator[Transaction, None, None]:
        """事务上下文管理器。"""
        tx = Transaction()
        try:
            yield tx
            tx.commit()
        except Exception as e:
            tx.rollback()
            raise
```

---

### 七、错误处理

#### 7.1 异常类定义

```python
class AgentRTError(Exception):
    """Airymax 错误基类。"""
    
    def __init__(self, message: str, code: str = "AIRY_ERROR") -> None:
        self.message = message
        self.code = code
        super().__init__(self.message)

class SchedulerError(AgentRTError):
    """调度器错误。"""
    pass

class SchedulerClosedError(SchedulerError):
    """调度器已关闭错误。"""
    
    def __init__(self) -> None:
        super().__init__("Scheduler is closed", "SCHEDULER_CLOSED")

class TaskNotFoundError(SchedulerError):
    """任务不存在错误。"""
    
    def __init__(self, task_id: str) -> None:
        super().__init__(f"Task not found: {task_id}", "TASK_NOT_FOUND")
        self.task_id = task_id

class ValidationError(AgentRTError):
    """验证错误。"""
    pass

class ResourceError(AgentRTError):
    """资源错误。"""
    pass
```

#### 7.2 错误处理模式

```python
from typing import Optional, Callable
import logging
import functools

logger = logging.getLogger(__name__)

def handle_errors(
    error_type: type[Exception] = Exception,
    default_return: Optional[Any] = None,
    log_traceback: bool = True,
) -> Callable:
    """错误处理装饰器。"""
    def decorator(func: Callable) -> Callable:
        @functools.wraps(func)
        def wrapper(*args, **kwargs):
            try:
                return func(*args, **kwargs)
            except error_type as e:
                if log_traceback:
                    logger.exception(f"Error in {func.__name__}: {e}")
                else:
                    logger.error(f"Error in {func.__name__}: {e}")
                return default_return
        return wrapper
    return decorator

async def safe_execute(
    func: Callable,
    *args,
    **kwargs,
) -> Result:
    """安全执行异步函数。"""
    try:
        result = await func(*args, **kwargs)
        return Result.ok(result)
    except Exception as e:
        logger.exception(f"Execution failed: {e}")
        return Result.err(e)
```

---

### 八、异步编程

#### 8.1 异步上下文管理器

```python
import asyncio
from contextlib import asynccontextmanager
from typing import AsyncGenerator

class AsyncResourcePool:
    """异步资源池。"""
    
    def __init__(self, max_size: int = 10) -> None:
        self.max_size = max_size
        self._available: asyncio.Queue = asyncio.Queue(max_size)
        self._used: set = set()
    
    @asynccontextmanager
    async def acquire(self) -> AsyncGenerator:
        """获取资源。"""
        resource = await self._available.get()
        self._used.add(resource)
        try:
            yield resource
        finally:
            self._used.remove(resource)
            await self._available.put(resource)
    
    async def __aenter__(self) -> "AsyncResourcePool":
        """进入异步上下文。"""
        return self
    
    async def __aexit__(self, exc_type, exc_val, exc_tb) -> None:
        """退出异步上下文。"""
        while not self._available.empty():
            self._available.get_nowait()
```

#### 8.2 并发控制

```python
import asyncio
from typing import List, TypeVar

T = TypeVar("T")

async def execute_with_concurrency(
    tasks: List[asyncio.Task[T]],
    max_concurrency: int,
) -> List[T]:
    """限制并发数执行任务。
    
    Args:
        tasks: 任务列表
        max_concurrency: 最大并发数
        
    Returns:
        所有任务的结果列表
    """
    semaphore = asyncio.Semaphore(max_concurrency)
    
    async def bounded_task(task: asyncio.Task[T]) -> T:
        async with semaphore:
            return await task
    
    bounded_tasks = [bounded_task(t) for t in tasks]
    return await asyncio.gather(*bounded_tasks)

async def execute_parallel(
    functions: List[Callable],
    concurrency: int = 10,
) -> List:
    """并行执行函数，限制并发数。"""
    semaphore = asyncio.Semaphore(concurrency)
    
    async def run_with_limit(func: Callable) -> Any:
        async with semaphore:
            return await func()
    
    return await asyncio.gather(*[run_with_limit(f) for f in functions])
```

---

### 九、模块与导入

#### 9.1 导入语句顺序

```python
# 1. __future__ 导入（必须是第一行）
from __future__ import annotations

# 2. 标准库
import logging
from typing import Optional, List, Dict, Any, Callable
from dataclasses import dataclass, field
from enum import Enum, auto

# 3. 第三方库
import pydantic
from pydantic import BaseModel, Field
import numpy as np

# 4. 项目内部
from agentrt.types import TaskPlan, TaskResult
from agentrt.errors import SchedulerError
from agentrt.constants import DEFAULT_TIMEOUT

# 5. 类型导入（使用 type: ignore 类型注释）
from agentrt.manager import manager  # type: ignore[attr-defined]
```

#### 9.2 导出模式

```python
# __init__.py
"""Airymax 调度模块。

Example:
    >>> from agentrt.scheduler import TaskScheduler
    >>> scheduler = TaskScheduler()
"""

# 显式导出
__all__ = [
    "TaskScheduler",
    "SchedulerConfig",
    "SchedulerError",
    "SchedulerClosedError",
    "TaskNotFoundError",
]

# 导入并重新导出
from agentrt.scheduler.scheduler import TaskScheduler
from agentrt.scheduler.manager import SchedulerConfig
from agentrt.scheduler.errors import (
    SchedulerError,
    SchedulerClosedError,
    TaskNotFoundError,
)
```

---

### 十、测试集成

#### 10.1 pytest 示例

```python
"""
Airymax 任务调度器测试。
"""

import pytest
import asyncio
from typing import Optional
from agentrt.scheduler import TaskScheduler, SchedulerConfig, TaskStatus

@pytest.fixture
def scheduler() -> TaskScheduler:
    """创建测试用调度器。"""
    return TaskScheduler(name="test", max_workers=2)

@pytest.fixture
def sample_plan() -> TaskPlan:
    """创建示例任务计划。"""
    return TaskPlan(
        id="task-001",
        name="test-task",
        steps=[],
        strategy="sequential"
    )

class TestTaskScheduler:
    """TaskScheduler 测试类。"""
    
    def test_submit_returns_task_id(self, scheduler: TaskScheduler, sample_plan: TaskPlan) -> None:
        """测试提交任务返回任务 ID。"""
        task_id = asyncio.run(scheduler.submit(sample_plan))
        assert task_id == "task-001"
    
    def test_submit_with_priority(self, scheduler: TaskScheduler, sample_plan: TaskPlan) -> None:
        """测试带优先级的任务提交。"""
        task_id = asyncio.run(scheduler.submit(sample_plan, priority=8))
        assert task_id == "task-001"
    
    def test_invalid_priority_raises_error(self, scheduler: TaskScheduler, sample_plan: TaskPlan) -> None:
        """测试无效优先级抛出错误。"""
        with pytest.raises(ValueError, match="priority must be in"):
            asyncio.run(scheduler.submit(sample_plan, priority=15))
    
    @pytest.mark.asyncio
    async def test_wait_returns_result(self, scheduler: TaskScheduler, sample_plan: TaskPlan) -> None:
        """测试等待返回结果。"""
        task_id = await scheduler.submit(sample_plan)
        result = await scheduler.wait(task_id, timeout=5.0)
        assert result.success is True

@pytest.mark.parametrize("strategy", [
    "sequential",
    "parallel",
    "adaptive"
])
def test_scheduler_with_strategy(strategy: str, scheduler: TaskScheduler) -> None:
    """参数化测试：不同策略。"""
    assert scheduler.strategy == strategy
```

---

### 十-A、C FFI 绑定规范

#### 10-A.1 FFI 框架选择

| 场景 | 推荐框架 | 理由 |
|------|---------|------|
| 简单绑定（少量函数、基本类型） | `ctypes` | 标准库内置，零依赖，适合简单 C 函数调用 |
| 复杂绑定（结构体嵌套、回调、指针运算） | `cffi` | 性能更优，支持 ABI 和 API 两种模式，复杂类型声明更清晰 |

#### 10-A.2 绑定代码命名与组织

- **文件命名**：`airy_<module>_ffi.py`，例如 `airy_scheduler_ffi.py`、`airy_memory_ffi.py`
- **目录位置**：与 C 头文件对应，放置在 `sdks/python/agentrt/` 下
- **导出规范**：FFI 模块仅暴露 Pythonic 接口，不暴露原始 C 指针

```python
# airy_scheduler_ffi.py
"""Airymax 调度器 FFI 绑定。"""

from ctypes import CDLL, c_int, c_char_p, POINTER, byref
from agentrt.errors import AgentRTError, SchedulerError

# 加载共享库
_lib = CDLL("libairy_scheduler.so")

# 函数签名声明
_lib.cognition_schedule.argtypes = [c_char_p, c_char_p, POINTER(c_char_p)]
_lib.cognition_schedule.restype = c_int


def schedule(plan_json: str) -> str:
    """提交调度计划（Pythonic 封装）。

    Args:
        plan_json: 计划 JSON 字符串

    Returns:
        任务 ID

    Raises:
        SchedulerError: 调度失败
    """
    out_id = c_char_p()
    rc = _lib.cognition_schedule(
        plan_json.encode("utf-8"),
        None,  # 默认配置
        byref(out_id)
    )
    if rc != 0:
        raise _convert_error(rc, "cognition_schedule")
    result = out_id.value.decode("utf-8")
    _lib.airy_free(out_id)  # C 侧分配的内存由 C 侧释放
    return result
```

#### 10-A.3 错误码转换

所有 `AIRY_E*` C 错误码必须转换为 Python 异常：

```python
# agentrt/errors/ffi_errors.py
"""FFI 错误码到 Python 异常的映射。"""

from agentrt.errors import AgentRTError, ValidationError, ResourceError

# 错误码 → 异常类映射表
_ERROR_CODE_MAP: dict[int, type[AgentRTError]] = {
    0: None,                          # AIRY_EOK
    -2: ValidationError,              # AIRY_EINVAL
    -4: ResourceError,                # AIRY_ENOMEM
    -8: ResourceError,                # AIRY_ETIMEDOUT
    -17: ResourceError,               # AIRY_EBUSY
}


def _convert_error(rc: int, context: str) -> AgentRTError:
    """将 C 错误码转换为 Python 异常。

    Args:
        rc: C 函数返回的错误码
        context: 调用上下文描述

    Returns:
        对应的 Python 异常实例
    """
    exc_class = _ERROR_CODE_MAP.get(rc, AgentRTError)
    return exc_class(f"[{context}] C error code: {rc}")
```

#### 10-A.4 FFI 内存所有权规则

| 所有权 | 规则 | 示例 |
|--------|------|------|
| C → Python（C 分配） | Python 使用后必须调用 C 侧释放函数 | `_lib.airy_free(out_id)` |
| Python → C（Python 分配） | Python 保持引用，C 侧不得释放 | `byref(c_buffer)` |
| 共享缓冲区 | 明确文档化生命周期，使用 `memoryview` 避免拷贝 | `memoryview(c_array)` |

> **关键原则**：谁分配谁释放。跨 FFI 边界传递指针时，必须在文档中明确所有权归属。

---

### 十-B、Python 禁止模式清单（BAN-180~186）

所有 Python PR 必须通过以下禁止模式检查：

| 编号 | 禁止模式 | 检测方式 | 替代方案 |
|------|---------|----------|---------|
| BAN-180 | 禁止 `bare except:` | `ruff rule E722` / `flake8 E722` | 使用 `except Exception:` 并记录日志 |
| BAN-185 | 禁止 `bare except:` | `ruff rule E722` | 必须指定异常类型：`except ValueError:` 或 `except Exception:` |
| BAN-186 | 禁止 `except` 块中 `pass` 且无日志 | 代码审查 / 自定义 ruff 规则 | 至少记录 `logger.warning()` 或 `logger.exception()` |
| BAN-187 | 禁止生产代码中使用 `print()` | `ruff rule T201` | 使用 `logging` 模块：`logger.info()` / `logger.debug()` |

**示例**：

```python
# ❌ BAN-185: 裸 except
try:
    result = do_something()
except:  # 禁止！
    pass

# 正确：指定异常类型并记录
try:
    result = do_something()
except ValueError as e:
    logger.warning("Invalid value: %s", e)

# ❌ BAN-186: except 块中 pass 且无日志
try:
    result = do_something()
except Exception:
    pass  # 禁止！吞掉异常且无记录

# 正确：记录异常
try:
    result = do_something()
except Exception as e:
    logger.exception("Failed to do something: %s", e)
    raise

# ❌ BAN-187: 生产代码中使用 print()
print(f"Task {task_id} completed")  # 禁止！

# 正确：使用 logging
logger.info("Task %s completed", task_id)
```

---

### 十一、Airymax 模块 Python 编码示例

#### 11.1 daemon（守护层）Python 实现
Backs模块作为系统用户态服务，需要高可靠性和可观测性：

##### 11.1.1 IPC通信用户态服务（映射原则：E-3 通信基础设施）
```python
"""
IPC通信用户态服务 - 体现系统观（S-3）和工程观（E-4）原则

实现与Atoms模块的高性能进程间通信。
集成OpenTelemetry可观测性和消息加密。
"""
import asyncio
from typing import Optional
from dataclasses import dataclass
from contextlib import asynccontextmanager

@dataclass
class SecureIpcMessage:
    """安全IPC消息 - 体现防御深度（D-3）原则"""
    message_id: str
    sender: str
    operation: str
    payload: bytes
    signature: bytes
    timestamp: float
    
    def validate(self) -> bool:
        """消息验证 - 多层安全校验"""
        # 层次1：消息完整性
        if not self.verify_signature():
            return False
        
        # 层次2：时间戳有效性（防重放）
        if not self.check_timestamp():
            return False
        
        # 层次3：操作权限验证
        if not self.validate_operation():
            return False
        
        return True
    
    def verify_signature(self) -> bool:
        """验证消息签名"""
        # 实际签名验证逻辑
        return True

class IpcDaemon:
    """IPC用户态服务 - 体现工程观（E-2）和设计美学（A-1）原则"""
    
    def __init__(self, manager: dict):
        self.manager = manager
        self.logger = self.setup_logger()
        self.metrics = self.setup_metrics()
        self.crypto = self.setup_crypto()
        
    async def process_message(self, message: SecureIpcMessage) -> dict:
        """处理安全消息 - 结构化错误处理和审计"""
        self.logger.info(
            "IPC message received",
            extra={
                "message_id": message.message_id,
                "sender": message.sender,
                "operation": message.operation,
                "size": len(message.payload),
                "log_type": "ipc_request"
            }
        )
        
        try:
            # 消息验证
            if not message.validate():
                self.logger.warning(
                    "Invalid IPC message",
                    extra={"message_id": message.message_id, "log_type": "security_audit"}
                )
                raise SecurityError("Message validation failed")
            
            # 处理消息
            result = await self._do_process(message)
            
            # 成功日志
            self.logger.debug(
                "IPC message processed",
                extra={
                    "message_id": message.message_id,
                    "duration_ms": result.get("duration_ms", 0),
                    "log_type": "ipc_response"
                }
            )
            
            return result
            
        except SecurityError as e:
            # 安全异常详细记录
            self.logger.error(
                "IPC security violation",
                extra={
                    "message_id": message.message_id,
                    "error": str(e),
                    "log_type": "security_audit"
                },
                exc_info=True
            )
            raise
            
        except Exception as e:
            # 一般异常处理
            self.logger.error(
                "IPC processing failed",
                extra={
                    "message_id": message.message_id,
                    "error": str(e),
                    "log_type": "error"
                },
                exc_info=True
            )
            raise
    
    @asynccontextmanager
    async def secure_session(self, client_id: str):
        """安全会话上下文管理器 - 体现资源确定性原则"""
        session_key = await self.crypto.establish_session(client_id)
        try:
            yield session_key
        finally:
            await self.crypto.close_session(client_id)
            self.logger.debug(f"Session closed for client: {client_id}")
```

#### 11.2 commons（公共库层）Python 实现
Common模块提供跨层基础设施，强调通用性和性能：

##### 11.2.1 向量数据库客户端（映射原则：E-1 基础设施）
```python
"""
向量数据库客户端 - 体现工程观（E-1, E-3）和认知观（C-3）原则

封装FAISS和HNSW向量索引，支持持久化存储和拓扑分析。
集成HDBSCAN聚类算法，自动发现数据中的拓扑结构。
"""
import numpy as np
from typing import List, Optional, Tuple
from dataclasses import dataclass
from contextlib import contextmanager
from functools import lru_cache

@dataclass
class VectorSearchResult:
    """向量搜索结果 - 类型安全的数据结构"""
    id: str
    score: float
    metadata: dict
    vector: np.ndarray
    
    def __post_init__(self):
        """数据验证 - 体现防御性编程"""
        if not 0 <= self.score <= 1:
            raise ValueError(f"Invalid score: {self.score}")
        if len(self.vector.shape) != 1:
            raise ValueError(f"Vector must be 1D, got shape: {self.vector.shape}")

class VectorDBClient:
    """向量数据库客户端 - 高性能和类型安全设计"""
    
    def __init__(self, index_path: str, dimension: int):
        self.index_path = index_path
        self.dimension = dimension
        self.index = self._load_index()
        self.metadata_store = MetadataStore()
        self.cache = LRUCache(maxsize=1000)
        
    @lru_cache(maxsize=100)
    def search(
        self,
        query: np.ndarray,
        k: int = 10,
        **kwargs
    ) -> List[VectorSearchResult]:
        """
        向量搜索 - 性能优化和缓存友好
        
        参数:
            query: 查询向量，形状为(dimension,)
            k: 返回结果数量
            **kwargs: 搜索选项
            
        返回:
            搜索结果列表，按相似度排序
        """
        # 参数验证
        self._validate_query(query)
        if k <= 0:
            raise ValueError(f"k must be positive, got: {k}")
        
        # 缓存查询
        cache_key = self._hash_vector(query)
        if cached := self.cache.get(cache_key):
            return cached
        
        # 执行搜索
        start_time = time.perf_counter()
        indices, scores = self.index.search(query.reshape(1, -1), k)
        search_time = time.perf_counter() - start_time
        
        # 批量获取元数据
        metadata_list = self.metadata_store.batch_get(indices[0])
        
        # 构造结果
        results = [
            VectorSearchResult(
                id=str(idx),
                score=float(score),
                metadata=metadata,
                vector=self.index.reconstruct(idx)
            )
            for idx, score, metadata in zip(indices[0], scores[0], metadata_list)
        ]
        
        # 更新缓存
        self.cache[cache_key] = results
        
        # 记录性能指标
        self.metrics.record_search(search_time, len(results))
        
        return results
    
    def _validate_query(self, query: np.ndarray) -> None:
        """查询向量验证 - 防御性编程"""
        if query.shape != (self.dimension,):
            raise ValueError(
                f"Query shape mismatch: expected ({self.dimension},), "
                f"got {query.shape}"
            )
        if not np.isfinite(query).all():
            raise ValueError("Query vector contains NaN or infinite values")
```

---

### 十二、参考文献

1. **Airymax 架构设计原则**: [00-architectural-principles.md](../../10-architecture/00-architectural-principles.md)
2. **Google Python Style Guide**: https://google.github.io/styleguide/pyguide.html
3. **Python PEP 8**: https://www.python.org/dev/peps/pep-0008/
4. **Python PEP 484**: https://www.python.org/dev/peps/pep-0484/
5. **Airymax 核心架构文档**:
   - [coreloopthree.md](../../10-architecture/02-coreloopthree.md)
   - [memoryrovol.md](../../10-architecture/03-memoryrovol.md)
   - [microcorert.md](../../10-architecture/04-microcorert.md)
   - [ipc.md](../../10-architecture/30-kernel/01-ipc.md)
   - [logging.md](../../10-architecture/50-services/02-logging.md)

---

### 附录：跨文档规范引用

本规范与以下 Airymax 工程规范一致，所有 Python 代码须同时遵循：

| 规范集 | 说明 | 来源文档 |
|--------|------|---------|
| **BAN-01~13** | 13 项禁止模式（桩函数/假数据/空返回等） | [C编码规范 §18](./C_Cpp_coding_style.md) |
| **SDK-01~05** | 4 SDK 编译验证规范 | 工程规范化标准手册 v10.5 |
| **标准术语** | 8 个架构组件标准名称 | [90-terminology.md](../90-terminology.md) |

**关键术语映射**（需在代码和文档中使用标准名称）：
- `daemon` → **用户态服务层**（禁止使用"守护进程"）
- `coreloopthree` → **认知循环运行时**
- `memoryrovol` → **记忆卷载**
- `triple_coordinator` → **认知层双思考功能**

---

## Part II: Airymax JavaScript 编码规范

### 一、概述

#### 1.1 编制目的

本规范为 Airymax 项目中的 JavaScript/TypeScript 代码提供统一的编码标准。基于项目架构设计原则的五维正交系统，本规范聚焦于工程观维度（E-1至E-4），为开发者提供可操作的代码实现指南。

#### 1.2 理论基础

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

#### 1.3 适用范围

| 环境 | 标准 | 运行环境 |
|------|------|----------|
| Node.js | ES2022+ / TypeScript 5.0+ | Node.js 18+ |
| 浏览器 | ES2022 / TypeScript 5.0+ | ES2022 兼容浏览器 |
| SDK | ES2022 / TypeScript 5.0+ | Node.js 18+ |

---

### 二、文件组织

#### 2.1 文件命名

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

#### 2.2 模块结构

```typescript
/**
 * @fileoverview 任务调度器模块
 *
 * 提供任务调度核心功能，包括任务提交、状态管理、
 * 优先级队列和依赖解析。
 *
 * @module agentrt/scheduler
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

### 三、命名规范

#### 3.1 命名风格

| 类型 | 风格 | 示例 |
|------|------|------|
| 变量/函数 | camelCase | `taskId`, `submitTask()`, `getTaskStatus()` |
| 类/接口/类型 | PascalCase | `class TaskScheduler`, `interface TaskConfig` |
| 常量 | UPPER_SNAKE_CASE | `MAX_RETRY_COUNT`, `DEFAULT_TIMEOUT` |
| 枚举成员 | camelCase | `TaskStatus.Idle`, `Priority.Normal` |
| 文件名 | kebab-case | `task-scheduler.ts`, `memory-manager.js` |
| 目录名 | kebab-case | `task-scheduler/`, `memory-agentrt/manager/` |

#### 3.2 命名示例

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

### 四、类型设计

#### 4.1 类型别名

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

#### 4.2 接口定义

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

#### 4.3 枚举

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

### 五、函数设计

#### 5.1 函数签名规范

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

#### 5.2 参数验证

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

#### 5.3 返回值约定

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

### 六、类设计

#### 6.1 类结构模板

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

#### 6.2 私有成员命名

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

### 七、错误处理

#### 7.1 错误类定义

```typescript
/**
 * Airymax 错误基类
 */
export class AgentRTError extends Error {
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
export class SchedulerError extends AgentRTError {
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
export class ValidationError extends AgentRTError {
    public constructor(message: string) {
        super(message, 'VALIDATION_ERROR');
    }
}
```

#### 7.2 错误处理模式

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
        if (error instanceof AgentRTError) {
            logger.error(`Task ${taskId} failed: ${error.message}`, { code: error.code });
            throw error;
        }
        logger.error(`Unexpected error processing task ${taskId}`, error);
        throw new SchedulerError('Internal error');
    }
}
```

---

### 八、异步编程

#### 8.1 Promise 使用

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

#### 8.2 async/await 模式

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

### 九、模块与导入

#### 9.1 导入语句顺序

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

#### 9.2 导出模式

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

### 十、注释规范

#### 10.1 JSDoc 注释

```typescript
/**
 * @fileoverview 任务调度器模块
 *
 * 提供任务调度核心功能，包括任务提交、状态管理、
 * 优先级队列和依赖解析。调度器采用双思考系统架构：
 * - t1 快思考：快速路径，处理简单任务
 * - t2 慢思考：深度路径，处理复杂任务
 *
 * @module agentrt/scheduler
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

#### 10.2 TSDoc 类型注释

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

### 十一、类型安全

#### 11.1 严格模式

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

#### 11.2 类型守卫

```typescript
/**
 * 类型守卫：检查是否为 Airymax 错误
 */
function isAgentRTError(error: unknown): error is AgentRTError {
    return error instanceof AgentRTError;
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
    if (isAgentRTError(error)) {
        logger.error(`Airymax error: ${error.code} - ${error.message}`);
        // error.code 是确定存在的
    } else if (error instanceof Error) {
        logger.error(`Unexpected error: ${error.message}`);
    }
}
```

---

### 十二、测试集成

#### 12.1 Jest 测试示例

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

### 十三-A、TypeScript/JavaScript 禁止模式清单（BAN-300~305）

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

// 正确：使用具体类型或泛型
function process<T>(data: T): Result<T> { ... }

// ❌ BAN-301: 使用 @ts-ignore
// @ts-ignore
const result = someFunction();  // 禁止！

// 正确：修复类型或使用类型守卫
if (isValidData(data)) {
    const result = someFunction(data);
}

// ❌ BAN-302: 使用 console.log
console.log('Task completed');  // 禁止！

// 正确：使用 logger
import { logger } from '../utils/logger';
logger.info('Task completed', { taskId });

// ❌ BAN-303: 空 catch 块
try { ... } catch (e) { }  // 禁止！

// 正确：记录异常
try { ... } catch (e) {
    logger.error('Operation failed', { error: e });
}

// ❌ BAN-304: 硬编码 URL
const API_URL = 'http://localhost:8080/api';  // 禁止！

// 正确：使用配置
const API_URL = config.get('api.url');
```

---

### 十三-B、C 内核通信规范

#### 13-B.1 通信架构

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

#### 13-B.2 WebSocket/HTTP API 规范

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

#### 13-B.3 错误码映射

AIRY_E* 错误码到 HTTP 状态码的映射：

| AGENTRT 错误码 | HTTP 状态码 | 说明 |
|---------------|------------|------|
| `AIRY_EOK` (0) | 200 | 成功 |
| `AIRY_EINVAL` (-1) | 400 | 参数无效 |
| `AIRY_ENOMEM` (-2) | 503 | 资源不足 |
| `AIRY_ETIMEDOUT` (-11) | 408 / 504 | 请求超时 |
| `AIRY_EBUSY` (-9) | 429 | 服务繁忙 |
| `AIRY_ENOENT` | 404 | 资源不存在 |
| `AIRY_EPERM` | 403 | 权限不足 |
| 其他负值 | 500 | 内部错误 |

#### 13-B.4 JSON-RPC 2.0 服务层规范

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
 * -32000 ~ -32099: Server error（服务端自定义错误，映射 AIRY_E*）
 */
```

---

### 十四、Airymax 模块 JavaScript/TypeScript 编码示例

#### 14.1 daemon（守护层）TypeScript 实现
Backs模块作为系统用户态服务，需要高可靠性和可观测性：

##### 14.1.1 IPC 通信服务（映射原则：E-3 通信基础设施）
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

##### 14.1.2 任务监控用户态服务（映射原则：E-2 可维护性）
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

#### 14.2 cupolas（安全穹顶）TypeScript 实现
cupolas模块实现零信任安全模型，需要严格的安全保障：

##### 14.2.1 安全策略引擎（映射原则：D-1 最小权限）
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

#### 14.3 commons（公共库层）TypeScript 实现
Common模块提供跨层基础设施，强调通用性和性能：

##### 14.3.1 向量数据库客户端（映射原则：E-1 基础设施）
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

#### 14.4 前端SDK TypeScript 实现
Airymax前端SDK需要与后端架构保持一致性：

##### 14.4.1 统一状态管理（映射原则：S-2 模块化设计）
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
### 十五、参考文献

1. **Airymax 架构设计原则**: [00-architectural-principles.md](../../10-architecture/00-architectural-principles.md)
2. **Google TypeScript Style Guide**: https://google.github.io/styleguide/tsguide.html
3. **TypeScript Documentation**: https://www.typescriptlang.org/docs/
4. **Airbnb JavaScript Style Guide**: https://github.com/airbnb/javascript
5. **Airymax 核心架构文档**:
   - [coreloopthree.md](../../10-architecture/02-coreloopthree.md)
   - [memoryrovol.md](../../10-architecture/03-memoryrovol.md)
   - [microcorert.md](../../10-architecture/04-microcorert.md)
   - [ipc.md](../../10-architecture/30-kernel/01-ipc.md)
   - [syscall.md](../../10-architecture/05-syscall.md)
   - [logging.md](../../10-architecture/50-services/02-logging.md)

---

### 附录：跨文档规范引用

本规范与以下 Airymax 工程规范一致，所有 JavaScript/TypeScript 代码须同时遵循：

| 规范集 | 说明 | 来源文档 |
|--------|------|---------|
| **BAN-01~13** | 13 项禁止模式（桩函数/假数据/空返回等） | [C编码规范 §18](./C_Cpp_coding_style.md) |
| **SDK-01~05** | 4 SDK 编译验证规范 | 工程规范化标准手册 v10.5 |
| **标准术语** | 8 个架构组件标准名称 | [90-terminology.md](../90-terminology.md) |

**关键术语映射**（需在代码和文档中使用标准名称）：
- `daemon` → **用户态服务层**（禁止使用"守护进程"）
- `coreloopthree` → **认知循环运行时**
- `memoryrovol` → **记忆卷载**
- `triple_coordinator` → **认知层双思考功能**

---

## Part III: Airymax Java 安全编码指南

### 一、概述

#### 1.1 编制目的

本指南为 Airymax 项目中的 Java 代码提供安全编码标准。基于项目架构设计原则的五维正交系统，本指南聚焦于安全观维度（D-1至D-4安全工程原则），为开发者提供防止安全漏洞的编码实践。

#### 1.2 与 Airymax 架构的关系

基于架构设计原则**E-1（安全内生原则）**，Airymax 的安全穹顶（cupolas）采用四层防护体系：

| 层次 | 名称 | 组件路径 | 功能 | Java SDK 实现 |
|------|------|---------|------|--------------|
| D1 | **虚拟工位** | `agentrt/cupolas/workbench/` | 进程/容器级隔离 | 沙箱 ClassLoader |
| D2 | **权限裁决** | `agentrt/cupolas/permission/` | YAML 规则引擎 | SecurityManager |
| D3 | **输入净化** | `agentrt/cupolas/sanitizer/` | 正则过滤 | InputValidator |
| D4 | **审计追踪** | `agentrt/cupolas/audit/` | 全链路追踪 | AuditLogger |

**防护原则**:
- **纵深防御**: 四层防护相互独立，单层失效不影响整体安全（映射原则：D-3）
- **默认拒绝**: 未经明确允许的访问一律拒绝（映射原则：D-1）
- **失效安全**: 安全机制故障时默认进入安全状态（映射原则：D-4）

Java SDK 作为 Airymax 的重要组成部分，必须遵循这些安全原则，并在代码层面实现相应的安全机制。

#### 1.3 五维正交系统的安全视角

Airymax的五维正交系统为Java安全编码提供了多层次的理论框架：

##### 1.3.1 系统观（System View）安全
- **垂直分层安全**：Java SDK需遵循S-1（垂直分层）原则，在应用层、服务层、内核层间建立清晰的信任边界
- **模块化安全设计**：基于S-2原则，安全功能模块化，支持独立部署和升级

##### 1.3.2 内核观（Kernel View）安全
- **微核心安全模型**：Java代码需与microcorert.md中定义的微核心安全原语保持一致
- **系统调用保护**：Java Native Interface（JNI）调用必须严格验证，防止非法跨越用户态-内核态边界

##### 1.3.3 认知观（Cognitive View）安全
- **Thinkdual 认知偏差防护**：基于C-3原则，Java API设计需考虑开发者认知模式，防止安全配置错误
- **安全代码审查**：利用t2 慢思考（慢速路径）深度分析能力进行安全代码审查

##### 1.3.4 工程观（Engineering View）安全
- **安全工程化**：将D-1至D-4安全工程原则转化为具体的Java编码模式
- **安全基础设施**：基于E-1至E-4原则，构建可观测、可审计、可恢复的Java安全基础设施

##### 1.3.5 设计美学（Design Aesthetics）安全
- **安全即美感**：安全的Java代码应是简洁、优雅、易于理解的（映射原则：A-1）
- **细节中的安全**：安全漏洞常隐藏在细节中，必须贯彻A-2（细节关注）原则

#### 1.4 Thinkdual 双思考系统与安全编程

Airymax 的 Thinkdual 双思考系统为 Java 安全编码提供了心理学基础：

##### 1.4.1 t1 快思考（快速路径）安全
- **直觉安全设计**：常见安全操作应直观易懂，支持快速正确使用
- **默认安全行为**：API默认配置应是安全的，防止疏忽导致的安全漏洞
- **示例**：安全敏感的API应使安全用法成为最自然的选择

##### 1.4.2 t2 慢思考（慢速路径）安全
- **深度安全分析**：复杂安全决策需要t2 慢思考的深度分析能力
- **安全配置验证**：复杂安全配置需要明确的验证机制和文档
- **示例**：密钥管理系统需要详细的配置向导和安全检查

#### 1.5 安全原则

基于架构设计原则 E-1（安全内生原则），本指南遵循以下安全原则：

| 原则 | 说明 | 在 Airymax 中的体现 | 关联原则 |
|------|------|---------------------|---------|
| 最小权限 | 只授予完成任务所需的最小权限 | D2 权限裁决 | K-1 |
| 纵深防御 | 多层安全检查 | cupolas 四层防护 | S-2 |
| 输入验证 | 所有外部输入必须经过验证 | D3 输入净化 | E-1 |
| 安全默认值 | 默认配置应是安全的 | 默认拒绝策略 | E-1 |
| 故障安全 | 发生错误时默认拒绝操作 | 安全机制故障时拒绝访问 | E-6 |
| 可观测性 | 所有安全事件必须可追踪 | D4 审计追踪 | E-2 |

---

### 二、输入验证

#### 2.1 通用输入验证

```java
/**
 * 验证输入参数
 *
 * @param taskId 任务 ID
 * @param input 输入数据
 * @throws IllegalArgumentException 当输入无效时
 */
public void processTask(String taskId, byte[] input) {
    // 1. 基本空值检查
    if (taskId == null) {
        throw new IllegalArgumentException("Task ID cannot be null");
    }
    
    // 2. 格式验证
    if (!TASK_ID_PATTERN.matcher(taskId).matches()) {
        throw new IllegalArgumentException("Invalid task ID format: " + taskId);
    }
    
    // 3. 长度验证
    if (taskId.length() > MAX_TASK_ID_LENGTH) {
        throw new IllegalArgumentException("Task ID too long: " + taskId.length());
    }
    
    // 4. 内容验证（二进制数据）
    if (input != null && input.length > MAX_INPUT_SIZE) {
        throw new IllegalArgumentException("Input too large: " + input.length);
    }
    
    // 继续处理...
}

private static final Pattern TASK_ID_PATTERN = Pattern.compile("^[a-zA-Z0-9_-]{1,64}$");
private static final int MAX_TASK_ID_LENGTH = 64;
private static final int MAX_INPUT_SIZE = 10 * 1024 * 1024; // 10MB
```

#### 2.2 SQL 注入防护

```java
// 使用参数化查询
public List<Task> findTasksByStatus(String status) {
    // 正确：使用参数化查询
    String sql = "SELECT * FROM tasks WHERE status = ?";
    return jdbcTemplate.query(sql, (rs, rowNum) -> {
        Task task = new Task();
        task.setId(rs.getString("id"));
        task.setStatus(rs.getString("status"));
        return task;
    }, status);
}

// 错误：字符串拼接 SQL
// DO NOT: sql = "SELECT * FROM tasks WHERE status = '" + status + "'";
```

#### 2.3 命令注入防护

```java
// 错误：直接执行用户输入
// DO NOT: Runtime.getRuntime().exec("grep " + userInput);

// 正确：使用 API 而非命令行
public List<String> searchLogs(String pattern) {
    // 使用专门的日志查询 API
    return logQueryApi.search(pattern, LogLevel.INFO);
}

// 如果必须执行命令，使用白名单
public void cleanupTempFiles(String directory) {
    // 白名单验证
    Path path = Paths.get(directory).normalize();
    if (!ALLOWED_CLEANUP_DIRS.contains(path.toString())) {
        throw new SecurityException("Directory not allowed: " + directory);
    }
    
    // 使用 FileVisitor 安全删除
    Files.walkFileTree(path, new SimpleFileVisitor<>() {
        @Override
        public FileVisitResult visitFile(Path file, BasicFileAttributes attrs) throws IOException {
            Files.delete(file);
            return FileVisitResult.CONTINUE;
        }
    });
}

private static final Set<String> ALLOWED_CLEANUP_DIRS = Set.of("/tmp/agentrt", "/var/agentrt/cleanup");
```

---

### 三、认证与授权

#### 3.1 认证令牌处理

```java
public class AgentAuthenticator {
    private static final Logger logger = LoggerFactory.getLogger(AgentAuthenticator.class);
    
    /**
     * 验证认证令牌
     *
     * @param token 用户提供的令牌
     * @return 认证结果
     */
    public AuthenticationResult authenticate(String token) {
        if (token == null || token.isBlank()) {
            logger.warn("Empty authentication token received");
            return AuthenticationResult.failure("Token is required");
        }
        
        try {
            // 验证 JWT 格式和签名
            JwtParser parser = JwtParserBuilder.builder()
                .setSigningKey(loadSigningKey())
                .build();
            
            Jwt jwt = parser.parseClaimsJws(token);
            Claims claims = jwt.getBody();
            
            // 验证有效期
            Date expiration = claims.getExpiration();
            if (expiration == null || expiration.before(new Date())) {
                logger.warn("Expired token for subject: {}", claims.getSubject());
                return AuthenticationResult.failure("Token expired");
            }
            
            // 验证颁发者
            String issuer = claims.getIssuer();
            if (!EXPECTED_ISSUER.equals(issuer)) {
                logger.warn("Invalid token issuer: {}", issuer);
                return AuthenticationResult.failure("Invalid issuer");
            }
            
            return AuthenticationResult.success(claims.getSubject(), extractRoles(claims));
            
        } catch (JwtException e) {
            logger.warn("Invalid JWT token: {}", e.getMessage());
            return AuthenticationResult.failure("Invalid token");
        }
    }
    
    private Set<String> extractRoles(Claims claims) {
        @SuppressWarnings("unchecked")
        List<String> roles = claims.get("roles", List.class);
        return roles != null ? new HashSet<>(roles) : Collections.emptySet();
    }
    
    private Key loadSigningKey() {
        // 从安全存储加载密钥
        return KeyRepository.getInstance().getSigningKey();
    }
}
```

#### 3.2 权限检查

```java
public class TaskExecutor {
    private final PermissionChecker permissionChecker;
    
    /**
     * 执行任务
     *
     * @param taskId 任务 ID
     * @param userContext 用户上下文
     * @throws SecurityException 当权限不足时
     */
    public void executeTask(String taskId, UserContext userContext) {
        // 1. 验证任务存在
        Task task = taskRepository.findById(taskId);
        if (task == null) {
            throw new IllegalArgumentException("Task not found: " + taskId);
        }
        
        // 2. 权限检查 - 遵循最小权限原则
        if (!permissionChecker.hasPermission(userContext, "task:execute", task.getOwner())) {
            logger.warn("Permission denied for user {} to execute task {}",
                userContext.getUserId(), taskId);
            throw new SecurityException("Permission denied: task:execute");
        }
        
        // 3. 资源配额检查
        if (!resourceQuotaManager.checkQuota(userContext.getUserId(), ResourceType.CPU, task.getRequiredCpu())) {
            throw new ResourceQuotaExceededException("CPU quota exceeded");
        }
        
        // 4. 执行任务...
    }
}
```

---

### 四、加密处理

#### 4.1 密钥管理

```java
public class KeyManager {
    private static final String AIRY_KEY_ALGORITHM = "AES/GCM/NoPadding";
    private static final int GCM_IV_LENGTH = 12;
    private static final int GCM_TAG_LENGTH = 128;
    
    /**
     * 生成数据加密密钥
     *
     * @param purpose 密钥用途
     * @return DEK（数据加密密钥）
     */
    public SecretKey generateDataKey(String purpose) {
        KeyGenerator keyGen;
        try {
            keyGen = KeyGenerator.getInstance("AES");
            keyGen.init(256);
            return keyGen.generateKey();
        } catch (NoSuchAlgorithmException e) {
            throw new CryptoException("AES not supported", e);
        }
    }
    
    /**
     * 使用 KEK 加密 DEK
     */
    public byte[] wrapKey(SecretKey dek, String userId) {
        SecretKey kek = loadUserKEK(userId);
        Cipher cipher;
        try {
            cipher = Cipher.getInstance(AIRY_KEY_ALGORITHM);
            cipher.init(Cipher.WRAP_MODE, kek);
            return cipher.wrap(dek);
        } catch (NoSuchAlgorithmException | NoSuchPaddingException | InvalidKeyException e) {
            throw new CryptoException("Failed to wrap key", e);
        }
    }
    
    /**
     * 安全存储加密密钥
     *
     * @param keyId 密钥标识
     * @param encryptedKey 加密后的密钥
     */
    public void storeEncryptedKey(String keyId, byte[] encryptedKey) {
        // 使用安全的密钥存储服务
        keyStorage.store(keyId, encryptedKey, 
            new KeyAttributes()
                .setOwner(currentUserId())
                .setCreatedAt(Instant.now())
                .setAlgorithm("AES-256-GCM")
        );
    }
}
```

#### 4.2 数据加密解密

```java
public class DataEncryptor {
    private final KeyManager keyManager;
    
    /**
     * 加密数据
     *
     * @param plaintext 明文数据
     * @param keyId 密钥 ID
     * @return 加密后的数据（包含 IV 和密文）
     */
    public EncryptedData encrypt(byte[] plaintext, String keyId) {
        // 1. 生成随机 IV
        byte[] iv = new byte[GCM_IV_LENGTH];
        SecureRandom random = new SecureRandom();
        random.nextBytes(iv);
        
        // 2. 加载密钥
        SecretKey key = keyManager.loadKey(keyId);
        
        // 3. 加密
        Cipher cipher;
        try {
            cipher = Cipher.getInstance(AIRY_KEY_ALGORITHM);
            GCMParameterSpec gcmSpec = new GCMParameterSpec(GCM_TAG_LENGTH, iv);
            cipher.init(Cipher.ENCRYPT_MODE, key, gcmSpec);
            byte[] ciphertext = cipher.doFinal(plaintext);
            return new EncryptedData(iv, ciphertext);
        } catch (NoSuchAlgorithmException | NoSuchPaddingException | 
                 InvalidKeyException | InvalidAlgorithmParameterException | 
                 IllegalBlockSizeException | BadPaddingException e) {
            throw new CryptoException("Encryption failed", e);
        }
    }
    
    /**
     * 解密数据
     */
    public byte[] decrypt(EncryptedData encrypted, String keyId) {
        SecretKey key = keyManager.loadKey(keyId);
        
        Cipher cipher;
        try {
            cipher = Cipher.getInstance(AIRY_KEY_ALGORITHM);
            GCMParameterSpec gcmSpec = new GCMParameterSpec(GCM_TAG_LENGTH, encrypted.getIv());
            cipher.init(Cipher.DECRYPT_MODE, key, gcmSpec);
            return cipher.doFinal(encrypted.getCiphertext());
        } catch (Exception e) {
            throw new CryptoException("Decryption failed", e);
        }
    }
}

public record EncryptedData(byte[] iv, byte[] ciphertext) {
    /**
     * 序列化格式：IV长度(4字节) + IV + 密文
     */
    public byte[] toBytes() {
        ByteBuffer buffer = ByteBuffer.allocate(4 + iv.length + ciphertext.length);
        buffer.putInt(iv.length);
        buffer.put(iv);
        buffer.put(ciphertext);
        return buffer.array();
    }
    
    public static EncryptedData fromBytes(byte[] data) {
        ByteBuffer buffer = ByteBuffer.wrap(data);
        int ivLength = buffer.getInt();
        byte[] iv = new byte[ivLength];
        byte[] ciphertext = new byte[buffer.remaining()];
        buffer.get(iv);
        buffer.get(ciphertext);
        return new EncryptedData(iv, ciphertext);
    }
}
```

---

### 五、安全日志

#### 5.1 日志脱敏

```java
public class SafeLogger {
    private static final Pattern SENSITIVE_PATTERN = 
        Pattern.compile("(password|token|secret|key|credential)\\s*[:=]\\s*\\S+", 
                       Pattern.CASE_INSENSITIVE);
    
    /**
     * 记录安全相关事件
     */
    public void logSecurityEvent(SecurityEvent event) {
        // 1. 脱敏敏感字段
        String sanitizedMessage = sanitize(event.getMessage());
        
        // 2. 记录结构化日志
        log.info("SecurityEvent: type={}, userId={}, resource={}, result={}, timestamp={}",
            event.getType(),
            event.getUserId(),
            event.getResource(),
            event.getResult(),
            event.getTimestamp());
    }
    
    /**
     * 脱敏敏感信息
     */
    private String sanitize(String message) {
        if (message == null) return null;
        return SENSITIVE_PATTERN.matcher(message).replaceAll("$1=[REDACTED]");
    }
    
    /**
     * 不记录的内容
     */
    public void logTaskSubmit(TaskSubmitRequest request) {
        // DO NOT: log.info("Submitting task: {}", request); // 可能包含敏感数据
        
        // 正确：只记录必要信息
        log.info("Task submitted: id={}, type={}, priority={}",
            request.getTaskId(),
            request.getType(),
            request.getPriority());
    }
}

public record SecurityEvent(
    SecurityEventType type,
    String userId,
    String resource,
    SecurityResult result,
    Instant timestamp,
    String message
) {}

public enum SecurityEventType {
    AUTH_SUCCESS,
    AUTH_FAILURE,
    PERMISSION_DENIED,
    RESOURCE_ACCESS,
    KEY_OPERATION,
    CONFIG_CHANGE
}
```

#### 5.2 审计日志

```java
public class AuditLogger {
    private final AsyncAuditQueue auditQueue;
    
    /**
     * 记录审计日志
     */
    public void audit(AuditRecord record) {
        // 1. 验证审计记录完整性
        if (!record.isValid()) {
            throw new IllegalArgumentException("Invalid audit record");
        }
        
        // 2. 异步写入，不阻塞主流程
        auditQueue.enqueue(record);
    }
    
    /**
     * 记录关键操作
     */
    public void auditKeyOperation(String operator, KeyOperation op, String keyId, boolean success) {
        audit(new AuditRecord.Builder()
            .operator(operator)
            .operation(op.name())
            .resource("key:" + keyId)
            .result(success ? "success" : "failure")
            .timestamp(Instant.now())
            .build());
    }
}

public class AuditRecord {
    private final String operator;
    private final String operation;
    private final String resource;
    private final String result;
    private final Instant timestamp;
    private final Map<String, String> metadata;
    
    public static class Builder {
        private String operator;
        private String operation;
        private String resource;
        private String result;
        private Instant timestamp = Instant.now();
        private Map<String, String> metadata = new HashMap<>();
        
        public Builder operator(String operator) {
            this.operator = Objects.requireNonNull(operator);
            return this;
        }
        
        public Builder operation(String operation) {
            this.operation = Objects.requireNonNull(operation);
            return this;
        }
        
        public Builder resource(String resource) {
            this.resource = Objects.requireNonNull(resource);
            return this;
        }
        
        public Builder result(String result) {
            this.result = Objects.requireNonNull(result);
            return this;
        }
        
        public Builder timestamp(Instant timestamp) {
            this.timestamp = Objects.requireNonNull(timestamp);
            return this;
        }
        
        public AuditRecord build() {
            return new AuditRecord(operator, operation, resource, result, timestamp, metadata);
        }
    }
    
    public boolean isValid() {
        return operator != null && !operator.isBlank()
            && operation != null && !operation.isBlank()
            && timestamp != null;
    }
}
```

---

### 六、错误处理

#### 6.1 异常安全

```java
public class SafeTaskProcessor {
    private final ResourceCleanup cleanup;
    
    /**
     * 处理任务
     */
    public ProcessingResult processTask(Task task) {
        ResourceHandle handle = null;
        try {
            // 1. 资源获取
            handle = resourceManager.acquire(task.getRequiredResources());
            
            // 2. 处理任务
            ProcessingResult result = doProcess(task, handle);
            
            // 3. 成功时不额外记录，异常会在 finally 中统一处理
            return result;
            
        } catch (ResourceException e) {
            // 资源相关异常
            logger.warn("Resource error processing task {}: {}", task.getId(), e.getMessage());
            return ProcessingResult.resourceError(e.getMessage());
            
        } catch (ProcessingException e) {
            // 处理异常
            logger.error("Processing failed for task {}: {}", task.getId(), e.getMessage());
            return ProcessingResult.failure(e.getMessage());
            
        } finally {
            // 4. 无论如何都要释放资源
            if (handle != null) {
                try {
                    handle.release();
                } catch (ReleaseException e) {
                    // 记录释放异常，但不抛出，避免掩盖原始异常
                    logger.error("Failed to release resource for task {}: {}", 
                        task.getId(), e.getMessage());
                }
            }
        }
    }
}
```

#### 6.2 信息泄露防护

```java
public class ErrorHandler {
    private static final Logger logger = LoggerFactory.getLogger(ErrorHandler.class);
    
    /**
     * 处理错误
     */
    public ErrorResponse handleError(String operation, Exception e) {
        // 1. 记录完整错误信息（内部）
        logger.error("Operation {} failed", operation, e);
        
        // 2. 返回脱敏的错误信息（外部）
        if (e instanceof SecurityException) {
            return ErrorResponse.of("security_error", "Security violation detected");
        }
        
        if (e instanceof ResourceQuotaException) {
            return ErrorResponse.of("quota_exceeded", "Resource quota exceeded");
        }
        
        if (e instanceof ValidationException) {
            return ErrorResponse.of("validation_error", "Input validation failed");
        }
        
        // 未知错误，返回通用消息
        return ErrorResponse.of("internal_error", "An internal error occurred");
    }
}

public record ErrorResponse(String code, String message) {
    // 禁止在错误响应中包含堆栈跟踪、SQL 错误、内部路径等
    
    public static ErrorResponse of(String code, String message) {
        return new ErrorResponse(code, message);
    }
    
    /**
     * 转换为 API 响应
     */
    public Map<String, Object> toApiResponse() {
        return Map.of(
            "success", false,
            "error", Map.of(
                "code", code,
                "message", message
            )
        );
    }
}
```

---

### 七、并发安全

#### 7.1 线程安全集合

```java
public class TaskRegistry {
    // 使用线程安全的 ConcurrentHashMap
    private final ConcurrentHashMap<String, TaskState> tasks = new ConcurrentHashMap<>();
    
    /**
     * 注册任务
     */
    public void registerTask(String taskId, TaskState state) {
        tasks.put(taskId, state);
    }
    
    /**
     * 获取任务状态
     */
    public Optional<TaskState> getTaskState(String taskId) {
        return Optional.ofNullable(tasks.get(taskId));
    }
    
    /**
     * 安全更新任务状态
     */
    public boolean updateTaskState(String taskId, Function<TaskState, TaskState> updater) {
        return tasks.computeIfPresent(taskId, (id, current) -> updater.apply(current)) != null;
    }
    
    /**
     * 原子操作
     */
    public TaskState computeIfAbsent(String taskId, Supplier<TaskState> factory) {
        return tasks.computeIfAbsent(taskId, k -> {
            TaskState state = factory.get();
            logger.debug("Created new task state for: {}", taskId);
            return state;
        });
    }
}
```

#### 7.2 同步控制

```java
public class SharedResourceManager {
    private final Map<String, ReadWriteLock> resourceLocks = new ConcurrentHashMap<>();
    private final Map<String, Resource> resources = new ConcurrentHashMap<>();
    
    /**
     * 读取资源（读锁）
     */
    public <T> T readResource(String resourceId, Function<Resource, T> reader) {
        ReadWriteLock lock = getOrCreateLock(resourceId);
        lock.readLock().lock();
        try {
            Resource resource = resources.get(resourceId);
            if (resource == null) {
                throw new ResourceNotFoundException(resourceId);
            }
            return reader.apply(resource);
        } finally {
            lock.readLock().unlock();
        }
    }
    
    /**
     * 修改资源（写锁）
     */
    public void writeResource(String resourceId, Consumer<Resource> writer) {
        ReadWriteLock lock = getOrCreateLock(resourceId);
        lock.writeLock().lock();
        try {
            Resource resource = resources.get(resourceId);
            if (resource == null) {
                throw new ResourceNotFoundException(resourceId);
            }
            writer.accept(resource);
        } finally {
            lock.writeLock().unlock();
        }
    }
    
    private ReadWriteLock getOrCreateLock(String resourceId) {
        return resourceLocks.computeIfAbsent(resourceId, k -> new ReentrantReadWriteLock());
    }
}
```

---

### 八、依赖安全

#### 8.1 依赖管理

```java
// pom.xml - 使用 BOM 管理依赖版本
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.agentrt</groupId>
            <artifactId>agentrt-bom</artifactId>
            <version>${agentrt.version}</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>

// 禁止使用已知漏洞的依赖
// 检查：mvn org.owasp:dependency-check-maven:check
```

#### 8.2 安全配置

```java
public class SecurityConfig {
    /**
     * 创建 HttpSecurity 配置
     */
    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        return http
            // 禁用 iframe
            .headers(headers -> headers.frameOptions(frame -> frame.deny()))
            
            // CSRF 防护
            .csrf(csrf -> csrf
                .csrfTokenRepository(CookieCsrfTokenRepository.withHttpOnlyFalse())
                .ignoringAntMatchers("/api/public/**"))
            
            // 会话管理
            .sessionManagement(session -> session
                .sessionCreationPolicy(SessionCreationPolicy.STATELESS))
            
            // 认证规则
            .authorizeHttpRequests(auth -> auth
                .antMatchers("/api/public/**").permitAll()
                .antMatchers("/api/admin/**").hasRole("ADMIN")
                .anyRequest().authenticated())
            
            // 异常处理
            .exceptionHandling(ex -> ex
                .authenticationEntryPoint((request, response, authException) -> {
                    response.setStatus(401);
                    response.setContentType("application/json");
                    response.getWriter().write("{\"error\":\"unauthorized\"}");
                })
                .accessDeniedHandler((request, response, accessDeniedException) -> {
                    response.setStatus(403);
                    response.setContentType("application/json");
                    response.getWriter().write("{\"error\":\"forbidden\"}");
                }))
            
            .build();
    }
}
```

---
### 十、Airymax 模块 Java 安全编码示例

#### 10.1 cupolas（安全穹顶）Java 安全实现
cupolas模块实现Airymax的安全穹顶，Java SDK需要严格遵循其安全模型：

##### 10.1.1 虚拟工位（Virtual Workbench）沙箱
```java
/**
 * 虚拟工位沙箱 - 体现D-2（安全隔离）和S-1（垂直分层）原则
 * 
 * 基于ClassLoader隔离机制，为每个任务创建独立的执行环境。
 * 集成资源限制和权限控制，防止任务间相互影响。
 * 
 * @see agentrt/cupolas/workbench/ 中的进程隔离原语
 */
public class VirtualWorkbenchSandbox {
    private final SandboxClassLoader classLoader;
    private final ResourceLimiter limiter;
    private final SecurityManager securityManager;
    
    /**
     * 在沙箱中执行任务 - 体现D-1（最小权限）原则
     * 
     * @param task 任务定义
     * @param context 安全上下文
     * @return 执行结果
     */
    public ExecutionResult executeInSandbox(Task task, SecurityContext context) {
        // 1. 创建隔离的ClassLoader
        classLoader = new SandboxClassLoader(
            task.getRequiredLibraries(),
            getSystemClassLoader()
        );
        
        // 2. 设置资源限制
        limiter = new ResourceLimiter(
            task.getCpuQuota(),
            task.getMemoryQuota(),
            task.getDiskQuota()
        );
        
        // 3. 配置安全管理器
        securityManager = new SandboxSecurityManager(
            context.getPermissions(),
            new AuditLogger()
        );
        
        // 4. 在受限环境中执行
        return doExecuteWithRestrictions(task);
    }
    
    private ExecutionResult doExecuteWithRestrictions(Task task) {
        Thread currentThread = Thread.currentThread();
        ClassLoader originalLoader = currentThread.getContextClassLoader();
        SecurityManager originalManager = System.getSecurityManager();
        
        try {
            // 切换ClassLoader
            currentThread.setContextClassLoader(classLoader);
            System.setSecurityManager(securityManager);
            
            // 应用资源限制
            limiter.apply();
            
            // 执行任务
            return task.execute();
            
        } catch (SecurityException e) {
            logAudit("Sandbox security violation: {}", e.getMessage());
            return ExecutionResult.failure("Security violation: " + e.getMessage());
            
        } finally {
            // 恢复原始环境
            currentThread.setContextClassLoader(originalLoader);
            System.setSecurityManager(originalManager);
            limiter.release();
        }
    }
}
```

##### 10.1.2 权限裁决（Permission Arbiter）
```java
/**
 * 权限裁决引擎 - 体现D-1（最小权限）和D-4（安全审计）原则
 * 
 * 基于YAML规则引擎实现细粒度访问控制。
 * 支持动态策略更新和实时审计。
 * 
 * @see agentrt/cupolas/permission/ 中的规则引擎
 */
@Service
public class PermissionArbiter {
    private final RuleEngine ruleEngine;
    private final AuditTrail auditTrail;
    
    /**
     * 裁决访问请求 - 体现防御深度（D-3）原则
     * 
     * 多层验证：
     * 1. 主体身份验证
     * 2. 资源权限检查
     * 3. 上下文策略验证
     * 4. 风险评估
     */
    public AccessDecision decide(AccessRequest request) {
        // 层次1：主体验证
        if (!validateSubject(request.getSubject())) {
            logAudit("Invalid subject: {}", request.getSubject());
            return AccessDecision.deny("Invalid subject");
        }
        
        // 层次2：资源权限检查
        Permission required = request.getRequiredPermission();
        if (!hasPermission(request.getSubject(), required)) {
            logAudit("Permission denied: subject={}, resource={}, permission={}",
                request.getSubject(), request.getResource(), required);
            return AccessDecision.deny("Insufficient permission");
        }
        
        // 层次3：上下文策略验证
        Context context = request.getContext();
        if (!evaluateContextPolicy(context)) {
            logAudit("Context policy violation: {}", context);
            return AccessDecision.deny("Context policy violation");
        }
        
        // 层次4：风险评估
        RiskAssessment risk = assessRisk(request);
        if (risk.getLevel() == RiskLevel.HIGH) {
            logAudit("High risk access denied: risk={}", risk.getScore());
            return AccessDecision.deny("High risk access");
        }
        
        // 授予访问权限
        logAudit("Access granted: subject={}, resource={}", 
            request.getSubject(), request.getResource());
        return AccessDecision.grant(risk.getScore());
    }
}
```

#### 10.2 daemon（守护层）Java 安全实现
Backs模块作为系统服务，需实现可靠的安全通信和资源管理：

##### 10.2.1 安全IPC客户端
```java
/**
 * 安全IPC客户端 - 体现E-3（通信基础设施）和D-3（纵深防御）原则
 * 
 * 实现与Atoms模块的安全进程间通信。
 * 集成消息加密、身份验证和完整性保护。
 * 
 * @see ipc.md 中的安全通信协议
 */
public class SecureIpcClient {
    private final CryptoService crypto;
    private final AuthService auth;
    private final IpcChannel channel;
    
    /**
     * 发送安全消息 - 体现防御深度原则
     */
    public SecureResponse sendSecure(SecureRequest request) {
        // 层次1：请求签名
        byte[] signature = crypto.sign(request.getPayload());
        request.setSignature(signature);
        
        // 层次2：消息加密
        EncryptedMessage encrypted = crypto.encrypt(
            request.toBytes(),
            getSessionKey()
        );
        
        // 层次3：身份验证
        AuthToken token = auth.getCurrentToken();
        if (!auth.isValid(token)) {
            throw new SecurityException("Authentication token expired");
        }
        
        // 层次4：发送并验证响应
        RawResponse raw = channel.send(encrypted);
        SecureResponse response = parseResponse(raw);
        
        // 验证响应完整性
        if (!crypto.verify(response.getSignature(), response.getPayload())) {
            throw new SecurityException("Response integrity check failed");
        }
        
        return response;
    }
}
```

#### 10.3 commons（公共库层）Java 安全实现
Common模块提供跨层安全基础设施：

##### 10.3.1 统一密钥管理服务
```java
/**
 * 统一密钥管理服务 - 体现E-1（基础设施）和D-4（安全审计）原则
 * 
 * 提供密钥生成、存储、轮换和审计的完整生命周期管理。
 * 支持多租户隔离和硬件安全模块（HSM）集成。
 */
@Service
public class UnifiedKeyManager {
    private final KeyStorage keyStorage;
    private final KeyGenerator keyGenerator;
    private final AuditLogger auditLogger;
    
    /**
     * 生成并存储密钥 - 体现密钥生命周期管理
     */
    public KeyMetadata generateKey(KeySpec spec, String owner) {
        // 1. 生成密钥
        SecretKey key = keyGenerator.generate(spec);
        
        // 2. 加密存储
        byte[] encrypted = encryptKey(key, getMasterKey());
        String keyId = UUID.randomUUID().toString();
        
        // 3. 安全存储
        keyStorage.store(keyId, encrypted, new StorageOptions()
            .setEncryptionAlgorithm("AES-256-GCM")
            .setOwner(owner)
            .setCreatedAt(Instant.now())
        );
        
        // 4. 审计日志
        auditLogger.logKeyOperation(owner, KeyOperation.GENERATE, keyId, true);
        
        return new KeyMetadata(keyId, spec, owner);
    }
    
    /**
     * 密钥轮换策略 - 体现E-2（可维护性）原则
     */
    @Scheduled(fixedRate = 30 * 24 * 60 * 60 * 1000) // 每月轮换
    public void rotateExpiringKeys() {
        List<KeyMetadata> expiring = keyStorage.findKeysExpiringSoon();
        for (KeyMetadata metadata : expiring) {
            try {
                // 生成新密钥
                KeyMetadata newKey = generateKey(metadata.getSpec(), metadata.getOwner());
                
                // 迁移数据
                migrateData(metadata.getKeyId(), newKey.getKeyId());
                
                // 标记旧密钥为已轮换
                keyStorage.markAsRotated(metadata.getKeyId(), newKey.getKeyId());
                
                auditLogger.logKeyOperation("system", KeyOperation.ROTATE, 
                    metadata.getKeyId(), true);
                
            } catch (Exception e) {
                auditLogger.logKeyOperation("system", KeyOperation.ROTATE, 
                    metadata.getKeyId(), false);
                logger.error("Failed to rotate key: {}", metadata.getKeyId(), e);
            }
        }
    }
}
```

#### 10.4 Atoms（原子层）Java Native Interface (JNI) 安全
Atoms模块通过JNI提供原生接口，需要特殊的安全考虑：

##### 10.4.1 安全JNI绑定
```java
/**
 * 安全JNI绑定 - 体现K-1（最小特权）和D-2（安全隔离）原则
 * 
 * 封装与Atoms微核心的本地调用，实现严格参数验证和边界检查。
 * 防止缓冲区溢出和类型混淆攻击。
 */
public class SecureKernelBinding {
    static {
        // 加载本地库
        System.loadLibrary("airy_kernel");
    }
    
    // 本地方法声明
    private native long nativeAllocateMemory(long size, int numaNode);
    private native int nativeScheduleTask(long taskHandle, int priority);
    
    /**
     * 安全内存分配 - 体现M-3（拓扑优化）和防御深度原则
     */
    public MemoryHandle allocateMemory(long size, int numaNode) {
        // 层次1：参数验证
        if (size <= 0 || size > MAX_ALLOC_SIZE) {
            throw new IllegalArgumentException("Invalid allocation size: " + size);
        }
        
        // 层次2：NUMA节点验证
        if (numaNode < -1 || numaNode >= getNumaNodeCount()) {
            throw new IllegalArgumentException("Invalid NUMA node: " + numaNode);
        }
        
        // 层次3：权限检查
        if (!checkMemoryPermission(size)) {
            throw new SecurityException("Insufficient memory permission");
        }
        
        // 调用本地方法
        long nativeHandle = nativeAllocateMemory(size, numaNode);
        if (nativeHandle == 0) {
            throw new RuntimeException("Memory allocation failed");
        }
        
        return new MemoryHandle(nativeHandle, size);
    }
    
    /**
     * 清理本地资源 - 体现资源确定性原则
     */
    @Override
    protected void finalize() throws Throwable {
        try {
            releaseNativeResources();
        } finally {
            super.finalize();
        }
    }
}
```

---
### 十一、参考文献

1. **Airymax 架构设计原则**: [00-architectural-principles.md](../../10-architecture/00-architectural-principles.md)
2. **OWASP Top 10**: https://owasp.org/www-project-top-ten/
3. **Java Secure Coding Guidelines**: https://wiki.sei.cmu.edu/confluence/display/java
4. **Spring Security Documentation**: https://docs.spring.io/spring-security/site/docs/current/reference/html5/
5. **Airymax 核心架构文档**:
   - [coreloopthree.md](../../10-architecture/02-coreloopthree.md)
   - [memoryrovol.md](../../10-architecture/03-memoryrovol.md)
   - [microcorert.md](../../10-architecture/04-microcorert.md)
   - [ipc.md](../../10-architecture/30-kernel/01-ipc.md)
   - [syscall.md](../../10-architecture/05-syscall.md)
   - [logging_system.md](../../10-architecture/50-services/02-logging.md)
   - [Security_design_standard.md](../Security_design_standard.md)

---

© 2025-2026 SPHARX Ltd. All Rights Reserved.
