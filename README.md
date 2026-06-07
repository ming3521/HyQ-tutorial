# HyQ：混合量子算法工具包 -- 使用教程
由本人主要负责开发的华翊量子化学算法库 HyQ (曾用名：HYQ-ALG-LIB) 已在 PyPI 开源，网址：https://pypi.org/project/hyq/ 

此Github仓库是基于HyQ量子化学库的一些使用示例与教程，包含编译功能使用、分子哈密顿量生成、VQE/VarQITE/SA-QITE等变分量子化学算法的原理及其求解器的调用方式 等。 用户可以在安装好 HyQ 算法库 之后，使用本仓库中的Jupyter Book文档学习库的使用与开发方法。

HyQ 是一个轻量级 Python 工具包，用于构建与后端无关的量子线路并运行混合量子算法，例如 VQE、VarQITE、SA‑QITE、SS‑QITE 和 RITE。它提供：

- 用于构建线路的统一 `QuantumCircuit` 抽象。
- 可插拔的后端接口，支持两种执行模式：
    - 采样模式：返回测量计数，类似于硬件或基于采样的模拟器。
    - 状态向量模式：返回精确状态向量，适用于模拟器和可微分算法。
- 针对 **Qiskit**、**PennyLane** 和 **TensorCircuit** 的后端实现。
- 统一 **Hamiltonian** 格式表示 `qubit Hamiltonian`
- 可选化学工具，使用 **OpenFermion** 和 **Psi4** 生成分子哈密顿量。
- `VQE`、`VarQITE`、`QITE` 系列求解器，以及 `classical reference`、`Trotter`、`QPE`、`QSE`、`VQD`、`classical shadow` 等算法工具。


## 推荐的仓库结构

```text
.
├── algorithms/
├── ansatz/
├── backends/
├── chemistry/
│   ├── __init__.py
│   ├── hamiltonian.py
│   ├── molecule.py
│   └── psi4_driver.py
├── solvers/
│   ├── __init__.py
│   ├── base.py
│   ├── vqe.py
│   ├── var_qite.py
│   ├── sa_qite.py
│   ├── ss_qite.py
│   └── rite.py
├── examples/
├── tutorials/
├── requirements.txt
└── README.md
```

## 安装

```bash
pip install hyq
```

默认可用 Qiskit 后端（线路构建、采样、QASM 等）。若需要可微分或其它模拟器，可加装 extras：

```bash
pip install "hyq[tensorcircuit]"              # VQE / VarQITE 等可微分算法
pip install "hyq[tensorcircuit,pennylane]"    # 常用组合
pip install "hyq[all]"                        # 全部可选 Python 依赖（体积较大）
```

### 可选依赖（extras）

| Extra | 用途 |
|-------|------|
| `pennylane` | PennyLane 后端 |
| `tensorcircuit` | TensorCircuit 后端（推荐可微分流程） |
| `optional-backends` | Cirq、Qulacs、QuTiP |
| `chemistry` | OpenFermion、PySCF（分子哈密顿量，**默认 PySCF**） |
| `psi4` | 仅 openfermionpsi4 桥接包；**须**已 `conda install psi4` |
| `notebooks` | 运行 notebook 的辅助包 |
| `dev` | pytest、black 等开发工具 |
| `all` | 上述除 `dev` 外的组合 |

### `pip install hyq` 和 `pip install "hyq[chemistry]"` 的区别

| 安装命令 | 自动安装的内容 | 能否用 `hyq.chemistry` |
|----------|----------------|-------------------------|
| `pip install hyq` | NumPy、SciPy、PyTorch、Qiskit、qiskit-aer、pylatexenc | **不能**（缺 OpenFermion / PySCF） |
| `pip install "hyq[chemistry]"` | 上述 **加上** openfermion、openfermionpyscf、pyscf（**不含** openfermionpsi4） | **可以**（走 **PySCF**） |
| `pip install "hyq[chemistry,psi4]"` | 再加 openfermionpsi4；**还须** Conda 安装 `psi4` 程序 | 使用 Psi4 后端 |

只做 VQE、QPE、线路编译等 **不需要** `[chemistry]`。运行 `examples/hamiltonian.ipynb` 或 `from hyq.chemistry import Hamiltonian` **需要** `[chemistry]`。


### 方式一：仅 pip（推荐，无需 Psi4）

Python 版本须为 **3.10 或 3.11**（与 `hyq` 一致）。

```bash
pip install -U pip
pip install "hyq[chemistry]"
```

验证：

```bash
python -c "import hyq; from hyq.chemistry import Hamiltonian; print('hyq', hyq.__version__)"
python -c "from openfermionpyscf import run_pyscf; import pyscf; print('PySCF ok')"
```

最小用法：

```python
from hyq.chemistry import Hamiltonian

ham = Hamiltonian(
    symbols=["H", "H"],
    geometry=[[0.0, 0.0, 0.0], [0.0, 0.0, 0.74]],
    basis="sto-3g",
)
terms, n_qubits, n_electrons = ham.get_processed_hamiltonian(
    n_active_electrons=2,
    n_active_orbitals=2,
    mapping="jw",
)
```

### 方式二：使用 Psi4（可选，需 Conda）

**Psi4 本体不在 PyPI**。默认 `hyq[chemistry]` **不会**安装 `openfermionpsi4`（避免仅有 Python 包、没有 `psi4` 命令时触发第三方库 bug）。需要 Psi4 时请：

```bash
# 1. 用 Conda 安装 Psi4（示例）
conda create -n hyq_chem python=3.11 psi4=1.10 pip -y
conda activate hyq_chem

# 2. 再装 hyq：化学 + Psi4 桥接
pip install "hyq[chemistry,psi4]"
```

验证：

```bash
python -c "import psi4; print('psi4', psi4.__version__)"
python -c "from openfermionpsi4 import run_psi4; print('openfermionpsi4 ok')"
```

可选环境变量（磁盘/权限异常时）：

```bash
export PSI4_SCRATCH=/tmp
export TMPDIR=/tmp
```

运行时会使用当前目录下的 `.psi4_temp_data/` 存放临时文件。

### 从源码开发（Conda 一键环境）

克隆仓库后可用 `environment.yml`（已包含 Psi4 + PySCF + 全部 pip 依赖）：

```bash
conda env create -f environment.yml
conda activate hyq_alg_lib_env
pip install -e .
```

详见仓库内 **`README_HYQ.md`**。

### 常见报错

若出现：

```text
ImportError: 需要安装 openfermionpyscf/pyscf，或安装 openfermionpsi4 并配置 Psi4。
```

表示当前环境 **未安装** `hyq[chemistry]`。请执行 `pip install "hyq[chemistry]"`。

若出现 `UnboundLocalError: process`（来自 `openfermionpsi4`），说明曾安装 `openfermionpsi4` 但 **没有** `psi4` 可执行文件：请 `pip uninstall openfermionpsi4`，或改用 `26.1.4+` 的 `hyq`（默认不再拉取该包），或按上文安装 Conda 版 Psi4 后使用 `hyq[chemistry,psi4]`。

## 导入方式

安装后统一从 `hyq` 命名空间导入（与 `pip` 包名一致）：

```python
import hyq
print(hyq.__version__)

from hyq.backends import get_backend, available_backends, set_backend
from hyq.backends.core import QuantumCircuit
from hyq.backends.Tensorcircuit import TensorCircuitBackend
from hyq.solvers import VQESolver
from hyq.algorithms import exact_diagonalization, build_qpe_circuit
from hyq.ansatz import HEAAnsatz, UCCSD
from hyq.chemistry import Hamiltonian
```

## 选择默认后端

```bash
export HYQ_BACKEND=tensorcircuit   # 或 qiskit、pennylane 等
```

```python
from hyq.backends import available_backends, set_backend

print(available_backends())
backend = set_backend("tensorcircuit")
```

## 快速示例（VQE）

```python
import torch
from hyq.backends.core import QuantumCircuit
from hyq.backends.Tensorcircuit import TensorCircuitBackend
from hyq.solvers import VQESolver

backend = TensorCircuitBackend()
solver = VQESolver(backend)

def ansatz(params):
    qc = QuantumCircuit(2)
    qc.ry(0, params[0])
    qc.cx(0, 1)
    return qc

hamiltonian = [(-1.0, "ZI"), (-1.0, "IZ"), (0.5, "XX")]
init_params = torch.tensor([0.1], dtype=torch.float64)

energy, params, history = solver.solve(
    ansatz, init_params, hamiltonian, steps=100, lr=0.1
)
print(energy, params)
```

## Hamiltonian 标准格式

最新版本统一推荐使用如下格式：

```python
hamiltonian = [
    (-1.0, "ZI"),
    (-1.0, "IZ"),
    (0.5, "XX"),
]
```

其中字符串第 `i` 个字符对应第 `i` 个 qubit，支持 `I/X/Y/Z`。

## 后端能力概览

| 后端 | 采样 `run_sampling` | 状态向量 `get_statevector` | PyTorch 自动微分 | 推荐用途 |
|---|---:|---:|---:|---|
| Qiskit | 支持 | 支持 | 不支持 | 采样、QASM、绘图、编译 |
| PennyLane | 支持 | 支持 | 支持 | 可微分模拟和算法原型 |
| TensorCircuit | 支持 | 支持 | 支持 | VQE / VarQITE 等梯度算法 |
| Cirq | 支持 | 支持 | 不支持 | 备用门级模拟后端 |
| Qulacs | 支持 | 支持 | 不支持 | 快速状态向量模拟 |
| QuTiP | 支持 | 支持 | 不支持 | QuTiP 生态入口和教学 |

可用后端可通过：

```python
from backends import available_backends, set_backend, get_backend

print(available_backends())
backend = set_backend("tensorcircuit")
```


## 最小线路示例

```python
import torch
from backends.core import QuantumCircuit
from backends.Tensorcircuit import TensorCircuitBackend

qc = QuantumCircuit(2)
qc.h(0)
qc.cx(0, 1)

backend = TensorCircuitBackend()
state = backend.get_statevector(qc)
print(state)

counts = backend.run_sampling(qc, shots=1024, measure_qubits=[0, 1])
print(counts)
```
## 使用求解器

```python
import torch
import hyq
print(hyq.__version__)
from hyq.backends.core import QuantumCircuit
from hyq.backends.Tensorcircuit import TensorCircuitBackend
from hyq.solvers.vqe import VQESolver

backend = TensorCircuitBackend()
solver = VQESolver(backend)

def ansatz(params):
    qc = QuantumCircuit(2)
    qc.ry(0, params[0])
    qc.cx(0, 1)
    return qc

hamiltonian = [(-1.0, "ZI"), (-1.0, "IZ"), (0.5, "XX")]
init_params = torch.tensor([0.1], dtype=torch.float64)
energy, params, history = solver.solve(ansatz, init_params, hamiltonian, steps=100, lr=0.1)
```

## 生成分子哈密顿量

```python
from hyq.chemistry.hamiltonian import Hamiltonian

symbols = ["H", "H"]
geometry = [[0.0, 0.0, 0.0], [0.0, 0.0, 0.74]]

ham = Hamiltonian(symbols, geometry, charge=0, multiplicity=1, basis="sto-3g")
terms, n_qubits, n_electrons = ham.get_processed_hamiltonian(
    n_active_electrons=2,
    n_active_orbitals=2,
    mapping="jw",
    reverse_endian=False,
)
```


