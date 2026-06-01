# HYQ-ALG-LIB：混合量子算法工具包
HYQ-ALG-LIB 是一个轻量级 Python 工具包，用于构建与后端无关的量子线路并运行混合量子算法，例如 VQE、VarQITE、SA‑QITE、SS‑QITE 和 RITE。它提供：

- 用于构建线路的统一 `QuantumCircuit` 抽象。
- 可插拔的后端接口，支持两种执行模式：
    - 采样模式：返回测量计数，类似于硬件或基于采样的模拟器。
    - 状态向量模式：返回精确状态向量，适用于模拟器和可微分算法。
- 针对 Qiskit、PennyLane 和 TensorCircuit 的后端实现。
- 统一 Hamiltonian 格式表示 qubit Hamiltonian
- 可选化学工具，使用 OpenFermion 和 Psi4 生成分子哈密顿量。
- VQE、VarQITE、QITE 系列求解器，以及 classical reference、Trotter、QPE、QSE、VQD、classical shadow 等算法工具。


## 推荐的仓库结构

```text
.
├── algorithms/
│   ├── __init__.py
│   ├── classical_eigensolvers.py
│   ├── phase_estimation.py
│   ├── qse.py
│   ├── quantum_chemistry.py
│   ├── shadow.py
│   ├── trotter.py
│   └── vqd.py
├── ansatz/
│   ├── __init__.py
│   ├── base.py
│   ├── HEA.py
│   ├── ryrz.py
│   ├── compact.py
│   └── adapt.py
├── backends/
│   ├── __init__.py
│   ├── base.py
│   ├── core.py
│   ├── _matrix_simulator.py
│   ├── Qiskit.py
│   ├── Pennylane.py
│   ├── Tensorcircuit.py
│   ├── Cirq.py
│   ├── Qulacs.py
│   └── Qutip.py
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
├── requirements-qiskit.txt
├── requirements-pennylane.txt
├── requirements-tensorcircuit.txt
├── requirements-optional-backends.txt
├── requirements-chemistry.txt
├── requirements-dev.txt
└── README.md
```

## 安装

首先创建并激活虚拟环境。

```bash
python -m venv .venv
source .venv/bin/activate      # Linux / macOS
# .venv\Scripts\activate       # Windows PowerShell

pip install -U pip
```

安装核心依赖：
```bash
pip install -r requirements.txt
```

根据您的使用场景安装至少一个后端。

### TensorCircuit 后端：推荐用于可微分状态向量算法

```bash
pip install -r requirements-tensorcircuit.txt
export HYQ_BACKEND=tensorcircuit      # Linux / macOS
# set HYQ_BACKEND=tensorcircuit       # Windows CMD
# $env:HYQ_BACKEND="tensorcircuit"    # Windows PowerShell
```

### Qiskit 后端：推荐用于线路转换、QASM、绘图和采样风格工作流

```bash
pip install -r requirements-qiskit.txt
export HYQ_BACKEND=qiskit
```

### PennyLane 后端：推荐用于可微分算法原型开发

```bash
pip install -r requirements-pennylane.txt
export HYQ_BACKEND=pennylane
```

### 可选后端：Cirq / Qulacs / QuTiP

```bash
pip install -r requirements-optional-backends.txt
```

### 量子化学依赖

```bash
pip install -r requirements-chemistry.txt
```

Psi4 在部分平台上不适合通过 pip 安装。如需使用 `chemistry/psi4_driver.py`，推荐使用 conda 安装 Psi4：

```bash
conda create -n psi4-env -c conda-forge psi4
conda activate psi4-env
python -c "import psi4; print(psi4.__version__)"
```

### 开发和 Notebook 环境

```bash
pip install -r requirements-dev.txt
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
from backends.core import QuantumCircuit
from backends.Tensorcircuit import TensorCircuitBackend
from solvers.vqe import VQESolver

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
from chemistry.hamiltonian import Hamiltonian

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

## VQE 示例

```python
import torch
from backends.core import QuantumCircuit
from backends.Tensorcircuit import TensorCircuitBackend
from solvers.vqe import VQESolver

backend = TensorCircuitBackend()
solver = VQESolver(backend)

def ansatz(params):
    qc = QuantumCircuit(2)
    qc.ry(0, params[0])
    qc.cx(0, 1)
    return qc

hamiltonian = [
    (-1.0, "ZI"),
    (-1.0, "IZ"),
    (0.5, "XX"),
]

init_params = torch.tensor([0.1], dtype=torch.float64)
energy, params, history = solver.solve(
    ansatz,
    init_params,
    hamiltonian,
    steps=100,
    lr=0.1,
)

print(energy)
print(params)
```

## Quantum Phase Estimation

```python
import numpy as np
from backends.core import QuantumCircuit
from algorithms.phase_estimation import build_qpe_circuit, phases_from_counts

phi = 0.375
unitary = QuantumCircuit(1)
unitary.p(0, 2 * np.pi * phi)

qpe = build_qpe_circuit(unitary, n_ancilla=3)
print(qpe)

# 后端采样后：
# counts = backend.run_sampling(qpe, shots=2048, measure_qubits=[0, 1, 2])
# result = phases_from_counts(counts, n_ancilla=3)
```

## QSE：Quantum Subspace Expansion

```python
import numpy as np
from algorithms.qse import quantum_subspace_expansion

reference_state = np.array([1.0, 0.0], dtype=np.complex128)
hamiltonian = [(1.0, "Z"), (0.2, "X")]
excitation_terms = [[(1.0, "X")]]

result = quantum_subspace_expansion(
    reference_state=reference_state,
    hamiltonian=hamiltonian,
    excitation_terms=excitation_terms,
    n_qubits=1,
)

print(result.energies)
print(result.kept_subspace_dimension)
```

## VQD：Variational Quantum Deflation 辅助工具

```python
import numpy as np
from algorithms.vqd import deflated_eigensystem, vqd_objective_from_matrix

H = np.diag([0.0, 1.0, 2.0]).astype(np.complex128)
ground_state = np.array([1.0, 0.0, 0.0], dtype=np.complex128)

vals, vecs = deflated_eigensystem(H, previous_states=[ground_state], betas=[5.0])
print(vals)

breakdown = vqd_objective_from_matrix(
    H,
    candidate_state=vecs[:, 0],
    previous_states=[ground_state],
    betas=[5.0],
)
print(breakdown)
```

## Classical Shadow

```python
from algorithms.shadow import ClassicalShadow

shadow = ClassicalShadow(
    backend=backend,
    circuit=qc,
    seed=123,
)

shadow.collect(5000)
print(shadow.estimate_observable("XX"))

hamiltonian = [(1.0, "II"), (0.5, "XX"), (-0.3, "ZZ")]
print(shadow.estimate_hamiltonian(hamiltonian))
```

## 量子化学辅助函数

```python
import numpy as np
from algorithms.quantum_chemistry import (
    fci_reference_energy,
    chemical_accuracy_error,
    mp2_energy_from_integrals,
    state_fidelity,
    particle_number_expectation,
    spin_z_expectation,
)

hamiltonian = [(1.0, "Z"), (0.2, "X")]
ref = fci_reference_energy(hamiltonian, n_qubits=1)
print(ref)
print(chemical_accuracy_error(estimated_energy=-1.018, reference_energy=ref))

state = np.array([0.0, 0.0, 0.0, 2.0], dtype=np.complex128)
print(particle_number_expectation(state, n_qubits=2))
print(spin_z_expectation(state, n_qubits=2))
```


