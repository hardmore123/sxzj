## 目录



***

## 概述

熔断器（Circuit Breaker）是一种用于防止系统雪崩的保护机制。当下游服务出现故障或响应缓慢时，熔断器会自动切断请求，快速失败，避免故障扩散。本文档详细介绍熔断器的设计原理和企业级实现。

***

## 2. 核心概念

### 2.1 熔断器状态机

```plaintext
┌─────────────────────────────────────────────────────────────────┐
│                    熔断器三态状态机                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│                                                                  │
│      ┌──────────────────────────────────────────────┐          │
│      │                                               │          │
│      │              CLOSED（闭合）                   │          │
│      │                                               │          │
│      │  - 正常放行所有请求                          │          │
│      │  - 统计失败率和响应时间                      │          │
│      │  - 失败率超过阈值 → OPEN                     │          │
│      │                                               │          │
│      └──────────────┬───────────────────────────────┘          │
│                     │                                           │
│                     │ 失败率 > 阈值                             │
│                     │ (如 50% 错误率)                           │
│                     │                                           │
│                     ▼                                           │
│      ┌──────────────────────────────────────────────┐          │
│      │                                               │          │
│      │               OPEN（断开）                    │          │
│      │                                               │          │
│      │  - 快速失败，直接拒绝请求                    │          │
│      │  - 不调用下游服务                            │          │
│      │  - 等待冷却期（如30s）                       │          │
│      │  - 冷却期后 → HALF_OPEN                      │          │
│      │                                               │          │
│      └──────────────┬───────────────────────────────┘          │
│                     │                                           │
│                     │ 等待冷却期                                │
│                     │ (如 30 秒)                                │
│                     │                                           │
│                     ▼                                           │
│      ┌──────────────────────────────────────────────┐          │
│      │                                               │          │
│      │            HALF_OPEN（半开）                  │          │
│      │                                               │          │
│      │  - 允许少量探测请求通过                      │          │
│      │  - 如果探测成功 → CLOSED                     │          │
│      │  - 如果探测失败 → OPEN                       │          │
│      │                                               │          │
│      └───────────────────────┬──────────────────────┘          │
│              │                │                                 │
│    探测失败  │                │  探测成功                       │
│              │                │                                 │
│              └────────────────┘                                 │
│                   回到对应状态                                  │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 2.2 熔断决策流程

```plaintext
┌─────────────────────────────────────────────────────────────────┐
│                      熔断决策流程                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│                    ┌─────────────┐                              │
│                    │   请求到来  │                              │
│                    └──────┬──────┘                              │
│                           │                                     │
│                           ▼                                     │
│                    ┌─────────────┐                              │
│                    │ 检查状态    │                              │
│                    └──────┬──────┘                              │
│                           │                                     │
│         ┌─────────────────┼─────────────────┐                  │
│         │                 │                 │                  │
│         ▼                 ▼                 ▼                  │
│   ┌─────────┐      ┌──────────┐      ┌──────────┐            │
│   │ CLOSED  │      │   OPEN   │      │ HALF_OPEN│            │
│   └────┬────┘      └────┬─────┘      └────┬─────┘            │
│        │                │                  │                   │
│        │                │                  │                   │
│        ▼                ▼                  ▼                   │
│   ┌─────────┐      ┌──────────┐      ┌──────────┐            │
│   │ 执行请求│      │ 快速失败 │      │ 探测请求 │            │
│   └────┬────┘      └────┬─────┘      └────┬─────┘            │
│        │                │                  │                   │
│        ▼                │                  ▼                   │
│   ┌─────────┐           │            ┌──────────┐             │
│   │ 记录结果│           │            │ 记录结果 │             │
│   └────┬────┘           │            └────┬─────┘             │
│        │                │                  │                   │
│        ▼                │           ┌──────┴──────┐            │
│   ┌─────────┐           │           │             │            │
│   │失败率？ │           │      探测成功？    探测失败？         │
│   └────┬────┘           │           │             │            │
│        │                │           │             │            │
│   ┌────┴────┐           │           │             │            │
│   │         │           │           │             │            │
│  高?       低?          │           │             │            │
│   │         │           │           │             │            │
│   ▼         ▼           ▼           ▼             ▼            │
│ OPEN     CLOSED    返回错误      CLOSED        OPEN           │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

***

## 3. 核心实现

### 3.1 熔断器基础实现

```python
# app/core/circuit_breaker.py

import time
import threading
from enum import Enum
from typing import Callable, Optional, TypeVar, Any
from dataclasses import dataclass, field
from datetime import datetime, timedelta
import logging

logger = logging.getLogger(__name__)

T = TypeVar('T')


class CircuitState(Enum):
    """熔断器状态"""
    CLOSED = "closed"           # 闭合，正常工作
    OPEN = "open"               # 断开，拒绝请求
    HALF_OPEN = "half_open"     # 半开，探测恢复


@dataclass
class CircuitBreakerConfig:
    """熔断器配置"""

    # 失败阈值配置
    failure_threshold: int = 5              # 失败次数阈值
    failure_rate_threshold: float = 0.5     # 失败率阈值（50%）
    min_calls: int = 10                     # 最小调用次数（用于计算失败率）

    # 时间窗口配置
    window_size: int = 60                   # 统计窗口大小（秒）
    open_timeout: int = 30                  # OPEN状态持续时间（秒）

    # 半开状态配置
    half_open_max_calls: int = 3            # 半开状态最大探测次数

    # 超时配置
    timeout: Optional[float] = None         # 请求超时时间（秒）

    # 异常配置
    excluded_exceptions: tuple = ()         # 不计入失败的异常


@dataclass
class CallRecord:
    """调用记录"""
    timestamp: float
    success: bool
    duration: float
    exception: Optional[Exception] = None


class CircuitBreaker:
    """熔断器"""

    def __init__(self, config: CircuitBreakerConfig = None):
        self.config = config or CircuitBreakerConfig()
        self._state = CircuitState.CLOSED
        self._state_lock = threading.RLock()

        # 调用记录（滑动窗口）
        self._calls: list[CallRecord] = []
        self._calls_lock = threading.RLock()

        # 状态切换时间
        self._last_state_change = time.time()

        # 半开状态探测计数
        self._half_open_calls = 0

    @property
    def state(self) -> CircuitState:
        """获取当前状态"""
        with self._state_lock:
            self._update_state()
            return self._state

    def _update_state(self):
        """更新状态（从OPEN到HALF_OPEN）"""
        if self._state == CircuitState.OPEN:
            elapsed = time.time() - self._last_state_change
            if elapsed >= self.config.open_timeout:
                self._change_state(CircuitState.HALF_OPEN)

    def _change_state(self, new_state: CircuitState):
        """改变状态"""
        old_state = self._state
        self._state = new_state
        self._last_state_change = time.time()

        if new_state == CircuitState.HALF_OPEN:
            self._half_open_calls = 0

        logger.info(
            f"Circuit breaker state changed: {old_state.value} -> {new_state.value}"
        )

    def _clean_old_calls(self):
        """清理过期的调用记录"""
        current_time = time.time()
        window_start = current_time - self.config.window_size

        with self._calls_lock:
            self._calls = [
                call for call in self._calls
                if call.timestamp >= window_start
            ]

    def _record_call(self, success: bool, duration: float, exception: Optional[Exception] = None):
        """记录调用结果"""
        record = CallRecord(
            timestamp=time.time(),
            success=success,
            duration=duration,
            exception=exception,
        )

        with self._calls_lock:
            self._calls.append(record)
            self._clean_old_calls()

    def _get_failure_rate(self) -> float:
        """计算失败率"""
        with self._calls_lock:
            self._clean_old_calls()

            if len(self._calls) < self.config.min_calls:
                return 0.0

            failures = sum(1 for call in self._calls if not call.success)
            return failures / len(self._calls)

    def _get_failure_count(self) -> int:
        """获取失败次数"""
        with self._calls_lock:
            self._clean_old_calls()
            return sum(1 for call in self._calls if not call.success)

    def _should_open(self) -> bool:
        """判断是否应该打开熔断器"""
        failure_rate = self._get_failure_rate()
        failure_count = self._get_failure_count()

        # 失败率超过阈值
        if failure_rate >= self.config.failure_rate_threshold:
            logger.warning(
                f"Failure rate {failure_rate:.2%} exceeded threshold "
                f"{self.config.failure_rate_threshold:.2%}"
            )
            return True

        # 失败次数超过阈值
        if failure_count >= self.config.failure_threshold:
            logger.warning(
                f"Failure count {failure_count} exceeded threshold "
                f"{self.config.failure_threshold}"
            )
            return True

        return False

    def _is_excluded_exception(self, exception: Exception) -> bool:
        """判断是否是排除的异常"""
        if not self.config.excluded_exceptions:
            return False
        return isinstance(exception, self.config.excluded_exceptions)

    def call(self, func: Callable[..., T], *args, **kwargs) -> T:
        """
        执行函数调用（带熔断保护）

        Args:
            func: 要执行的函数
            *args: 函数参数
            **kwargs: 函数关键字参数

        Returns:
            函数执行结果

        Raises:
            CircuitBreakerOpenError: 熔断器打开时
            TimeoutError: 超时
            其他异常: 函数执行失败
        """
        with self._state_lock:
            self._update_state()
            current_state = self._state

            # OPEN状态：快速失败
            if current_state == CircuitState.OPEN:
                raise CircuitBreakerOpenError(
                    f"Circuit breaker is OPEN. "
                    f"Will retry after {self.config.open_timeout}s"
                )

            # HALF_OPEN状态：检查探测次数
            if current_state == CircuitState.HALF_OPEN:
                if self._half_open_calls >= self.config.half_open_max_calls:
                    raise CircuitBreakerOpenError(
                        "Circuit breaker is HALF_OPEN and max probe calls reached"
                    )
                self._half_open_calls += 1

        # 执行函数
        start_time = time.time()
        success = False
        exception = None

        try:
            # 超时控制
            if self.config.timeout:
                import signal

                def timeout_handler(signum, frame):
                    raise TimeoutError(f"Call timeout after {self.config.timeout}s")

                old_handler = signal.signal(signal.SIGALRM, timeout_handler)
                signal.alarm(int(self.config.timeout))

                try:
                    result = func(*args, **kwargs)
                finally:
                    signal.alarm(0)
                    signal.signal(signal.SIGALRM, old_handler)
            else:
                result = func(*args, **kwargs)

            success = True
            return result

        except Exception as e:
            exception = e
            success = False

            # 如果是排除的异常，不计入失败
            if self._is_excluded_exception(e):
                success = True

            raise

        finally:
            duration = time.time() - start_time

            # 记录调用结果
            self._record_call(success, duration, exception)

            # 根据结果更新状态
            with self._state_lock:
                if current_state == CircuitState.CLOSED:
                    if not success and self._should_open():
                        self._change_state(CircuitState.OPEN)

                elif current_state == CircuitState.HALF_OPEN:
                    if success:
                        # 探测成功，关闭熔断器
                        logger.info("Probe succeeded, closing circuit breaker")
                        self._change_state(CircuitState.CLOSED)
                    else:
                        # 探测失败，重新打开
                        logger.warning("Probe failed, reopening circuit breaker")
                        self._change_state(CircuitState.OPEN)

    def __call__(self, func: Callable[..., T]) -> Callable[..., T]:
        """装饰器用法"""
        def wrapper(*args, **kwargs) -> T:
            return self.call(func, *args, **kwargs)
        return wrapper

    def get_metrics(self) -> dict:
        """获取熔断器指标"""
        with self._calls_lock:
            self._clean_old_calls()

            total_calls = len(self._calls)
            if total_calls == 0:
                return {
                    "state": self.state.value,
                    "total_calls": 0,
                    "failure_rate": 0.0,
                    "success_rate": 0.0,
                }

            failures = sum(1 for call in self._calls if not call.success)
            avg_duration = sum(call.duration for call in self._calls) / total_calls

            return {
                "state": self.state.value,
                "total_calls": total_calls,
                "failures": failures,
                "failure_rate": failures / total_calls,
                "success_rate": (total_calls - failures) / total_calls,
                "avg_duration": avg_duration,
                "last_state_change": datetime.fromtimestamp(
                    self._last_state_change
                ).isoformat(),
            }


class CircuitBreakerOpenError(Exception):
    """熔断器打开异常"""
    pass
```

### 3.2 装饰器用法

```python
# app/core/circuit_breaker_decorator.py

from functools import wraps
from typing import Callable, TypeVar, Optional
import logging

from .circuit_breaker import CircuitBreaker, CircuitBreakerConfig

logger = logging.getLogger(__name__)

T = TypeVar('T')


def circuit_breaker(
    failure_threshold: int = 5,
    failure_rate_threshold: float = 0.5,
    timeout: int = 30,
    excluded_exceptions: tuple = (),
):
    """
    熔断器装饰器

    使用示例:
        @circuit_breaker(failure_threshold=5, timeout=30)
        def call_external_api():
            ...
    """
    config = CircuitBreakerConfig(
        failure_threshold=failure_threshold,
        failure_rate_threshold=failure_rate_threshold,
        open_timeout=timeout,
        excluded_exceptions=excluded_exceptions,
    )

    breaker = CircuitBreaker(config)

    def decorator(func: Callable[..., T]) -> Callable[..., T]:
        @wraps(func)
        def wrapper(*args, **kwargs) -> T:
            return breaker.call(func, *args, **kwargs)

        # 添加熔断器属性，方便访问
        wrapper.circuit_breaker = breaker
        return wrapper

    return decorator


# 使用示例

@circuit_breaker(
    failure_threshold=5,
    failure_rate_threshold=0.5,
    timeout=30,
)
def call_unstable_api(url: str) -> dict:
    """调用不稳定的外部API"""
    import requests

    response = requests.get(url, timeout=10)
    response.raise_for_status()
    return response.json()


# 查看熔断器状态
def check_circuit_breaker_status():
    """检查熔断器状态"""
    if hasattr(call_unstable_api, 'circuit_breaker'):
        breaker = call_unstable_api.circuit_breaker
        metrics = breaker.get_metrics()
        print(f"Circuit Breaker Status: {metrics}")
```

### 3.3 熔断器管理器

```python
# app/core/circuit_breaker_manager.py

from typing import Dict, Optional
import logging
from threading import RLock

from .circuit_breaker import CircuitBreaker, CircuitBreakerConfig, CircuitState

logger = logging.getLogger(__name__)


class CircuitBreakerManager:
    """熔断器管理器（全局单例）"""

    _instance: Optional['CircuitBreakerManager'] = None
    _lock = RLock()

    def __new__(cls):
        if cls._instance is None:
            with cls._lock:
                if cls._instance is None:
                    cls._instance = super().__new__(cls)
        return cls._instance

    def __init__(self):
        if not hasattr(self, '_initialized'):
            self._breakers: Dict[str, CircuitBreaker] = {}
            self._configs: Dict[str, CircuitBreakerConfig] = {}
            self._breakers_lock = RLock()
            self._initialized = True

    def register(
        self,
        name: str,
        config: Optional[CircuitBreakerConfig] = None
    ) -> CircuitBreaker:
        """
        注册熔断器

        Args:
            name: 熔断器名称
            config: 熔断器配置

        Returns:
            熔断器实例
        """
        with self._breakers_lock:
            if name not in self._breakers:
                self._configs[name] = config or CircuitBreakerConfig()
                self._breakers[name] = CircuitBreaker(self._configs[name])
                logger.info(f"Registered circuit breaker: {name}")

            return self._breakers[name]

    def get(self, name: str) -> Optional[CircuitBreaker]:
        """获取熔断器"""
        with self._breakers_lock:
            return self._breakers.get(name)

    def get_or_create(
        self,
        name: str,
        config: Optional[CircuitBreakerConfig] = None
    ) -> CircuitBreaker:
        """获取或创建熔断器"""
        breaker = self.get(name)
        if breaker is None:
            breaker = self.register(name, config)
        return breaker

    def get_all_metrics(self) -> Dict[str, dict]:
        """获取所有熔断器的指标"""
        with self._breakers_lock:
            return {
                name: breaker.get_metrics()
                for name, breaker in self._breakers.items()
            }

    def get_open_breakers(self) -> list[str]:
        """获取所有打开的熔断器"""
        with self._breakers_lock:
            return [
                name
                for name, breaker in self._breakers.items()
                if breaker.state == CircuitState.OPEN
            ]

    def reset(self, name: str):
        """重置熔断器"""
        with self._breakers_lock:
            if name in self._breakers:
                config = self._configs[name]
                self._breakers[name] = CircuitBreaker(config)
                logger.info(f"Reset circuit breaker: {name}")

    def reset_all(self):
        """重置所有熔断器"""
        with self._breakers_lock:
            for name in list(self._breakers.keys()):
                self.reset(name)


# 全局实例
circuit_breaker_manager = CircuitBreakerManager()


# 使用示例

def call_service_a():
    """调用服务A"""
    breaker = circuit_breaker_manager.get_or_create(
        "service_a",
        CircuitBreakerConfig(
            failure_threshold=5,
            failure_rate_threshold=0.5,
            open_timeout=30,
        )
    )

    return breaker.call(lambda: requests.get("https://service-a.com/api"))


def call_service_b():
    """调用服务B"""
    breaker = circuit_breaker_manager.get_or_create(
        "service_b",
        CircuitBreakerConfig(
            failure_threshold=3,
            failure_rate_threshold=0.3,
            open_timeout=60,
        )
    )

    return breaker.call(lambda: requests.get("https://service-b.com/api"))


def monitor_all_breakers():
    """监控所有熔断器"""
    metrics = circuit_breaker_manager.get_all_metrics()

    for name, metric in metrics.items():
        print(f"\n{name}:")
        print(f"  State: {metric['state']}")
        print(f"  Total Calls: {metric['total_calls']}")
        print(f"  Failure Rate: {metric.get('failure_rate', 0):.2%}")

    # 检查打开的熔断器
    open_breakers = circuit_breaker_manager.get_open_breakers()
    if open_breakers:
        logger.warning(f"Open circuit breakers: {open_breakers}")
```

### 3.4 与Retry组合使用

```python
# app/core/retry_with_circuit_breaker.py

import logging
from typing import Callable, TypeVar, Optional
from functools import wraps

from .retry_decorator import retry, RetryConfig
from .circuit_breaker import CircuitBreaker, CircuitBreakerConfig, CircuitBreakerOpenError

logger = logging.getLogger(__name__)

T = TypeVar('T')


def retry_with_circuit_breaker(
    retry_config: Optional[RetryConfig] = None,
    breaker_config: Optional[CircuitBreakerConfig] = None,
):
    """
    重试 + 熔断器组合装饰器

    执行顺序:
    1. 先检查熔断器状态
    2. 如果熔断器打开，快速失败
    3. 如果熔断器闭合，执行重试逻辑
    4. 根据执行结果更新熔断器状态

    使用示例:
        @retry_with_circuit_breaker(
            retry_config=RetryConfig(max_attempts=3),
            breaker_config=CircuitBreakerConfig(failure_threshold=5)
        )
        def call_api():
            ...
    """
    if retry_config is None:
        retry_config = RetryConfig()

    if breaker_config is None:
        breaker_config = CircuitBreakerConfig()

    # 创建熔断器
    breaker = CircuitBreaker(breaker_config)

    def decorator(func: Callable[..., T]) -> Callable[..., T]:
        # 先应用重试装饰器
        retried_func = retry(retry_config)(func)

        @wraps(func)
        def wrapper(*args, **kwargs) -> T:
            # 再通过熔断器执行
            return breaker.call(retried_func, *args, **kwargs)

        # 添加属性
        wrapper.circuit_breaker = breaker
        wrapper.retry_config = retry_config

        return wrapper

    return decorator


# 使用示例

@retry_with_circuit_breaker(
    retry_config=RetryConfig(
        max_attempts=3,
        base_delay=1.0,
        exponential_base=2.0,
    ),
    breaker_config=CircuitBreakerConfig(
        failure_threshold=5,
        failure_rate_threshold=0.5,
        open_timeout=30,
    ),
)
def call_critical_api(endpoint: str) -> dict:
    """
    调用关键API

    - 最多重试3次
    - 5次失败后熔断
    - 熔断后30秒自动尝试恢复
    """
    import requests

    response = requests.get(endpoint, timeout=10)
    response.raise_for_status()
    return response.json()
```

### 3.5 实战案例：LLM调用熔断

```python
# app/service/llm_circuit_breaker.py

import logging
from typing import Optional
import os
from openai import OpenAI, RateLimitError, APITimeoutError, APIConnectionError

from app.core.circuit_breaker import CircuitBreaker, CircuitBreakerConfig
from app.core.circuit_breaker_manager import circuit_breaker_manager

logger = logging.getLogger(__name__)


class LLMCircuitBreaker:
    """LLM调用熔断器"""

    def __init__(self):
        # 为不同的LLM模型配置不同的熔断器
        self.gpt4_breaker = circuit_breaker_manager.get_or_create(
            "llm_gpt4",
            CircuitBreakerConfig(
                failure_threshold=3,        # GPT-4更容易触发限流，阈值设低
                failure_rate_threshold=0.3,
                open_timeout=60,            # 冷却时间60秒
                min_calls=5,
                excluded_exceptions=(      # 不计入熔断的异常
                    ValueError,
                    TypeError,
                ),
            )
        )

        self.deepseek_breaker = circuit_breaker_manager.get_or_create(
            "llm_deepseek",
            CircuitBreakerConfig(
                failure_threshold=5,
                failure_rate_threshold=0.5,
                open_timeout=30,
                min_calls=10,
            )
        )

    def call_gpt4(
        self,
        prompt: str,
        temperature: float = 0.7,
        max_tokens: int = 2000,
    ) -> Optional[str]:
        """
        调用GPT-4（带熔断保护）
        """
        def _call():
            client = OpenAI(api_key=os.getenv("OPENAI_API_KEY"))

            response = client.chat.completions.create(
                model="gpt-4",
                messages=[{"role": "user", "content": prompt}],
                temperature=temperature,
                max_tokens=max_tokens,
                timeout=60.0,
            )

            return response.choices[0].message.content

        try:
            return self.gpt4_breaker.call(_call)
        except Exception as e:
            logger.error(f"GPT-4 call failed: {e}")
            return None

    def call_deepseek(
        self,
        prompt: str,
        temperature: float = 0.7,
        max_tokens: int = 2000,
    ) -> Optional[str]:
        """
        调用Deepseek（带熔断保护）
        """
        def _call():
            client = OpenAI(
                api_key=os.getenv("DASHSCOPE_API_KEY"),
                base_url="https://dashscope.aliyuncs.com/compatible-mode/v1"
            )

            response = client.chat.completions.create(
                model="deepseek-chat",
                messages=[{"role": "user", "content": prompt}],
                temperature=temperature,
                max_tokens=max_tokens,
                timeout=60.0,
            )

            return response.choices[0].message.content

        try:
            return self.deepseek_breaker.call(_call)
        except Exception as e:
            logger.error(f"Deepseek call failed: {e}")
            return None

    def call_with_fallback(
        self,
        prompt: str,
        temperature: float = 0.7,
        max_tokens: int = 2000,
    ) -> Optional[str]:
        """
        调用LLM（带降级策略）

        降级链: GPT-4 -> Deepseek -> None
        """
        # 尝试GPT-4
        result = self.call_gpt4(prompt, temperature, max_tokens)
        if result:
            logger.info("Used GPT-4")
            return result

        logger.warning("GPT-4 failed, falling back to Deepseek")

        # 降级到Deepseek
        result = self.call_deepseek(prompt, temperature, max_tokens)
        if result:
            logger.info("Used Deepseek")
            return result

        logger.error("All LLM providers failed")
        return None

    def get_status(self) -> dict:
        """获取所有LLM熔断器状态"""
        return {
            "gpt4": self.gpt4_breaker.get_metrics(),
            "deepseek": self.deepseek_breaker.get_metrics(),
        }


# 使用示例

llm_breaker = LLMCircuitBreaker()


def analyze_with_llm(text: str) -> Optional[str]:
    """
    使用LLM分析文本（带熔断和降级）
    """
    prompt = f"Please analyze the following text:\n\n{text}"

    result = llm_breaker.call_with_fallback(prompt)

    # 查看熔断器状态
    status = llm_breaker.get_status()
    logger.debug(f"LLM Circuit Breaker Status: {status}")

    return result
```

***

## 4. 监控与告警

### 4.1 熔断器监控端点

```python
# app/api/circuit_breaker_monitor.py

from fastapi import APIRouter, HTTPException
from typing import Optional

from app.core.circuit_breaker_manager import circuit_breaker_manager

router = APIRouter(prefix="/api/circuit-breakers", tags=["monitoring"])


@router.get("/")
def get_all_breakers():
    """获取所有熔断器状态"""
    return circuit_breaker_manager.get_all_metrics()


@router.get("/{name}")
def get_breaker(name: str):
    """获取指定熔断器状态"""
    breaker = circuit_breaker_manager.get(name)
    if not breaker:
        raise HTTPException(status_code=404, detail="Circuit breaker not found")

    return breaker.get_metrics()


@router.post("/{name}/reset")
def reset_breaker(name: str):
    """重置熔断器"""
    breaker = circuit_breaker_manager.get(name)
    if not breaker:
        raise HTTPException(status_code=404, detail="Circuit breaker not found")

    circuit_breaker_manager.reset(name)
    return {"message": f"Circuit breaker {name} reset successfully"}


@router.get("/status/open")
def get_open_breakers():
    """获取所有打开的熔断器"""
    open_breakers = circuit_breaker_manager.get_open_breakers()

    return {
        "count": len(open_breakers),
        "breakers": open_breakers,
    }
```

***

## 5. 最佳实践

### 5.1 熔断器配置建议

```python
# app/config/circuit_breaker_profiles.py

from app.core.circuit_breaker import CircuitBreakerConfig


class CircuitBreakerProfiles:
    """熔断器配置预设"""

    # 敏感服务（快速熔断）
    SENSITIVE = CircuitBreakerConfig(
        failure_threshold=3,
        failure_rate_threshold=0.3,
        min_calls=5,
        window_size=30,
        open_timeout=60,
    )

    # 标准服务
    STANDARD = CircuitBreakerConfig(
        failure_threshold=5,
        failure_rate_threshold=0.5,
        min_calls=10,
        window_size=60,
        open_timeout=30,
    )

    # 容错服务（慢速熔断）
    TOLERANT = CircuitBreakerConfig(
        failure_threshold=10,
        failure_rate_threshold=0.7,
        min_calls=20,
        window_size=120,
        open_timeout=20,
    )
```

**配置预设对比&#x20;**：

### 5.2 熔断恢复策略

```python
# app/core/circuit_breaker_recovery.py

import logging
from typing import Callable, TypeVar, Optional

logger = logging.getLogger(__name__)

T = TypeVar('T')


class GradualRecoveryCircuitBreaker(CircuitBreaker):
    """渐进式恢复熔断器"""

    def __init__(self, config: CircuitBreakerConfig):
        super().__init__(config)
        self._recovery_percentage = 0.1  # 初始恢复10%流量

    def call(self, func: Callable[..., T], *args, **kwargs) -> T:
        """
        渐进式恢复执行

        HALF_OPEN状态下，逐步增加流量
        """
        import random

        # 检查状态
        if self.state == CircuitState.HALF_OPEN:
            # 只允许部分请求通过
            if random.random() > self._recovery_percentage:
                raise CircuitBreakerOpenError("In gradual recovery, request rejected")

        # 执行原有逻辑
        result = super().call(func, *args, **kwargs)

        # 如果成功，增加恢复百分比
        if self.state == CircuitState.HALF_OPEN:
            self._recovery_percentage = min(1.0, self._recovery_percentage + 0.1)
            logger.info(f"Recovery percentage increased to {self._recovery_percentage:.0%}")

        return result
```

***

## 6. 总结

熔断器是防止系统雪崩的关键机制。本文档提供了完整的企业级实现，包括:

1. **三态状态机&#x20;**：CLOSED/OPEN/HALF\_OPEN

2. **智能决策&#x20;**：基于失败率和失败次数

3) **渐进恢复&#x20;**：HALF\_OPEN状态的探测机制

4) **全局管理&#x20;**：统一的熔断器管理器

5. **组合使用&#x20;**：与重试、降级机制配合

在实际应用中，应根据服务特点调整熔断参数，并配合完善的监控告警体系。
