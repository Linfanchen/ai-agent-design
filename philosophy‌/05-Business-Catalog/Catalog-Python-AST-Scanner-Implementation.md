# Business Catalog：Python AST 扫描器实现

## 本文档在系统中的作用

给出 Python 源码静态扫描的实现边界和最小代码形状，帮助 Catalog Compiler 发现 FastAPI 路由、Pydantic 模型和权限装饰器。扫描器生成候选资产，不直接推断最终业务权限。

## 1. 为什么使用 AST

直接 import 被扫描项目可能触发模块顶层代码、数据库连接、环境变量读取和副作用。AST 允许在不执行代码的情况下分析语法结构，但不能完整理解动态注册、反射和运行时依赖注入。

## 2. 扫描器输出

每个 Asset 保存：
- asset_id；
- asset_type；
- source_file；
- start_line / end_line；
- git_commit；
- symbol_name；
- raw_signature；
- normalized_schema；
- decorators；
- references；
- parse_status。

## 3. 最小 AST 扫描示例

```python
import ast
from pathlib import Path
from dataclasses import dataclass

@dataclass
class FunctionAsset:
    name: str
    file_path: str
    line: int
    decorators: list[str]
    arguments: list[str]

class PythonAstScanner(ast.NodeVisitor):
    def __init__(self, file_path: str):
        self.file_path = file_path
        self.assets: list[FunctionAsset] = []

    def visit_FunctionDef(self, node: ast.FunctionDef):
        decorators = []
        for decorator in node.decorator_list:
            try:
                decorators.append(ast.unparse(decorator))
            except Exception:
                decorators.append("<unparsed>")
        arguments = [arg.arg for arg in node.args.args]
        self.assets.append(FunctionAsset(
            name=node.name,
            file_path=self.file_path,
            line=node.lineno,
            decorators=decorators,
            arguments=arguments,
        ))
        self.generic_visit(node)

def scan_file(path: Path) -> list[FunctionAsset]:
    source = path.read_text(encoding="utf-8")
    tree = ast.parse(source, filename=str(path))
    scanner = PythonAstScanner(str(path))
    scanner.visit(tree)
    return scanner.assets
```

此示例仅提取函数和装饰器文本。生产扫描器还需要处理 AsyncFunctionDef、类、注解、Import Alias、嵌套作用域、源码编码、语法错误和版本兼容。

## 4. FastAPI 路由识别

AST 可识别 `@router.get`、`@router.post` 等装饰器，但必须处理：
- Router Prefix；
- include_router；
- decorator alias；
- APIRouter 动态实例；
- 依赖注入；
- response_model；
- status_code；
- security dependency。

动态组合无法静态解析时标记为 `unresolved`，不要猜测最终 Path。

## 5. Pydantic Schema

Pydantic Model 的字段类型、Field 描述、Enum 和 Validator 可作为候选 Schema 来源。不要为了读取 Schema 而执行不可信仓库代码。若必须导入，应在隔离构建容器中执行，并限制环境变量和网络。

## 6. 权限注解识别

扫描 `Depends(require_permission(...))` 或自定义 Decorator 时，只能识别代码层面的候选声明。真正权限语义仍需与 Policy Registry 关联并人工审核。

## 7. 增量扫描

以 Git Commit / File Hash 为增量边界。文件未变化可复用解析结果；发生变化时重扫关联符号。删除文件或路由时要生成 Tombstone / Drift Event，不能只追加新资产。

## 8. 测试

建立小型 Fixture Repo，覆盖普通路由、异步路由、别名、嵌套 Router、动态 Prefix、Pydantic 模型、Enum、权限依赖和语法错误。验证行号、签名、Schema 和 unresolved 状态。
