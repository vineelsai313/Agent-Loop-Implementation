"""
ASP Planning & Diagnosis GUI - v13 Compact Header Scenario Sessions

This GUI follows the same workflow as test_file_v7.py:
1. Run step 0 planning.
2. For each later observation step M:
   - copy scenario_step_prev_m_planning.txt to scenario_step_M_diagnosis.txt
   - update #const m=M
   - append obs(..., M)
   - append hpd(...) from output_step_prev_m_planning.txt for all prior actions I < M
   - run diagnosis
   - copy diagnosis scenario to planning scenario
   - append occurs(...) from expl(...)
   - run planning
   - if UNSAT, write Answer Set with agent_occurs(stop(L),M) and KEEP prev_m=M

Main fixes:
- Do NOT reset prev_m / plan_length to None after UNSAT at step M.
- Write UNSAT wait output as agent_occurs(stop(L),M), not hpd(...), so the next step can parse it.
"""

import sys
import subprocess
import re
import shutil
from datetime import datetime
from pathlib import Path
from dataclasses import dataclass, field
from typing import List, Optional, Tuple

from PyQt5.QtCore import Qt, QObject, QThread, pyqtSignal
from PyQt5.QtGui import QFont, QColor
from PyQt5.QtWidgets import (
    QApplication,
    QFileDialog,
    QFrame,
    QGridLayout,
    QGroupBox,
    QHBoxLayout,
    QHeaderView,
    QLabel,
    QLineEdit,
    QMainWindow,
    QMessageBox,
    QPushButton,
    QComboBox,
    QSpinBox,
    QTabWidget,
    QTableWidget,
    QTableWidgetItem,
    QTextEdit,
    QVBoxLayout,
    QWidget,
)

# ----------------------------------------------------------------------
# Configuration
# ----------------------------------------------------------------------

CLINGO_CMD = "clingo"
WORK_DIR = Path(".")

PLANNING_FILES = [
    "dynamic_domain.txt",
    "policies.txt",
    "planning.txt",
    "reality_check.txt",
    "applicable_policies.txt",
    "penalty_and_time.txt",
]
DIAGNOSIS_FILES = ["dynamic_domain.txt", "reality_check.txt", "diagnosis.txt"]
FIND_LOC_FILES = ["dynamic_domain.txt", "reality_check.txt", "find_loc.txt"]


# ----------------------------------------------------------------------
# Runtime state
# ----------------------------------------------------------------------

@dataclass
class PlannerState:
    prev_m: Optional[int] = None
    plan_length: Optional[int] = None
    last_cum_time: Optional[int] = None
    last_cum_penalty: Optional[int] = None
    last_status: str = "Not started"
    active_run_dir: Optional[Path] = None
    input_dir: Optional[Path] = None
    output_dir: Optional[Path] = None
    log_dir: Optional[Path] = None
    history: List[str] = field(default_factory=list)


# ----------------------------------------------------------------------
# Backend: clingo and parsing
# ----------------------------------------------------------------------

def run_subprocess(args: List[str], cwd: Path) -> Tuple[str, str, int]:
    try:
        result = subprocess.run(args, capture_output=True, text=True, cwd=str(cwd))
        return result.stdout, result.stderr, result.returncode
    except FileNotFoundError:
        return "", f"Command not found: {args[0]}", 1


def is_unsat(clingo_output: str) -> bool:
    return "UNSATISFIABLE" in clingo_output


def extract_last_answer_line(clingo_output: str) -> Optional[str]:
    lines = clingo_output.strip().splitlines()
    answers = []
    for i, line in enumerate(lines):
        if line.startswith("Answer:") and i + 1 < len(lines):
            answers.append(lines[i + 1].strip())
    return answers[-1] if answers else None


def parse_agent_occurs_from_answer(last_line: Optional[str]) -> List[Tuple[str, int]]:
    if not last_line:
        return []
    res = []
    for tok in last_line.split():
        if not tok.startswith("agent_occurs("):
            continue
        atom = tok.rstrip(" .")
        if not atom.endswith(")"):
            continue
        inner = atom[len("agent_occurs("):-1]
        idx = inner.rfind(",")
        if idx == -1:
            continue
        action = inner[:idx].strip()
        time_s = inner[idx + 1:].strip()
        if time_s.isdigit():
            res.append((action, int(time_s)))
    return res


def parse_metrics_from_answer(last_line: Optional[str]) -> Tuple[Optional[int], Optional[int]]:
    if not last_line:
        return None, None
    tm = re.search(r"cumulative_time\((\d+)\)", last_line)
    pm = re.search(r"cumulative_penalty\((\d+)\)", last_line)
    return (int(tm.group(1)) if tm else None, int(pm.group(1)) if pm else None)


def _parse_explanations_from_text(answer_text: str) -> List[Tuple[str, int]]:
    """Parse expl(A,I) from a single answer text, including nested commas."""
    expls = []
    s = answer_text
    key = "expl("
    n = len(s)
    i = 0
    while True:
        start = s.find(key, i)
        if start == -1:
            break
        j = start + len(key)
        depth = 1
        while j < n and depth > 0:
            if s[j] == "(":
                depth += 1
            elif s[j] == ")":
                depth -= 1
                if depth == 0:
                    break
            j += 1
        if depth != 0:
            i = start + len(key)
            continue
        inner = s[start + len(key):j].strip()
        idx = inner.rfind(",")
        if idx != -1:
            action = inner[:idx].strip()
            time_s = inner[idx + 1:].strip()
            if time_s.isdigit():
                expls.append((action, int(time_s)))
        i = j + 1
    return expls


def extract_optimal_answer_text(clingo_output: str) -> Optional[str]:
    """
    Return the answer text for the final optimal model.

    Clingo prints improved models as:
      Answer: 1
      <atoms for model 1>
      Optimization: ...
      Answer: 2
      <atoms for better model 2>
      Optimization: ...
      OPTIMUM FOUND

    The correct explanation atoms are only in the last completed Answer block
    before OPTIMUM FOUND, not in all Answer blocks combined.
    """
    lines = clingo_output.splitlines()
    current_answer_lines = []
    last_completed_answer = None
    inside_answer = False

    for raw_line in lines:
        line = raw_line.strip()

        if line.startswith("Answer:"):
            inside_answer = True
            current_answer_lines = []
            continue

        if line.startswith("Optimization:"):
            if inside_answer:
                last_completed_answer = " ".join(current_answer_lines).strip()
            inside_answer = False
            current_answer_lines = []
            continue

        if line.startswith("OPTIMUM FOUND"):
            if inside_answer and current_answer_lines:
                last_completed_answer = " ".join(current_answer_lines).strip()
            break

        if inside_answer:
            current_answer_lines.append(line)

    # Fallback for SAT outputs that do not contain Optimization/OPTIMUM FOUND.
    if last_completed_answer is None:
        return extract_last_answer_line(clingo_output)

    return last_completed_answer


def parse_explanations_v2(clingo_output: str) -> List[Tuple[str, int]]:
    """Parse expl(A,I) only from the final optimal answer block."""
    optimal_answer = extract_optimal_answer_text(clingo_output)
    if not optimal_answer:
        return []
    return _parse_explanations_from_text(optimal_answer)



def run_planning(scenario_file: Path) -> Tuple[str, str, int]:
    return run_subprocess([CLINGO_CMD, *PLANNING_FILES, str(scenario_file)], WORK_DIR)


def run_diagnosis(scenario_file: Path) -> Tuple[str, str, int]:
    return run_subprocess([CLINGO_CMD, *DIAGNOSIS_FILES, str(scenario_file)], WORK_DIR)


def run_find_loc(scenario_planning_file: Path) -> Tuple[Optional[int], str, str]:
    stdout, stderr, _ = run_subprocess(
        [CLINGO_CMD, "dynamic_domain.txt", "reality_check.txt", str(scenario_planning_file), "find_loc.txt"],
        WORK_DIR,
    )
    match = re.search(r"at_loc\((\d+)\)", stdout)
    return (int(match.group(1)) if match else None, stdout, stderr)


# ----------------------------------------------------------------------
# Backend: scenario-session folders and file generation
# ----------------------------------------------------------------------

def require_files() -> List[str]:
    needed = sorted(set(PLANNING_FILES + DIAGNOSIS_FILES + FIND_LOC_FILES))
    return [name for name in needed if not (WORK_DIR / name).exists()]


def safe_name_piece(value: str) -> str:
    value = value.strip().replace(" ", "_")
    return re.sub(r"[^A-Za-z0-9_\-]", "", value) or "value"


def create_run_directories(start_loc: int, goal_loc: int, mode: str) -> Tuple[Path, Path, Path, Path]:
    """Create one isolated folder for a complete scenario run."""
    timestamp = datetime.now().strftime("%Y-%m-%d_%H-%M-%S")
    folder_name = f"scenario_{timestamp}_start{start_loc}_goal{goal_loc}_{safe_name_piece(mode)}"
    run_dir = WORK_DIR / "runs" / folder_name
    input_dir = run_dir / "inputs"
    output_dir = run_dir / "outputs"
    log_dir = run_dir / "logs"
    input_dir.mkdir(parents=True, exist_ok=True)
    output_dir.mkdir(parents=True, exist_ok=True)
    log_dir.mkdir(parents=True, exist_ok=True)
    return run_dir, input_dir, output_dir, log_dir


def create_step0_planning_file(
    start_loc: int,
    goal_loc: int,
    emergency_status: str,
    observations: List[Tuple[str, bool]],
    filename: str,
    input_dir: Path,
) -> Path:
    path = input_dir / filename
    goal_fluent = f"at_loc({goal_loc})"
    with path.open("w", encoding="utf-8") as f:
        f.write("#const m=0.\n \n")
        for fluent, truth in observations:
            if truth:
                f.write(f"obs({fluent}, true, 0).\n")
                f.write(f"holds({fluent}, 0).\n\n")
            else:
                f.write(f"obs({fluent}, false, 0).\n")
                f.write(f"-holds({fluent}, 0).\n\n")
        f.write(f"goal(I) :- holds({goal_fluent}, I), step(I).\n\n")
        f.write(f"{emergency_status}.\n\n")
    return path


def copy_and_update_m(prev_scenario_file: str, new_scenario_file: str, prev_m: int, new_m: int, input_dir: Path) -> Path:
    prev_path = input_dir / prev_scenario_file
    new_path = input_dir / new_scenario_file
    text = prev_path.read_text(encoding="utf-8")
    text = re.sub(rf"#const\s+m\s*=\s*{prev_m}\s*\.", f"#const m={new_m}.", text)
    new_path.write_text(text, encoding="utf-8")
    return new_path


def add_observations_to_file(filename: str, observations: List[Tuple[str, bool]], time_step: int, input_dir: Path) -> None:
    with (input_dir / filename).open("a", encoding="utf-8") as f:
        f.write("\n")
        for fluent, truth in observations:
            f.write(f"obs({fluent}, {'true' if truth else 'false'}, {time_step}).\n")
        f.write("\n")


def add_hpd_from_previous_plan(prev_m: int, scenario_diagnosis_file: str, M: int, input_dir: Path, output_dir: Path) -> List[str]:
    prev_output = output_dir / f"output_step_{prev_m}_planning.txt"
    added = []
    if not prev_output.exists():
        return added
    lines = prev_output.read_text(encoding="utf-8").splitlines()
    last_line = None
    for i, line in enumerate(lines):
        if line.startswith("Answer Set:") and i + 1 < len(lines):
            last_line = lines[i + 1].strip()
    if last_line is None:
        return added
    actions = parse_agent_occurs_from_answer(last_line)
    with (input_dir / scenario_diagnosis_file).open("a", encoding="utf-8") as f:
        f.write("\n")
        for action, t in actions:
            if t < M:
                line = f"hpd({action}, true, {t})."
                f.write(line + "\n")
                added.append(line)
        f.write("\n")
    return added


def remove_old_occurs_lines(filename: str, input_dir: Path) -> None:
    path = input_dir / filename
    if not path.exists():
        return
    lines = path.read_text(encoding="utf-8").splitlines(True)
    path.write_text("".join(line for line in lines if not line.strip().startswith("occurs(")), encoding="utf-8")


def add_occurs_from_expl(expls: List[Tuple[str, int]], scenario_planning_file: str, input_dir: Path) -> None:
    with (input_dir / scenario_planning_file).open("a", encoding="utf-8") as f:
        f.write("\n")
        for action, t in expls:
            f.write(f"occurs({action}, {t}).\n")
        f.write("\n")


def write_diagnosis_summary(step: int, clingo_output: str, filename: str, output_dir: Path) -> List[Tuple[str, int]]:
    expls = parse_explanations_v2(clingo_output)
    with (output_dir / filename).open("w", encoding="utf-8") as f:
        if not expls:
            f.write("Observations match the agent's expectations.\n")
        else:
            f.write("There were unexpected observations. The agent assumes that the following non-agent actions occurred:\n")
            for action, t in expls:
                f.write(f"  - {action} at time {t}\n")
    return expls


def write_planning_summary(step: int, clingo_output: str, filename: str, output_dir: Path) -> Tuple[int, List[Tuple[str, int]], Optional[int], Optional[int]]:
    answer = extract_last_answer_line(clingo_output)
    path = output_dir / filename
    if answer is None:
        path.write_text("No answer set found.\n", encoding="utf-8")
        return 0, [], None, None
    actions = parse_agent_occurs_from_answer(answer)
    cum_time, cum_pen = parse_metrics_from_answer(answer)
    with path.open("w", encoding="utf-8") as f:
        f.write("Answer Set:\n")
        f.write(answer + "\n\n")
        f.write("Actions taken: ")
        f.write(" ".join(f"agent_occurs({a},{t})" for a, t in actions) if actions else "None")
        f.write("\n\n")
        f.write(f"Cumulative time: {cum_time if cum_time is not None else 'N/A'}\n\n")
        f.write(f"Cumulative penalty: {cum_pen if cum_pen is not None else 'N/A'}\n")
    return len(actions), actions, cum_time, cum_pen


def write_wait_plan_output(step: int, loc: int, output_dir: Path) -> Path:
    """Important: must be agent_occurs(...), not hpd(...), because next iteration parses agent_occurs."""
    path = output_dir / f"output_step_{step}_planning.txt"
    path.write_text(f"Answer Set:\nagent_occurs(stop({loc}),{step}).\n", encoding="utf-8")
    return path


def write_raw_log(log_dir: Path, step: int, kind: str, stdout: str, stderr: str) -> None:
    log_path = log_dir / f"step_{step}_{kind}_raw_clingo.txt"
    log_path.write_text(
        "STDOUT:\n" + stdout + "\n\nSTDERR:\n" + stderr + "\n",
        encoding="utf-8",
    )
# ----------------------------------------------------------------------
# GUI widgets
# ----------------------------------------------------------------------

class Card(QFrame):
    def __init__(self, title: str):
        super().__init__()
        self.setObjectName("Card")
        self.layout = QVBoxLayout(self)
        self.layout.setContentsMargins(14, 12, 14, 14)
        self.layout.setSpacing(8)
        label = QLabel(title)
        label.setObjectName("CardTitle")
        self.layout.addWidget(label)


class ObservationBuilder(QWidget):
    def __init__(self):
        super().__init__()
        self.kind = QComboBox()
        self.kind.addItems(["at_loc(L)", "school_bus_is_stopped(L1,L2)", "pedestrians_are_crossing(L)", "light_color(C,L1,L2)"])
        self.p1 = QLineEdit()
        self.p2 = QLineEdit()
        self.p3 = QLineEdit()
        self.truth = QComboBox()
        self.truth.addItems(["true", "false"])
        self.add_btn = QPushButton("+ Add observation")
        self.remove_btn = QPushButton("Remove selected")
        self.clear_btn = QPushButton("Clear")
        self.table = QTableWidget(0, 2)
        self.table.setHorizontalHeaderLabels(["Fluent", "Truth"])
        self.table.horizontalHeader().setSectionResizeMode(0, QHeaderView.Stretch)
        self.table.horizontalHeader().setSectionResizeMode(1, QHeaderView.ResizeToContents)
        self.table.setSelectionBehavior(QTableWidget.SelectRows)
        self.table.setAlternatingRowColors(True)
        self.table.setMinimumHeight(160)

        grid = QGridLayout()
        grid.addWidget(QLabel("Fluent"), 0, 0)
        grid.addWidget(QLabel("Param 1"), 0, 1)
        grid.addWidget(QLabel("Param 2"), 0, 2)
        grid.addWidget(QLabel("Param 3"), 0, 3)
        grid.addWidget(QLabel("Truth"), 0, 4)
        grid.addWidget(self.kind, 1, 0)
        grid.addWidget(self.p1, 1, 1)
        grid.addWidget(self.p2, 1, 2)
        grid.addWidget(self.p3, 1, 3)
        grid.addWidget(self.truth, 1, 4)
        grid.addWidget(self.add_btn, 1, 5)
        btns = QHBoxLayout()
        btns.addWidget(self.remove_btn)
        btns.addWidget(self.clear_btn)
        btns.addStretch(1)
        layout = QVBoxLayout(self)
        layout.addLayout(grid)
        layout.addWidget(self.table)
        layout.addLayout(btns)

        self.kind.currentIndexChanged.connect(self.update_placeholders)
        self.add_btn.clicked.connect(self.add_observation)
        self.remove_btn.clicked.connect(self.remove_selected)
        self.clear_btn.clicked.connect(self.clear)
        self.update_placeholders()

    def update_placeholders(self):
        kind = self.kind.currentText()
        self.p1.clear(); self.p2.clear(); self.p3.clear()
        self.p3.setEnabled(False)
        if kind.startswith("at_loc"):
            self.p1.setPlaceholderText("L, e.g. 5")
            self.p2.setPlaceholderText("unused")
            self.p2.setEnabled(False)
        elif kind.startswith("pedestrians"):
            self.p1.setPlaceholderText("L, e.g. 7")
            self.p2.setPlaceholderText("unused")
            self.p2.setEnabled(False)
        elif kind.startswith("school_bus"):
            self.p1.setPlaceholderText("L1, e.g. 8")
            self.p2.setPlaceholderText("L2, e.g. 7")
            self.p2.setEnabled(True)
        else:
            self.p1.setPlaceholderText("Color: red/yellow/green")
            self.p2.setPlaceholderText("L1, e.g. 5")
            self.p3.setPlaceholderText("L2, e.g. 6")
            self.p2.setEnabled(True)
            self.p3.setEnabled(True)

    def build_fluent(self) -> Optional[str]:
        kind = self.kind.currentText()
        p1, p2, p3 = self.p1.text().strip(), self.p2.text().strip(), self.p3.text().strip()
        if kind.startswith("at_loc"):
            if not p1.isdigit():
                self.warn("L must be an integer."); return None
            return f"at_loc({p1})"
        if kind.startswith("pedestrians"):
            if not p1.isdigit():
                self.warn("L must be an integer."); return None
            return f"pedestrians_are_crossing({p1})"
        if kind.startswith("school_bus"):
            if not p1.isdigit() or not p2.isdigit():
                self.warn("L1 and L2 must be integers."); return None
            return f"school_bus_is_stopped({p1}, {p2})"
        color = p1.lower()
        if color not in {"red", "yellow", "green"}:
            self.warn("Color must be red, yellow, or green."); return None
        if not p2.isdigit() or not p3.isdigit():
            self.warn("L1 and L2 must be integers."); return None
        return f"light_color({color},{p2},{p3})"

    def warn(self, msg: str):
        QMessageBox.warning(self, "Invalid observation", msg)

    def add_observation(self):
        fluent = self.build_fluent()
        if not fluent:
            return
        row = self.table.rowCount()
        self.table.insertRow(row)
        self.table.setItem(row, 0, QTableWidgetItem(fluent))
        self.table.setItem(row, 1, QTableWidgetItem(self.truth.currentText()))
        self.p1.clear(); self.p2.clear(); self.p3.clear()

    def remove_selected(self):
        rows = sorted({idx.row() for idx in self.table.selectedIndexes()}, reverse=True)
        for row in rows:
            self.table.removeRow(row)

    def clear(self):
        self.table.setRowCount(0)

    def reset_inputs(self):
        """Clear both the observation table and the small parameter fields."""
        self.table.setRowCount(0)
        self.kind.setCurrentIndex(0)
        self.truth.setCurrentIndex(0)
        self.p1.clear()
        self.p2.clear()
        self.p3.clear()
        self.update_placeholders()

    def observations(self) -> List[Tuple[str, bool]]:
        out = []
        for row in range(self.table.rowCount()):
            fluent_item = self.table.item(row, 0)
            truth_item = self.table.item(row, 1)
            if fluent_item and truth_item:
                out.append((fluent_item.text().strip(), truth_item.text().strip().lower() == "true"))
        return out



# ----------------------------------------------------------------------
# Background worker helpers: keep the PyQt event loop responsive while
# clingo runs. This prevents Windows from showing "Not Responding".
# ----------------------------------------------------------------------

class WorkerSignals(QObject):
    finished = pyqtSignal(object)
    error = pyqtSignal(str)


class FunctionWorker(QObject):
    def __init__(self, fn, *args, **kwargs):
        super().__init__()
        self.fn = fn
        self.args = args
        self.kwargs = kwargs
        self.signals = WorkerSignals()

    def run(self):
        try:
            result = self.fn(*self.args, **self.kwargs)
            self.signals.finished.emit(result)
        except Exception as exc:
            self.signals.error.emit(str(exc))



def run_step0_backend(start: int, goal: int, mode: str, observations: List[Tuple[str, bool]]) -> dict:
    run_dir, input_dir, output_dir, log_dir = create_run_directories(start, goal, mode)
    scenario = "scenario_step_0_planning.txt"
    scenario_path = create_step0_planning_file(start, goal, mode, observations, scenario, input_dir)
    scenario_text = scenario_path.read_text(encoding="utf-8")

    stdout, stderr, rc = run_planning(scenario_path)
    write_raw_log(log_dir, 0, "planning", stdout, stderr)

    result = {
        "step": 0,
        "run_dir": str(run_dir),
        "input_dir": str(input_dir),
        "output_dir": str(output_dir),
        "log_dir": str(log_dir),
        "scenario_text": scenario_text,
        "stdout": stdout,
        "stderr": stderr,
        "rc": rc,
        "unsat": is_unsat(stdout),
        "state": {},
        "status": "",
        "output_text": "",
    }

    if is_unsat(stdout):
        loc, find_stdout, find_stderr = run_find_loc(scenario_path)
        write_raw_log(log_dir, 0, "find_loc", find_stdout, find_stderr)
        if loc is None:
            result["status"] = f"UNSAT at step 0, but find_loc could not determine at_loc(L). Run folder: {run_dir}"
            result["output_text"] = "UNSAT, but find_loc could not determine current at_loc(L)."
            result["state"] = {"prev_m": None, "plan_length": None, "cum_time": None, "cum_pen": None}
            return result
        output_path = write_wait_plan_output(0, loc, output_dir)
        result["status"] = f"UNSAT at step 0. Agent waits at location {loc}. Run folder created."
        result["output_text"] = output_path.read_text(encoding="utf-8")
        result["state"] = {"prev_m": 0, "plan_length": 0, "cum_time": None, "cum_pen": None}
        return result

    l, actions, cum_time, cum_pen = write_planning_summary(0, stdout, "output_step_0_planning.txt", output_dir)
    result["output_text"] = (output_dir / "output_step_0_planning.txt").read_text(encoding="utf-8")
    if l == 0:
        result["status"] = f"SAT, but no agent_occurs atoms were found. Run folder: {run_dir}"
        result["state"] = {"prev_m": None, "plan_length": None, "cum_time": None, "cum_pen": None}
    else:
        result["status"] = f"SAT. Plan length={l}, cumulative_time={cum_time}, cumulative_penalty={cum_pen}. Run folder created."
        result["state"] = {"prev_m": 0, "plan_length": l, "cum_time": cum_time, "cum_pen": cum_pen}
    return result


def run_stepM_backend(
    run_dir_s: str,
    input_dir_s: str,
    output_dir_s: str,
    log_dir_s: str,
    prev_m: int,
    plan_length: int,
    M: int,
    observations: List[Tuple[str, bool]],
) -> dict:
    run_dir = Path(run_dir_s)
    input_dir = Path(input_dir_s)
    output_dir = Path(output_dir_s)
    log_dir = Path(log_dir_s)

    prev_scenario = f"scenario_step_{prev_m}_planning.txt"
    if not (input_dir / prev_scenario).exists():
        raise FileNotFoundError(
            f"Could not find {input_dir / prev_scenario}. The workflow state and files are out of sync."
        )

    diag_file = f"scenario_step_{M}_diagnosis.txt"
    plan_file = f"scenario_step_{M}_planning.txt"

    copy_and_update_m(prev_scenario, diag_file, prev_m, M, input_dir)
    add_observations_to_file(diag_file, observations, M, input_dir)
    add_hpd_from_previous_plan(prev_m, diag_file, M, input_dir, output_dir)
    diag_scenario_text = (input_dir / diag_file).read_text(encoding="utf-8")

    diag_path = input_dir / diag_file
    stdout_diag, stderr_diag, rc_diag = run_diagnosis(diag_path)
    write_raw_log(log_dir, M, "diagnosis", stdout_diag, stderr_diag)
    expls = write_diagnosis_summary(M, stdout_diag, f"output_step_{M}_diagnosis.txt", output_dir)
    diag_output_text = (output_dir / f"output_step_{M}_diagnosis.txt").read_text(encoding="utf-8")

    shutil.copyfile(input_dir / diag_file, input_dir / plan_file)
    remove_old_occurs_lines(plan_file, input_dir)
    if expls:
        add_occurs_from_expl(expls, plan_file, input_dir)
    plan_scenario_text = (input_dir / plan_file).read_text(encoding="utf-8")

    plan_path = input_dir / plan_file
    stdout_plan, stderr_plan, rc_plan = run_planning(plan_path)
    write_raw_log(log_dir, M, "planning", stdout_plan, stderr_plan)
    result = {
        "step": M,
        "run_dir": str(run_dir),
        "input_dir": str(input_dir),
        "output_dir": str(output_dir),
        "log_dir": str(log_dir),
        "diag_scenario_text": diag_scenario_text,
        "diag_output_text": diag_output_text,
        "plan_scenario_text": plan_scenario_text,
        "stdout_diag": stdout_diag,
        "stderr_diag": stderr_diag,
        "rc_diag": rc_diag,
        "stdout_plan": stdout_plan,
        "stderr_plan": stderr_plan,
        "rc_plan": rc_plan,
        "unsat": is_unsat(stdout_plan),
        "status": "",
        "plan_output_text": "",
        "state": {},
    }

    if is_unsat(stdout_plan):
        loc, find_stdout, find_stderr = run_find_loc(plan_path)
        write_raw_log(log_dir, M, "find_loc", find_stdout, find_stderr)
        if loc is None:
            out_path = output_dir / f"output_step_{M}_planning.txt"
            out_path.write_text("UNSAT and no at_loc(L) found in find_loc output.\n", encoding="utf-8")
            result["plan_output_text"] = out_path.read_text(encoding="utf-8")
            result["status"] = f"UNSAT at step {M}, but current location could not be found."
            result["state"] = {"prev_m": prev_m, "plan_length": plan_length, "cum_time": None, "cum_pen": None}
            return result
        output_path = write_wait_plan_output(M, loc, output_dir)
        result["plan_output_text"] = output_path.read_text(encoding="utf-8")
        result["status"] = f"UNSAT at step {M}. Agent waits with agent_occurs(stop({loc}),{M}). You may continue if M < l."
        result["state"] = {"prev_m": M, "plan_length": plan_length, "cum_time": None, "cum_pen": None}
        return result

    l, actions, cum_time, cum_pen = write_planning_summary(M, stdout_plan, f"output_step_{M}_planning.txt", output_dir)
    result["plan_output_text"] = (output_dir / f"output_step_{M}_planning.txt").read_text(encoding="utf-8")
    if l == 0:
        result["status"] = f"SAT at step {M}, but no agent_occurs atoms were found."
        result["state"] = {"prev_m": prev_m, "plan_length": plan_length, "cum_time": None, "cum_pen": None}
    else:
        result["status"] = f"SAT at step {M}. New plan length={l}, cumulative_time={cum_time}, cumulative_penalty={cum_pen}."
        result["state"] = {"prev_m": M, "plan_length": l, "cum_time": cum_time, "cum_pen": cum_pen}
    return result

class MainWindow(QMainWindow):
    def __init__(self):
        super().__init__()
        self.state = PlannerState()
        self._thread = None
        self._worker = None
        self.setWindowTitle("ASP Planning & Diagnosis Interface - v13 - Scenario Sessions")
        self.resize(1280, 820)
        self.build_ui()
        self.apply_style()
        self.refresh_state()

    def build_ui(self):
        root = QWidget()
        main = QVBoxLayout(root)
        main.setContentsMargins(14, 14, 14, 14)

        top = QHBoxLayout()
        top.setSpacing(8)
        title = QLabel("ASP Planning & Diagnosis")
        title.setObjectName("Title")

        # Do not print the full work-directory path in the header.
        # Long OneDrive paths force Qt to create a huge minimum window width,
        # which causes QWindowsWindow::setGeometry warnings and hides buttons.
        self.run_dir_label = QLabel("Active run: none")
        self.run_dir_label.setObjectName("Muted")
        self.run_dir_label.setMaximumWidth(360)
        self.run_dir_label.setToolTip("The active scenario folder name. Use the buttons to view full paths.")

        choose_btn = QPushButton("Change Work Folder")
        choose_btn.clicked.connect(self.choose_workdir)
        self.show_workdir_btn = QPushButton("Show Work Folder")
        self.show_workdir_btn.clicked.connect(self.show_work_folder_path)
        self.new_scenario_btn = QPushButton("New Scenario / Reset")
        self.new_scenario_btn.clicked.connect(self.new_scenario_reset)
        self.open_run_btn = QPushButton("Open Active Run Folder")
        self.open_run_btn.clicked.connect(self.open_active_run_folder)
        self.show_run_path_btn = QPushButton("Show Active Run Path")
        self.show_run_path_btn.clicked.connect(self.show_active_run_path)

        top.addWidget(title)
        top.addStretch(1)
        top.addWidget(self.run_dir_label)
        top.addWidget(choose_btn)
        top.addWidget(self.show_workdir_btn)
        top.addWidget(self.new_scenario_btn)
        top.addWidget(self.open_run_btn)
        top.addWidget(self.show_run_path_btn)
        main.addLayout(top)

        self.tabs = QTabWidget()
        self.tab0 = QWidget()
        self.tabM = QWidget()
        self.tabs.addTab(self.tab0, "1  Step 0 Planning")
        self.tabs.addTab(self.tabM, "2  Diagnosis + Replanning")
        main.addWidget(self.tabs)
        self.setCentralWidget(root)
        self.build_step0_tab()
        self.build_loop_tab()

    def build_step0_tab(self):
        layout = QHBoxLayout(self.tab0)
        left = QVBoxLayout(); right = QVBoxLayout()
        layout.addLayout(left, 3); layout.addLayout(right, 5)

        inputs = Card("Initial setup")
        grid = QGridLayout()
        self.start_edit = QLineEdit(); self.start_edit.setPlaceholderText("Example: 5")
        self.goal_edit = QLineEdit(); self.goal_edit.setPlaceholderText("Example: 10")
        self.mode_combo = QComboBox(); self.mode_combo.addItems(["emergency", "non_emergency"])
        grid.addWidget(QLabel("Start location"), 0, 0); grid.addWidget(self.start_edit, 0, 1)
        grid.addWidget(QLabel("Goal location"), 1, 0); grid.addWidget(self.goal_edit, 1, 1)
        grid.addWidget(QLabel("Mode"), 2, 0); grid.addWidget(self.mode_combo, 2, 1)
        inputs.layout.addLayout(grid)
        left.addWidget(inputs)

        obs = Card("Extra observations at time 0")
        note = QLabel("The GUI automatically adds obs(at_loc(start), true, 0) and holds(at_loc(start), 0).")
        note.setObjectName("Muted")
        obs.layout.addWidget(note)
        self.obs0 = ObservationBuilder()
        obs.layout.addWidget(self.obs0)
        left.addWidget(obs)

        self.run0_btn = QPushButton("Start New Scenario + Run Step 0 Planning")
        self.run0_btn.setObjectName("Primary")
        self.run0_btn.clicked.connect(self.run_step0)
        left.addWidget(self.run0_btn)
        left.addStretch(1)

        self.status0 = QLabel("Ready")
        self.status0.setObjectName("Status")
        right.addWidget(self.status0)
        self.scenario0_view = self.make_text_view("scenario_step_0_planning.txt")
        self.output0_view = self.make_text_view("output_step_0_planning.txt")
        right.addWidget(QLabel("Scenario file")); right.addWidget(self.scenario0_view, 3)
        right.addWidget(QLabel("Planning summary")); right.addWidget(self.output0_view, 2)

    def build_loop_tab(self):
        layout = QHBoxLayout(self.tabM)
        left = QVBoxLayout(); right = QVBoxLayout()
        layout.addLayout(left, 3); layout.addLayout(right, 5)

        state_card = Card("Current workflow state")
        self.prev_label = QLabel("-"); self.length_label = QLabel("-"); self.time_label = QLabel("-"); self.pen_label = QLabel("-"); self.active_run_state_label = QLabel("-")
        grid = QGridLayout()
        for r, (name, widget) in enumerate([
            ("Previous m", self.prev_label), ("Current plan length l", self.length_label),
            ("Cumulative time", self.time_label), ("Cumulative penalty", self.pen_label),
            ("Active scenario folder", self.active_run_state_label)
        ]):
            grid.addWidget(QLabel(name), r, 0); grid.addWidget(widget, r, 1)
        state_card.layout.addLayout(grid)
        left.addWidget(state_card)

        step_card = Card("New observation step")
        self.spinM = QSpinBox(); self.spinM.setMinimum(1); self.spinM.setMaximum(9999)
        step_card.layout.addWidget(QLabel("Choose M such that prev_m < M <= l"))
        step_card.layout.addWidget(self.spinM)
        left.addWidget(step_card)

        obs_card = Card("Observations at time M")
        self.obsM = ObservationBuilder()
        obs_card.layout.addWidget(self.obsM)
        left.addWidget(obs_card)

        self.runM_btn = QPushButton("Run Diagnosis + Planning for Step M")
        self.runM_btn.setObjectName("Primary")
        self.runM_btn.clicked.connect(self.run_stepM)
        left.addWidget(self.runM_btn)
        left.addStretch(1)

        self.statusM = QLabel("Run Step 0 first.")
        self.statusM.setObjectName("Status")
        right.addWidget(self.statusM)
        self.diag_scenario_view = self.make_text_view("scenario_step_M_diagnosis.txt")
        self.plan_scenario_view = self.make_text_view("scenario_step_M_planning.txt")
        self.diag_output_view = self.make_text_view("output_step_M_diagnosis.txt")
        self.plan_output_view = self.make_text_view("output_step_M_planning.txt")
        right.addWidget(QLabel("Diagnosis scenario")); right.addWidget(self.diag_scenario_view, 2)
        right.addWidget(QLabel("Planning scenario")); right.addWidget(self.plan_scenario_view, 2)
        right.addWidget(QLabel("Diagnosis summary")); right.addWidget(self.diag_output_view, 1)
        right.addWidget(QLabel("Planning summary")); right.addWidget(self.plan_output_view, 1)

    def make_text_view(self, placeholder: str) -> QTextEdit:
        view = QTextEdit()
        view.setReadOnly(True)
        view.setPlaceholderText(placeholder)
        view.setFont(QFont("Consolas", 10))
        return view

    def choose_workdir(self):
        global WORK_DIR
        folder = QFileDialog.getExistingDirectory(self, "Select folder containing ASP files", str(WORK_DIR.resolve()))
        if folder:
            WORK_DIR = Path(folder)
            self.state = PlannerState()
            self.clear_all_user_inputs()
            self.refresh_state()
            self.clear_outputs()
            self.status0.setText("Ready. Start a new Step 0 scenario.")

    def clear_all_user_inputs(self):
        """Reset the form fields for a completely fresh scenario.

        This intentionally keeps the selected WORK_DIR unchanged, but clears
        everything that belongs to the previous scenario: start/goal values,
        mode selection, observation tables, parameter entry boxes, and Step M.
        """
        self.start_edit.clear()
        self.goal_edit.clear()
        self.mode_combo.setCurrentIndex(0)
        self.obs0.reset_inputs()
        self.obsM.reset_inputs()
        self.spinM.setMinimum(1)
        self.spinM.setMaximum(9999)
        self.spinM.setValue(1)

    def new_scenario_reset(self):
        self.state = PlannerState()
        self.clear_all_user_inputs()
        self.clear_outputs()
        self.tabs.setCurrentIndex(0)
        self.status0.setText("Ready. Enter a new start/goal and run Step 0 to create a new scenario folder.")
        self.statusM.setText("Run Step 0 first.")
        self.refresh_state()

    def show_work_folder_path(self):
        QMessageBox.information(self, "Current work folder", str(WORK_DIR.resolve()))

    def show_active_run_path(self):
        if not self.state.active_run_dir:
            QMessageBox.information(self, "No active run", "There is no active scenario folder yet. Run Step 0 first.")
            return
        QMessageBox.information(self, "Active run folder", str(self.state.active_run_dir.resolve()))

    def open_active_run_folder(self):
        if not self.state.active_run_dir:
            QMessageBox.information(self, "No active run", "There is no active scenario folder yet. Run Step 0 first.")
            return
        folder = self.state.active_run_dir.resolve()
        try:
            if sys.platform.startswith("win"):
                subprocess.Popen(["explorer", str(folder)])
            elif sys.platform == "darwin":
                subprocess.Popen(["open", str(folder)])
            else:
                subprocess.Popen(["xdg-open", str(folder)])
        except Exception as exc:
            QMessageBox.warning(self, "Could not open folder", f"Folder path:\n{folder}\n\nError:\n{exc}")

    def clear_outputs(self):
        for view in [self.scenario0_view, self.output0_view, self.diag_scenario_view, self.plan_scenario_view, self.diag_output_view, self.plan_output_view]:
            view.clear()

    def refresh_state(self):
        s = self.state
        self.prev_label.setText("-" if s.prev_m is None else str(s.prev_m))
        self.length_label.setText("-" if s.plan_length is None else str(s.plan_length))
        self.time_label.setText("-" if s.last_cum_time is None else str(s.last_cum_time))
        self.pen_label.setText("-" if s.last_cum_penalty is None else str(s.last_cum_penalty))
        active_run_text = "-" if s.active_run_dir is None else s.active_run_dir.name
        self.active_run_state_label.setText(active_run_text)
        if s.active_run_dir is not None:
            self.active_run_state_label.setToolTip(str(s.active_run_dir.resolve()))
        else:
            self.active_run_state_label.setToolTip("")
        self.run_dir_label.setText("Active run: none" if s.active_run_dir is None else f"Active run: {s.active_run_dir.name}")
        if s.active_run_dir is not None:
            self.run_dir_label.setToolTip(str(s.active_run_dir.resolve()))
        else:
            self.run_dir_label.setToolTip("The active scenario folder name. Use the buttons to view full paths.")
        self.open_run_btn.setEnabled(s.active_run_dir is not None)
        self.show_run_path_btn.setEnabled(s.active_run_dir is not None)
        enabled = s.prev_m is not None and s.plan_length is not None and s.input_dir is not None and s.output_dir is not None
        self.tabs.setTabEnabled(1, enabled)
        self.runM_btn.setEnabled(enabled)
        if enabled:
            self.spinM.setMinimum(s.prev_m + 1)
            self.spinM.setMaximum(max(s.prev_m + 1, s.plan_length))
            self.spinM.setValue(min(max(s.prev_m + 1, self.spinM.minimum()), self.spinM.maximum()))
        self.statusM.setText(s.last_status if enabled else "Run Step 0 first.")

    def show_missing_files(self) -> bool:
        missing = require_files()
        if missing:
            QMessageBox.warning(self, "Missing ASP files", "These files were not found in the selected work folder:\n\n" + "\n".join(missing))
            return True
        return False

    def run_step0(self):
        if self.show_missing_files():
            return
        if not self.start_edit.text().strip().isdigit() or not self.goal_edit.text().strip().isdigit():
            QMessageBox.warning(self, "Invalid input", "Start and goal locations must be integers.")
            return

        start = int(self.start_edit.text().strip())
        goal = int(self.goal_edit.text().strip())
        obs = [(f"at_loc({start})", True), *self.obs0.observations()]

        self.run0_btn.setEnabled(False)
        self.status0.setText("Creating a new scenario folder and running Step 0 planning...")
        QApplication.processEvents()
        self.start_background_task(
            run_step0_backend,
            self.on_step0_finished,
            start,
            goal,
            self.mode_combo.currentText(),
            obs,
        )

    def on_step0_finished(self, result: dict):
        self.run0_btn.setEnabled(True)
        self.scenario0_view.setPlainText(result.get("scenario_text", ""))
        self.output0_view.setPlainText(result.get("output_text", ""))
        self.status0.setText(result.get("status", "Finished."))

        st = result.get("state", {})
        # Store the active scenario-session folders even when Step 0 is UNSAT,
        # so the user can inspect the generated files and raw clingo logs.
        self.state.active_run_dir = Path(result["run_dir"])
        self.state.input_dir = Path(result["input_dir"])
        self.state.output_dir = Path(result["output_dir"])
        self.state.log_dir = Path(result["log_dir"])

        if st.get("prev_m") is None:
            self.state.prev_m = None
            self.state.plan_length = None
            self.state.last_cum_time = None
            self.state.last_cum_penalty = None
            self.state.last_status = result.get("status", "Ready.")
        else:
            self.state.prev_m = st.get("prev_m")
            self.state.plan_length = st.get("plan_length")
            self.state.last_cum_time = st.get("cum_time")
            self.state.last_cum_penalty = st.get("cum_pen")
            self.state.last_status = "Ready for next step." if self.state.plan_length else result.get("status", "Ready.")
            self.tabs.setCurrentIndex(1)
        self.refresh_state()

    def start_background_task(self, fn, finished_callback, *args):
        self._thread = QThread(self)
        self._worker = FunctionWorker(fn, *args)
        self._worker.moveToThread(self._thread)
        self._thread.started.connect(self._worker.run)
        self._worker.signals.finished.connect(finished_callback)
        self._worker.signals.finished.connect(self._thread.quit)
        self._worker.signals.error.connect(self.on_worker_error)
        self._worker.signals.error.connect(self._thread.quit)
        self._thread.finished.connect(self._worker.deleteLater)
        self._thread.finished.connect(self._thread.deleteLater)
        self._thread.finished.connect(self._clear_worker_refs)
        self._thread.start()

    def _clear_worker_refs(self):
        self._thread = None
        self._worker = None

    def on_worker_error(self, message: str):
        self.run0_btn.setEnabled(True)
        self.runM_btn.setEnabled(self.state.prev_m is not None and self.state.plan_length is not None and self.state.input_dir is not None)
        self.status0.setText("Error while running clingo.")
        self.statusM.setText("Error while running clingo.")
        QMessageBox.critical(self, "Runtime error", message)

    def run_stepM(self):
        if self.show_missing_files():
            return
        s = self.state
        if s.prev_m is None or s.plan_length is None or s.input_dir is None or s.output_dir is None or s.log_dir is None or s.active_run_dir is None:
            QMessageBox.warning(self, "No active scenario", "Run Step 0 planning successfully first so the GUI can create an active scenario folder.")
            return
        M = self.spinM.value()
        if not (M > s.prev_m and M <= s.plan_length):
            QMessageBox.warning(self, "Invalid M", f"M must satisfy: {s.prev_m} < M <= {s.plan_length}.")
            return

        self.runM_btn.setEnabled(False)
        self.statusM.setText(f"Running diagnosis + planning for step {M} inside the active scenario folder...")
        QApplication.processEvents()
        self.start_background_task(
            run_stepM_backend,
            self.on_stepM_finished,
            str(s.active_run_dir),
            str(s.input_dir),
            str(s.output_dir),
            str(s.log_dir),
            s.prev_m,
            s.plan_length,
            M,
            self.obsM.observations(),
        )

    def on_stepM_finished(self, result: dict):
        self.diag_scenario_view.setPlainText(result.get("diag_scenario_text", ""))
        self.diag_output_view.setPlainText(result.get("diag_output_text", ""))
        self.plan_scenario_view.setPlainText(result.get("plan_scenario_text", ""))
        self.plan_output_view.setPlainText(result.get("plan_output_text", ""))

        st = result.get("state", {})
        if st:
            self.state.active_run_dir = Path(result["run_dir"])
            self.state.input_dir = Path(result["input_dir"])
            self.state.output_dir = Path(result["output_dir"])
            self.state.log_dir = Path(result["log_dir"])
            self.state.prev_m = st.get("prev_m")
            self.state.plan_length = st.get("plan_length")
            self.state.last_cum_time = st.get("cum_time")
            self.state.last_cum_penalty = st.get("cum_pen")
            self.state.last_status = result.get("status", "Finished.")
        self.statusM.setText(result.get("status", "Finished."))
        self.obsM.clear()
        self.refresh_state()

    def apply_style(self):
        self.setStyleSheet("""
            QWidget { font-family: Segoe UI, Arial; font-size: 10.5pt; }
            #Title { font-size: 20pt; font-weight: 700; color: #1f2937; }
            #Muted { color: #6b7280; }
            #Status { font-weight: 700; color: #111827; padding: 8px; background: #eef2ff; border: 1px solid #c7d2fe; border-radius: 8px; }
            #Card { background: #ffffff; border: 1px solid #d1d5db; border-radius: 12px; }
            #CardTitle { font-size: 13pt; font-weight: 700; color: #111827; }
            QPushButton { padding: 7px 12px; border-radius: 8px; border: 1px solid #cbd5e1; background: #f8fafc; }
            QPushButton:hover { background: #eef2ff; }
            QPushButton#Primary { background: #2563eb; color: white; border: 1px solid #1d4ed8; font-weight: 700; padding: 10px 14px; }
            QPushButton#Primary:hover { background: #1d4ed8; }
            QLineEdit, QComboBox, QSpinBox, QTextEdit { border: 1px solid #cbd5e1; border-radius: 6px; padding: 5px; background: white; }
            QTableWidget { border: 1px solid #cbd5e1; border-radius: 6px; gridline-color: #e5e7eb; }
            QHeaderView::section { background: #f3f4f6; padding: 6px; border: 0px; font-weight: 700; }
            QTabWidget::pane { border: 1px solid #d1d5db; border-radius: 10px; background: #f9fafb; }
            QTabBar::tab { padding: 9px 16px; margin-right: 4px; border-top-left-radius: 8px; border-top-right-radius: 8px; background: #e5e7eb; }
            QTabBar::tab:selected { background: #ffffff; font-weight: 700; }
        """)


def main():
    app = QApplication(sys.argv)
    window = MainWindow()
    window.show()
    sys.exit(app.exec_())


if __name__ == "__main__":
    main()
