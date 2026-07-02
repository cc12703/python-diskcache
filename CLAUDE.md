# CLAUDE.md


## 要点
- 始终使用中文进行输出

## 项目

- `diskcache` — 纯 Python、Apache2 授权的磁盘缓存库
- 零运行时依赖（`setup.py` 中 `install_requires=[]`）
- SQLite 存元数据，大值溢出到文件（超过 `disk_min_file_size`）
- SQLite 保证线程/进程安全
- 版本号在 `diskcache/__init__.py`（`__version__`/`__build__`）。

## 常用命令

全部由 **tox** 驱动（见 `tox.ini` 与 `.github/workflows/integration.yml`）。

### 测试

```bash
uv run tox -e py              # 完整套件，匹配 CI（py38..py311）
uv run pytest                 # 装好 dev 依赖后直接跑

# 单测：禁用 xdist 与 coverage
DJANGO_SETTINGS_MODULE=tests.settings PYTHONPATH=. \
  pytest tests/test_core.py::TestCore::test_get -p no:xdist -p no:cacheprovider --no-cov

# Django 测试需要 settings + PYTHONPATH 在仓库根
DJANGO_SETTINGS_MODULE=tests.settings PYTHONPATH=. pytest tests/test_djangocache.py
```

`[pytest]`（`tox.ini`）关键项：
- `-n auto`（xdist 并行）
- `--cov=diskcache --cov-fail-under=98 --cov-branch` — **覆盖率低于 98% CI 即失败**
- `--doctest-glob="*.rst"` — RST doctest 也算测试
- 多个 `tests/` 脚本被 `--ignore`（benchmark、plot、`issue_*.py`）

### Lint / 类型 / 格式化

```bash
uv run tox -e bluecheck       # blue 格式化器（项目标准，不是 black），仅检查
uv run tox -e blue            # 应用 blue
uv run tox -e isortcheck      # isort 仅检查
uv run tox -e isort           # 应用 isort
uv run tox -e flake8          # max-line-length=120，排除 tests/test_djangocache.py，忽略 E203
uv run tox -e pylint          # 配置在 .pylintrc
uv run tox -e mypy            # mypy diskcache/
uv run tox -e doc8            # docs/ 的 RST lint
uv run tox -e rstcheck        # 仅 README.rst
uv run tox -e docs            # sphinx HTML（cd docs && make html）
```

注意：isort 行宽 79，flake8 行宽 120。格式化以 `blue` 为准。

### 压测与基准（不在 CI 测试路径内）

```bash
python -m tests.stress_test_core --help     # -n ops -k keys -t threads -p processes -v policy
python tests/benchmark_core.py --help       # 产出 tests/timings_*.txt
python tests/plot.py tests/timings_core_p1.txt
```

## 架构

### 模块（`diskcache/`）

| 模块 | 职责 |
|---|---|
| `core.py` | `Cache`、`Disk`、`JSONDisk`。地基，其他模块都基于 `Cache`。 |
| `fanout.py` | `FanoutCache` — 把 key 分片到 N 个 `Cache`（默认 8），提升写入并发。每分片独立目录+SQLite。 |
| `persistent.py` | `Deque`、`Index` — 基于 `Cache` 的持久容器。 |
| `recipes.py` | `Averager`、`Lock`、`RLock`、`BoundedSemaphore`、`barrier`、`throttle`、`memoize_stampede`。 |
| `djangocache.py` | `DjangoCache` — 继承 Django `BaseCache`，包装 `FanoutCache`。可选导入，Django 缺失时 `__init__.py` 吞掉 ImportError。 |
| `cli.py` | 占位；命令行是开放贡献项（见 `docs/development.rst`）。 |

### 核心设计

- `Cache`（`core.py`）= SQLite（`cache.db`）+ 文件目录。`Cache` 表以 rowid 为主键，按淘汰策略建索引（`Cache_store_time` / `Cache_access_time` / `Cache_access_count` / `Cache_tag_rowid` / `Cache_expire_time`）。最近两次提交把其中两个改成了 **partial index**（`WHERE col IS NOT NULL`）— 新增可空索引列时沿用此模式。
- `Disk` 子类控制序列化。五种存储模式（`MODE_NONE/RAW/BINARY/TEXT/PICKLE`），选最省的。重写 `Disk.put`/`Disk.get` 自定义（参考 `JSONDisk`）。
- 淘汰策略是 **数据而非代码**：`EVICTION_POLICY` 字典把策略名映射到 SQL 片段（`init`/`get`/`cull`），`Cache` 拼成查询。新增策略=加一条，无需改方法。默认 `least-recently-stored`。
- `DEFAULT_SETTINGS` 列出所有可调项（`size_limit`、`cull_limit`、`sqlite_journal_mode='wal'` 等）。`Cache.__init__` 接受其中任意 kwarg；未知 key 报错。

### FanoutCache 分片

委托给 N 个独立 `Cache` 分片。Key → 分片：`self._hash(key) % self._count`。`size_limit` 平均分到各分片。`cull_limit` 等是 **每分片** 而非全局 — 淘汰语义需注意。

### 测试布局（`tests/`）

- `test_*.py` — pytest 收集的单测。`test_core.py` 和 `test_djangocache.py` 是大头。
- `stress_test_*.py` — 长时间随机压测（含多进程变体），手动运行，不在 CI。
- `benchmark_*.py` — 微基准，产出 `timings_*.txt`。
- `settings.py` — 测试用 Django settings，把 `CACHES['default']` 指向 `diskcache.DjangoCache`。需 `DJANGO_SETTINGS_MODULE=tests.settings`。
- `issue_*.py` — 回归复现脚本，pytest 显式排除。

### 约定

- 纯 Python，无 C 扩展、无可选二进制依赖。
- Docstring 量大且含 doctest（RST + Python）。`docs/tutorial.rst` 的 doctest 会跑很多公开方法 — 改签名/repr 常会破坏 RST 测试。
- 格式化器是 `blue`，不要用 `black` 输出替代。
- 新增公开 API：在 `diskcache/__init__.py` 的 `__all__` 导出，并在 `docs/api.rst` + tutorial 记文档。
