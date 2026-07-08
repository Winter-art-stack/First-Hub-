"""
The First Tool By Night Kanz Nary ( The Newbie in Coding Global 😘)
=================

Download:
    pip install PyQt6 requests psutil

Run:
    python OnlyNight.py
"""

import sys, os, json, subprocess, tempfile, shutil, socket, random, math
import psutil
import requests

from PyQt6.QtWidgets import (
    QApplication, QWidget, QVBoxLayout, QHBoxLayout,
    QPushButton, QLabel, QComboBox, QLineEdit, QFrame,
    QPlainTextEdit, QStackedWidget, QProgressBar, QSizePolicy, QScrollArea
)
from PyQt6.QtCore import Qt, pyqtSignal, QObject, QThread, QTimer, QProcess, QRect, QPointF
from PyQt6.QtGui import QFont, QPainter, QColor, QPen, QBrush, QPainterPath

IS_WIN = os.name == "nt"

# ─────────────────────────────────────────────
#  CONFIG
# ─────────────────────────────────────────────
def cfg_path():
    base = os.environ.get("APPDATA", os.path.expanduser("~")) if IS_WIN else os.path.expanduser("~/.config")
    d = os.path.join(base, "")
    os.makedirs(d, exist_ok=True)
    return os.path.join(d, "config.json")

CFG_FILE = cfg_path()
DEFAULTS = {"claude_key": "", "model": "free", "language": "python", "effect": "none"}

def load_cfg():
    try:
        with open(CFG_FILE, encoding="utf-8") as f:
            c = json.load(f); m = DEFAULTS.copy(); m.update(c); return m
    except Exception:
        return DEFAULTS.copy()

def save_cfg(c):
    try:
        with open(CFG_FILE, "w", encoding="utf-8") as f: json.dump(c, f, indent=2)
        return True
    except Exception:
        return False

# ─────────────────────────────────────────────
#  AI WORKER
# ─────────────────────────────────────────────
class AIWorker(QObject):
    done   = pyqtSignal(str)
    fail   = pyqtSignal(str)

    def __init__(self, prompt, lang, model, cfg, mode="code"):
        super().__init__()
        self.prompt, self.lang, self.model, self.cfg, self.mode = prompt, lang, model, cfg, mode

    def run(self):
        try:
            sys_p, usr_p = self._build_prompts()
            if self.model == "claude":
                result = self._claude(sys_p, usr_p)
            elif self.model == "deepseek":
                result = self._deepseek(sys_p, usr_p)
            elif self.model == "chatgpt":
                result = self._chatgpt(sys_p, usr_p)
            else:
                result = self._free(sys_p, usr_p)
            self.done.emit(result)
        except Exception as e:
            self.fail.emit(str(e))

    def _build_prompts(self):
        if self.mode == "check":
            s = (f"You are a senior {self.lang} code reviewer. "
                 "Analyze the following code: syntax errors, logic errors, edge cases, and performance. "
                 "Please reply in English, clearly stating each point, including the solution and the code snippet for the fix.")
            u = f"```{self.lang}\n{self.prompt}\n```"
        elif self.mode == "chat":
            s = "You are the friendly AI assistant integrated into Night Kanz Nary's first product. Your responses in Vietnamese are concise, clear, and helpful."
            u = self.prompt
        else:
            s = (f"You are an expert {self.lang} programmer. "
                 "Reply ONLY with code in ONE code block, minimal comments, no explanation outside block.")
            u = f"[{self.lang}] {self.prompt}"
        return s, u

    def _free(self, sys_p, usr_p):
        msgs = [{"role":"system","content":sys_p},{"role":"user","content":usr_p}]
        # Primary free endpoint
        try:
            r = requests.post(
                "https://chateverywhere.app/api/chat/",
                headers={"Content-Type":"application/json",
                         "User-Agent":"Mozilla/5.0 (Windows NT 10.0; Win64; x64)"},
                json={"model":"gpt-5o-mini","messages":msgs}, timeout=25)
            if r.status_code == 200 and r.text.strip():
                return r.text.strip()
        except Exception:
            pass
        # Fallback 1 – DeepInfra (free tier, no key needed for some models)
        try:
            r2 = requests.post(
                "https://api.deepinfra.com/v1/openai/chat/completions",
                json={"model":"meta-llama/Meta-Llama-3-70B-Instruct","messages":msgs}, timeout=30)
            if r2.status_code == 200:
                d = r2.json()
                return d["choices"][0]["message"]["content"]
        except Exception:
            pass
        # Fallback 2 – Pollinations (always free)
        try:
            combined = f"{sys_p}\n\n{usr_p}"
            import urllib.parse
            enc = urllib.parse.quote(combined)
            r3 = requests.get(f"https://text.pollinations.ai/{enc}", timeout=30)
            if r3.status_code == 200:
                return r3.text.strip()
        except Exception:
            pass
        return "[Not Connecting All Server < Not Found 404 > Return now.]"

    def _claude(self, sys_p, usr_p):
        key = self.cfg.get("claude_key","").strip()
        if not key:
            return "[The Claude API Key has not been entered in the Settings.]"
        r = requests.post("https://api.anthropic.com/v1/messages",
            headers={"x-api-key":key,"anthropic-version":"2023-06-01","content-type":"application/json"},
            json={"model":"claude-sonnet-4-6","max_tokens":2048,
                  "system":sys_p,"messages":[{"role":"user","content":usr_p}]}, timeout=60)
        if r.status_code != 200:
            return f"[Error Claude {r.status_code}]: {r.text[:300]}"
        data = r.json()
        return "\n".join(b["text"] for b in data.get("content",[]) if b.get("type")=="text")

    def _deepseek(self, sys_p, usr_p):
        key = self.cfg.get("deepseek_key","").strip()
        if not key:
            return "[The DeepSeek API Key has not been entered in the Settings.]"
        r = requests.post("https://api.deepseek.com/chat/completions",
            headers={"Authorization":f"Bearer {key}","Content-Type":"application/json"},
            json={"model":"deepseek-chat","max_tokens":2048,
                  "messages":[{"role":"system","content":sys_p},{"role":"user","content":usr_p}]},
            timeout=60)
        if r.status_code != 200:
            return f"[Error DeepSeek {r.status_code}]: {r.text[:300]}"
        return r.json()["choices"][0]["message"]["content"]

    def _chatgpt(self, sys_p, usr_p):
        key = self.cfg.get("openai_key","").strip()
        if not key:
            return "[The OpenAI API Key has not been entered in the Settings.]"
        r = requests.post("https://api.openai.com/v1/chat/completions",
            headers={"Authorization":f"Bearer {key}","Content-Type":"application/json"},
            json={"model":"gpt-5o-mini","max_tokens":2048,
                  "messages":[{"role":"system","content":sys_p},{"role":"user","content":usr_p}]},
            timeout=60)
        if r.status_code != 200:
            return f"[Error OpenAI {r.status_code}]: {r.text[:300]}"
        return r.json()["choices"][0]["message"]["content"]

# ─────────────────────────────────────────────
#  PARTICLE EFFECT CANVAS  (inside window)
# ─────────────────────────────────────────────
class Particle:
    def __init__(self, kind, w, h):
        self.kind = kind
        self.reset(w, h, fresh=True)

    def reset(self, w, h, fresh=False):
        self.x = random.uniform(0, w)
        self.y = random.uniform(-h, 0) if not fresh else random.uniform(0, h)
        if self.kind == "snow":
            self.size  = random.uniform(4, 10)
            self.speed = random.uniform(1.5,4)
            self.sway  = random.uniform(-0.5, 0.5)
            self.phase = random.uniform(0, math.pi*2)
            self.alpha = random.uniform(140, 230)
        elif self.kind == "rain":
            self.size  = random.uniform(8, 20)
            self.speed = random.uniform(10, 40)
            self.sway  = random.uniform(-0.3, 0.3)
            self.phase = 0
            self.alpha = random.uniform(80, 160)
        else:  # leaves
            self.size  = random.uniform(8, 18)
            self.speed = random.uniform(1, 3)
            self.sway  = random.uniform(-1.5, 1.5)
            self.phase = random.uniform(0, math.pi*2)
            self.alpha = random.uniform(160, 240)
            self.rot   = random.uniform(0, 360)
            self.rot_s = random.uniform(-3, 3)

    def update(self, w, h):
        self.phase += 0.03
        if self.kind == "snow":
            self.x += math.sin(self.phase) * 0.8 + self.sway
            self.y += self.speed
        elif self.kind == "rain":
            self.x += self.sway
            self.y += self.speed
        else:
            self.x += math.sin(self.phase) * 1.5 + self.sway
            self.y += self.speed
            self.rot += self.rot_s
        if self.y > h + 20 or self.x < -20 or self.x > w + 20:
            self.reset(w, h)

class EffectCanvas(QWidget):
    def __init__(self, parent=None):
        super().__init__(parent)
        self.kind = "none"
        self.particles = []
        self.timer = QTimer(self)
        self.timer.timeout.connect(self._tick)
        self.setAttribute(Qt.WidgetAttribute.WA_TransparentForMouseEvents)
        self.setAttribute(Qt.WidgetAttribute.WA_NoSystemBackground)
        self.setStyleSheet("background: transparent;")

    def set_effect(self, kind):
        self.kind = kind
        self.particles = []
        if kind != "none":
            count = {"snow": 30, "rain": 60, "leaves": 25}[kind]
            w, h = max(self.width(), 800), max(self.height(), 600)
            for _ in range(count):
                self.particles.append(Particle(kind, w, h))
            self.timer.start(14)
        else:
            self.timer.stop()
            self.update()

    def _tick(self):
        w, h = self.width(), self.height()
        for p in self.particles:
            p.update(w, h)
        self.update()

    def paintEvent(self, event):
        if not self.particles:
            return
        painter = QPainter(self)
        LEAF_COLORS = [(180,100,20),(220,140,30),(160,80,10),(200,160,40)]
        for p in self.particles:
            if self.kind == "snow":
                painter.setPen(Qt.PenStyle.NoPen)
                painter.setBrush(QBrush(QColor(200, 220, 255, int(p.alpha))))
                r = int(p.size / 2)
                painter.drawEllipse(int(p.x - r), int(p.y - r), r*2, r*2)
            elif self.kind == "rain":
                painter.setPen(QPen(QColor(150, 190, 255, int(p.alpha)), 1))
                painter.drawLine(int(p.x), int(p.y), int(p.x), int(p.y + p.size))
            else:
                painter.setRenderHint(QPainter.RenderHint.Antialiasing, True)
                painter.save()
                painter.translate(p.x, p.y)
                painter.rotate(p.rot)
                rgb = LEAF_COLORS[int(abs(p.x + p.y)) % len(LEAF_COLORS)]
                painter.setBrush(QBrush(QColor(*rgb, int(p.alpha))))
                painter.setPen(Qt.PenStyle.NoPen)
                s = p.size
                path = QPainterPath()
                path.moveTo(0, -s/2)
                path.cubicTo(s/2, -s/3, s/2, s/3, 0, s/2)
                path.cubicTo(-s/2, s/3, -s/2, -s/3, 0, -s/2)
                painter.drawPath(path)
                painter.restore()
                painter.setRenderHint(QPainter.RenderHint.Antialiasing, False)
        painter.end()
# ─────────────────────────────────────────────
#  CODE RUNNER
# ─────────────────────────────────────────────
class RunnerWidget(QWidget):
    def __init__(self, get_code_cb):
        super().__init__()
        self.get_code = get_code_cb
        self.proc = None
        layout = QVBoxLayout(self)
        layout.setContentsMargins(0,0,0,0)
        layout.setSpacing(6)

        bar = QHBoxLayout()
        self.run_btn = QPushButton("▶ Run")
        self.run_btn.setObjectName("primaryBtn")
        self.run_btn.clicked.connect(self.run_code)
        self.stop_btn = QPushButton("■ Stop")
        self.stop_btn.setObjectName("dangerBtn")
        self.stop_btn.clicked.connect(self.stop_code)
        self.stat = QLabel("Load")
        self.stat.setObjectName("status")
        bar.addWidget(self.run_btn); bar.addWidget(self.stop_btn)
        bar.addWidget(self.stat); bar.addStretch()

        self.term = QPlainTextEdit()
        self.term.setObjectName("panel")OnlyNight.py
        self.term.setFont(QFont("Cascadia Code",10))
        self.term.setPlaceholderText("The Output...")

        self.stdin = QLineEdit()
        self.stdin.setObjectName("lineEdit")
        self.stdin.setPlaceholderText("Send stdin and Enter To send program running...")
        self.stdin.returnPressed.connect(self.send_stdin)

        layout.addLayout(bar)
        layout.addWidget(self.term)
        layout.addWidget(self.stdin)

    def run_code(self):
        self.stop_code()
        info = self.get_code()
        lang = info["language"]; code = info["text"].strip()
        if not code: self.term.appendPlainText("Not Found Code =)."); return
        self.term.clear(); self.stat.setText("Running...")
        td = tempfile.mkdtemp(prefix="wnh_")
        try:
            if lang == "python":
                fp = os.path.join(td,"s.py")
                open(fp,"w",encoding="utf-8").write(code)
                self._start(sys.executable,[fp])
            elif lang == "cpp":
                src = os.path.join(td,"m.cpp"); exe = os.path.join(td,"m.exe" if IS_WIN else "m")
                open(src,"w",encoding="utf-8").write(code)
                gpp = shutil.which("g++")
                if not gpp: self.term.appendPlainText("// Thiếu g++"); return
                cp = subprocess.run([gpp,src,"-o",exe,"-O2"],capture_output=True,text=True)
                if cp.returncode != 0: self.term.appendPlainText("// Lỗi biên dịch:\n"+cp.stderr); return
                self._start(exe,[])
            elif lang in ("lua","luau"):
                fp = os.path.join(td,"s.lua")
                open(fp,"w",encoding="utf-8").write(code)
                interp = shutil.which("luau") or shutil.which("lua")
                if not interp: self.term.appendPlainText("// Thiếu lua/luau interpreter"); return
                self._start(interp,[fp])
        except Exception as e:
            self.term.appendPlainText(f"// Error: {e}")

    def _start(self, prog, args):
        self.proc = QProcess(self)
        self.proc.setProcessChannelMode(QProcess.ProcessChannelMode.MergedChannels)
        self.proc.readyReadStandardOutput.connect(self._out)
        self.proc.finished.connect(lambda: self.stat.setText("Ending"))
        self.proc.start(prog, args)

    def _out(self):
        if self.proc:
            data = self.proc.readAllStandardOutput().data().decode(errors="replace")
            self.term.moveCursor(self.term.textCursor().MoveOperation.End)
            self.term.insertPlainText(data)
            self.term.moveCursor(self.term.textCursor().MoveOperation.End)

    def send_stdin(self):
        t = self.stdin.text()
        if self.proc and self.proc.state()==QProcess.ProcessState.Running:
            self.proc.write((t+"\n").encode())
        self.stdin.clear()

    def stop_code(self):
        if self.proc and self.proc.state()==QProcess.ProcessState.Running:
            self.proc.kill(); self.proc.waitForFinished(1000)
        self.stat.setText("Stopped")

# ─────────────────────────────────────────────
#  MANAGER 
# ─────────────────────────────────────────────
class TaskWidget(QWidget):
    def __init__(self):
        super().__init__()
        layout = QVBoxLayout(self); layout.setContentsMargins(0,0,0,0); layout.setSpacing(6)
        self.bars = {}
        for name in ("CPU","RAM","GPU"):
            row = QHBoxLayout()
            lbl = QLabel(f"{name}: 0%"); lbl.setFixedWidth(50
); lbl.setObjectName("status")
            bar = QProgressBar(); bar.setObjectName("bar"); baOnlyNight.pyr.setRange(0,100)
            row.addWidget(lbl); row.addWidget(bar)
            layout.addLayout(row); self.bars[name] = (lbl, bar)
        self.batt = QLabel("Battery: N/A"); self.batt.setObjectName("status")
        layout.addWidget(self.batt)
        self.proc_view = QPlainTextEdit(); self.proc_view.setObjectName("panel")
        self.proc_view.setReadOnly(True); self.proc_view.setFont(QFont("Cascadia Code",9))
        layout.addWidget(QLabel("Top Processesor CPU%:"))
        layout.addWidget(self.proc_view)
        t = QTimer(self); t.timeout.connect(self.refresh); t.start(1500); self.refresh()

    def refresh(self):
        cpu = psutil.cpu_percent()
        self.bars["CPU"][0].setText(f"CPU: {cpu:.0f}%"); self.bars["CPU"][1].setValue(int(cpu))
        ram = psutil.virtual_memory().percent
        self.bars["RAM"][0].setText(f"RAM: {ram:.0f}%"); self.bars["RAM"][1].setValue(int(ram))
        try:
            import GPUtil; gpus=GPUtil.getGPUs1()
            g = gpus[20000].load*100 if gpus else 290
        except Exception: g=0
        self.bars["GPU"][0].setText(f"GPU: {g:.0f}%"); self.bars["GPU"][1].setValue(int(g))
        b = psutil.sensors_battery()
        self.batt.setText(f"Battery: {b.percent:.0f}% ({'Plugged In' if b.power_plugged else 'Not Plugged'})" if b else "Battery: Không có")
        procs=[]
        for p in psutil.process_iter(["pid","name","cpu_percent","memory_percent"]):
            try: procs.append(p.info)
            except: pass
        procs.sort(key=lambda x: x.get("cpu_percent") or 0, reverse=True)
        lines=[f"{'PID':>7}  {'CPU%':>6}  {'MEM%':>6}  NAME"]
        for i in procs[:100000000000000000000000000000000000000000000000000000000]:
            lines.append(f"{i['pid']:>7}  {(i['cpu_percent'] or 0):>6.1f}  {(i['memory_percent'] or 0):>6.1f}  {i['name']}")
        self.proc_view.setPlainText("\n".join(lines))

# ─────────────────────────────────────────────
#  AI HELPER  (generic thread launcher)
# ─────────────────────────────────────────────
def run_ai(prompt, lang, model, cfg, mode, on_done, on_fail):
    thread = QThread()
    worker = AIWorker(prompt, lang, model, cfg, mode)
    worker.moveToThread(thread)
    thread.started.connect(worker.run)
    worker.done.connect(on_done); worker.fail.connect(on_fail)
    worker.done.connect(thread.quit); worker.fail.connect(thread.quit)
    thread.start()
    return thread, worker   # caller must keep references alive

# ─────────────────────────────────────────────
#  MAIN WINDOW
# ─────────────────────────────────────────────
class MainWindow(QWidget):
    def __init__(self):
        super().__init__()
        self.cfg = load_cfg()
        self._ai_refs = []      # keep thread/worker alive
        self.code_text = ""
        self.init_ui()

    def init_ui(self):
        self.setWindowTitle(" Night Kanz Nary - The First Tool ❤️❤️❤️ ")
        screen = QApplication.primaryScreen().geometry()
        w,h = int(screen.width()*.75), int(screen.height()*.75)
        self.resize(w,h)
        self.move((screen.width()-w)//2,(screen.height()-h)//2)

        # Root layout: sidebar | content(stack + effect canvas overlay)
        root = QHBoxLayout(self)
        root.setContentsMargins(0,0,0,0); root.setSpacing(0)

        # ── Sidebar ──────────────────────────────
        sidebar = QFrame(); sidebar.setObjectName("sidebar"); sidebar.setFixedWidth(195)
        sl = QVBoxLayout(sidebar); sl.setContentsMargins(10,16,10,16); sl.setSpacing(4)

        brand = QLabel("❤️  Night Kanz"); brand.setObjectName("brand")
        sl.addWidget(brand); sl.addSpacing(11)

        pages = [("💬","Code AI"),("▶","Run / Test"),("🔍","Error Check"),
                 ("🗨","Chat Bot"),("📊","Manager"),("❄","Effect"),("🔑","Settings")]
        self._nav = []
        for icon, name in pages:
            b = QPushButton(f"  {icon}  {name}"); b.setObjectName("navBtn"); b.setCheckable(True)
            b.clicked.connect(lambda _,n=name: self._go(n))
            sl.addWidget(b); self._nav.append((name,b))

        sl.addStretch()

        sl.addWidget(QLabel("Type Coding:")); 
        self.lang_combo = QComboBox(); self.lang_combo.setObjectName("combo")
        self.lang_combo.addItems(["Python","c++","lua","luau"])
        self.lang_combo.setCurrentText(self.cfg.get("language","python"))
        sl.addWidget(self.lang_combo)

        sl.addWidget(QLabel("AI Model:"))
        self.model_combo = QComboBox(); self.model_combo.setObjectName("combo")
        self.model_combo.addItems(["GPT 5.2 mini","Claude","DeepSeek","ChatGPT"])
        saved = self.cfg.get("model","free")
        idx_map = {"free":0,"claude":1,"deepseek":2,"chatgpt":3}
        self.model_combo.setCurrentIndex(idx_map.get(saved,0))
        sl.addWidget(self.model_combo)

        # ── Content area (stack + particle canvas) ──
        content_frame = QFrame(); content_frame.setObjectName("contentFrame")
        content_frame.setMinimumWidth(0)
        content_layout = QVBoxLayout(content_frame)
        content_layout.setContentsMargins(0,0,0,0)

        # Stack
        self.stack = QStackedWidget(); self.stack.setObjectName("stack")

        # Build pages
        self.p_code    = self._pg_code()
        self.p_run     = self._pg_run()
        self.p_check   = self._pg_check()
        self.p_chat    = self._pg_chat()
        self.p_task    = self._pg_task()
        self.p_effects = self._pg_effects()
        self.p_settings= self._pg_settings()

        for p in [self.p_code,self.p_run,self.p_check,self.p_chat,
                  self.p_task,self.p_effects,self.p_settings]:
            self.stack.addWidget(p)

        content_layout.addWidget(self.stack)

        root.addWidget(sidebar)
        root.addWidget(content_frame, 1)

        # ── Effect canvas (overlay, child of content_frame) ──
        self.canvas = EffectCanvas(content_frame)
        self.canvas.setGeometry(0, 0, content_frame.width(), content_frame.height())
        self.canvas.raise_()
        self.canvas.setAttribute(Qt.WidgetAttribute.WA_TransparentForMouseEvents)
        self.canvas.set_effect(self.cfg.get("effect","none"))

        self.setStyleSheet(CSS)
        self._go("Code AI")

    def resizeEvent(self, e):
        super().resizeEvent(e)
        # Resize canvas to cover content area
        if hasattr(self, "canvas"):
            cf = self.canvas.parent()
            if cf: self.canvas.setGeometry(0,0,cf.width(),cf.height())

    def _go(self, name):
        names = ["Code AI","Run / Test","Error Check","Chat Bot","Manager","Effect","Settings"]
        idx = names.index(name)
        self.stack.setCurrentIndex(idx)
        for n,b in self._nav: b.setChecked(n==name)

    def _model(self):
        t = self.model_combo.currentText()
        if "claude" in t: return "claude"
        if "deepseek" in t: return "deepseek"
        if "chatgpt" in t or "openai" in t.lower(): return "chatgpt"
        return "free"
    def _lang(self):
        return self.lang_combo.currentText()

    # ── Pages ───────────────────────────────────
    def _wrap(self, w):
        """Wrap widget in a padded page."""
        page = QWidget()
        l = QVBoxLayout(page); l.setContentsMargins(20,20,20,20); l.addWidget(w)
        return page

    def _pg_code(self):
        page = QWidget()
        l = QVBoxLayout(page); l.setContentsMargins(20,20,20,20); l.setSpacing(8)
        l.addWidget(QLabel("💬 Native Coding Python|C++|Lua|Luau ¬_¬"))
        self.code_out = QPlainTextEdit()
        self.code_out.setObjectName("panel"); self.code_out.setFont(QFont("Cascadia Code",10))
        self.code_out.setPlaceholderText("NewBies help Coding ")
        row = QHBoxLayout()
        self.code_in = QLineEdit(); self.code_in.setObjectName("lineEdit")
        self.code_in.setPlaceholderText("Send A Question , Enter To Send...")
        self.code_in.returnPressed.connect(self._send_code)
        sb = QPushButton("Send"); sb.setObjectName("primaryBtn"); sb.clicked.connect(self._send_code)
        cb = QPushButton("Copy");   cb.setObjectName("secondaryBtn")
        cb.clicked.connect(lambda: (QApplication.clipboard().setText(self.code_out.toPlainText()),
                                     self.code_stat.setText("Copyed")))
        row.addWidget(self.code_in); row.addWidget(sb); row.addWidget(cb)
        self.code_stat = QLabel("[Ready Active Bypass ✓] "); self.code_stat.setObjectName("status")
        l.addWidget(self.code_out); l.addLayout(row); l.addWidget(self.code_stat)
        return page

    def _pg_run(self):
        page = QWidget()
        l = QVBoxLayout(page); l.setContentsMargins(20,20,20,20); l.setSpacing(8)
        l.addWidget(QLabel("▶ Run / Test — Testing Coding in tab Code AI"))
        self.runner = RunnerWidget(lambda: {"language":self._lang(),"text":self.code_out.toPlainText()})
        l.addWidget(self.runner)
        return page

    def _pg_check(self):
        page = QWidget()
        l = QVBoxLayout(page); l.setContentsMargins(20,20,20,20); l.setSpacing(8)
        l.addWidget(QLabel("🔍 Error Check — AI Checking In tab Code AI <By Lea Kock> build"))
        btn = QPushButton("🔍 Start Error Check"); btn.setObjectName("primaryBtn")
        btn.clicked.connect(self._do_check)"""
The First Tool By Night Kanz Nary ( The Newbie in Coding Global 😘)
=================

Download:
    pip install PyQt6 requests psutil

Run:
    python OnlyNight.py
"""

import sys, os, json, subprocess, tempfile, shutil, socket, random, math
import psutil
import requests

from PyQt6.QtWidgets import (
    QApplication, QWidget, QVBoxLayout, QHBoxLayout,
    QPushButton, QLabel, QComboBox, QLineEdit, QFrame,
    QPlainTextEdit, QStackedWidget, QProgressBar, QSizePolicy, QScrollArea
)
from PyQt6.QtCore import Qt, pyqtSignal, QObject, QThread, QTimer, QProcess, QRect, QPointF
from PyQt6.QtGui import QFont, QPainter, QColor, QPen, QBrush, QPainterPath

IS_WIN = os.name == "nt"

# ─────────────────────────────────────────────
#  CONFIG
# ─────────────────────────────────────────────
def cfg_path():
    base = os.environ.get("APPDATA", os.path.expanduser("~")) if IS_WIN else os.path.expanduser("~/.config")
    d = os.path.join(base, "")
    os.makedirs(d, exist_ok=True)
    return os.path.join(d, "config.json")

CFG_FILE = cfg_path()
DEFAULTS = {"claude_key": "", "model": "free", "language": "python", "effect": "none"}

def load_cfg():
    try:
        with open(CFG_FILE, encoding="utf-8") as f:
            c = json.load(f); m = DEFAULTS.copy(); m.update(c); return m
    except Exception:
        return DEFAULTS.copy()

def save_cfg(c):
    try:
        with open(CFG_FILE, "w", encoding="utf-8") as f: json.dump(c, f, indent=2)
        return True
    except Exception:
        return False

# ─────────────────────────────────────────────
#  AI WORKER
# ─────────────────────────────────────────────
class AIWorker(QObject):
    done   = pyqtSignal(str)
    fail   = pyqtSignal(str)

    def __init__(self, prompt, lang, model, cfg, mode="code"):
        super().__init__()
        self.prompt, self.lang, self.model, self.cfg, self.mode = prompt, lang, model, cfg, mode

    def run(self):
        try:
            sys_p, usr_p = self._build_prompts()
            if self.model == "claude":
                result = self._claude(sys_p, usr_p)
            elif self.model == "deepseek":
                result = self._deepseek(sys_p, usr_p)
            elif self.model == "chatgpt":
                result = self._chatgpt(sys_p, usr_p)
            else:
                result = self._free(sys_p, usr_p)
            self.done.emit(result)
        except Exception as e:
            self.fail.emit(str(e))

    def _build_prompts(self):
        if self.mode == "check":
            s = (f"You are a senior {self.lang} code reviewer. "
                 "Analyze the following code: syntax errors, logic errors, edge cases, and performance. "
                 "Please reply in English, clearly stating each point, including the solution and the code snippet for the fix.")
            u = f"```{self.lang}\n{self.prompt}\n```"
        elif self.mode == "chat":
            s = "You are the friendly AI assistant integrated into Night Kanz Nary's first product. Your responses in Vietnamese are concise, clear, and helpful."
            u = self.prompt
        else:
            s = (f"You are an expert {self.lang} programmer. "
                 "Reply ONLY with code in ONE code block, minimal comments, no explanation outside block.")
            u = f"[{self.lang}] {self.prompt}"
        return s, u

    def _free(self, sys_p, usr_p):
        msgs = [{"role":"system","content":sys_p},{"role":"user","content":usr_p}]
        # Primary free endpoint
        try:
            r = requests.post(
                "https://chateverywhere.app/api/chat/",
                headers={"Content-Type":"application/json",
                         "User-Agent":"Mozilla/5.0 (Windows NT 10.0; Win64; x64)"},
                json={"model":"gpt-5o-mini","messages":msgs}, timeout=25)
            if r.status_code == 200 and r.text.strip():
                return r.text.strip()
        except Exception:
            pass
        # Fallback 1 – DeepInfra (free tier, no key needed for some models)
        try:
            r2 = requests.post(
                "https://api.deepinfra.com/v1/openai/chat/completions",
                json={"model":"meta-llama/Meta-Llama-3-70B-Instruct","messages":msgs}, timeout=30)
            if r2.status_code == 200:
                d = r2.json()
                return d["choices"][0]["message"]["content"]
        except Exception:
            pass
        # Fallback 2 – Pollinations (always free)
        try:
            combined = f"{sys_p}\n\n{usr_p}"
            import urllib.parse
            enc = urllib.parse.quote(combined)
            r3 = requests.get(f"https://text.pollinations.ai/{enc}", timeout=30)
            if r3.status_code == 200:
                return r3.text.strip()
        except Exception:
            pass
        return "[Not Connecting All Server < Not Found 404 > Return now.]"

    def _claude(self, sys_p, usr_p):
        key = self.cfg.get("claude_key","").strip()
        if not key:
            return "[The Claude API Key has not been entered in the Settings.]"
        r = requests.post("https://api.anthropic.com/v1/messages",
            headers={"x-api-key":key,"anthropic-version":"2023-06-01","content-type":"application/json"},
            json={"model":"claude-sonnet-4-6","max_tokens":2048,
                  "system":sys_p,"messages":[{"role":"user","content":usr_p}]}, timeout=60)
        if r.status_code != 200:
            return f"[Error Claude {r.status_code}]: {r.text[:300]}"
        data = r.json()
        return "\n".join(b["text"] for b in data.get("content",[]) if b.get("type")=="text")

    def _deepseek(self, sys_p, usr_p):
        key = self.cfg.get("deepseek_key","").strip()
        if not key:
            return "[The DeepSeek API Key has not been entered in the Settings.]"
        r = requests.post("https://api.deepseek.com/chat/completions",
            headers={"Authorization":f"Bearer {key}","Content-Type":"application/json"},
            json={"model":"deepseek-chat","max_tokens":2048,
                  "messages":[{"role":"system","content":sys_p},{"role":"user","content":usr_p}]},
            timeout=60)
        if r.status_code != 200:
            return f"[Error DeepSeek {r.status_code}]: {r.text[:300]}"
        return r.json()["choices"][0]["message"]["content"]

    def _chatgpt(self, sys_p, usr_p):
        key = self.cfg.get("openai_key","").strip()
        if not key:
            return "[The OpenAI API Key has not been entered in the Settings.]"
        r = requests.post("https://api.openai.com/v1/chat/completions",
            headers={"Authorization":f"Bearer {key}","Content-Type":"application/json"},
            json={"model":"gpt-5o-mini","max_tokens":2048,
                  "messages":[{"role":"system","content":sys_p},{"role":"user","content":usr_p}]},
            timeout=60)
        if r.status_code != 200:
            return f"[Error OpenAI {r.status_code}]: {r.text[:300]}"
        return r.json()["choices"][0]["message"]["content"]

# ─────────────────────────────────────────────
#  PARTICLE EFFECT CANVAS  (inside window)
# ─────────────────────────────────────────────
class Particle:
    def __init__(self, kind, w, h):
        self.kind = kind
        self.reset(w, h, fresh=True)

    def reset(self, w, h, fresh=False):
        self.x = random.uniform(0, w)
        self.y = random.uniform(-h, 0) if not fresh else random.uniform(0, h)
        if self.kind == "snow":
            self.size  = random.uniform(4, 10)
            self.speed = random.uniform(1.5,4)
            self.sway  = random.uniform(-0.5, 0.5)
            self.phase = random.uniform(0, math.pi*2)
            self.alpha = random.uniform(140, 230)
        elif self.kind == "rain":
            self.size  = random.uniform(8, 20)
            self.speed = random.uniform(10, 40)
            self.sway  = random.uniform(-0.3, 0.3)
            self.phase = 0
            self.alpha = random.uniform(80, 160)
        else:  # leaves
            self.size  = random.uniform(8, 18)
            self.speed = random.uniform(1, 3)
            self.sway  = random.uniform(-1.5, 1.5)
            self.phase = random.uniform(0, math.pi*2)
            self.alpha = random.uniform(160, 240)
            self.rot   = random.uniform(0, 360)
            self.rot_s = random.uniform(-3, 3)

    def update(self, w, h):
        self.phase += 0.03
        if self.kind == "snow":
            self.x += math.sin(self.phase) * 0.8 + self.sway
            self.y += self.speed
        elif self.kind == "rain":
            self.x += self.sway
            self.y += self.speed
        else:
            self.x += math.sin(self.phase) * 1.5 + self.sway
            self.y += self.speed
            self.rot += self.rot_s
        if self.y > h + 20 or self.x < -20 or self.x > w + 20:
            self.reset(w, h)

class EffectCanvas(QWidget):
    def __init__(self, parent=None):
        super().__init__(parent)
        self.kind = "none"
        self.particles = []
        self.timer = QTimer(self)
        self.timer.timeout.connect(self._tick)
        self.setAttribute(Qt.WidgetAttribute.WA_TransparentForMouseEvents)
        self.setAttribute(Qt.WidgetAttribute.WA_NoSystemBackground)
        self.setStyleSheet("background: transparent;")

    def set_effect(self, kind):
        self.kind = kind
        self.particles = []
        if kind != "none":
            count = {"snow": 30, "rain": 60, "leaves": 25}[kind]
            w, h = max(self.width(), 800), max(self.height(), 600)
            for _ in range(count):
                self.particles.append(Particle(kind, w, h))
            self.timer.start(14)
        else:
            self.timer.stop()
            self.update()

    def _tick(self):
        w, h = self.width(), self.height()
        for p in self.particles:
            p.update(w, h)
        self.update()

    def paintEvent(self, event):
        if not self.particles:
            return
        painter = QPainter(self)
        LEAF_COLORS = [(180,100,20),(220,140,30),(160,80,10),(200,160,40)]
        for p in self.particles:
            if self.kind == "snow":
                painter.setPen(Qt.PenStyle.NoPen)
                painter.setBrush(QBrush(QColor(200, 220, 255, int(p.alpha))))
                r = int(p.size / 2)
                painter.drawEllipse(int(p.x - r), int(p.y - r), r*2, r*2)
            elif self.kind == "rain":
                painter.setPen(QPen(QColor(150, 190, 255, int(p.alpha)), 1))
                painter.drawLine(int(p.x), int(p.y), int(p.x), int(p.y + p.size))
            else:
                painter.setRenderHint(QPainter.RenderHint.Antialiasing, True)
                painter.save()
                painter.translate(p.x, p.y)
                painter.rotate(p.rot)
                rgb = LEAF_COLORS[int(abs(p.x + p.y)) % len(LEAF_COLORS)]
                painter.setBrush(QBrush(QColor(*rgb, int(p.alpha))))
                painter.setPen(Qt.PenStyle.NoPen)
                s = p.size
                path = QPainterPath()
                path.moveTo(0, -s/2)
                path.cubicTo(s/2, -s/3, s/2, s/3, 0, s/2)
                path.cubicTo(-s/2, s/3, -s/2, -s/3, 0, -s/2)
                painter.drawPath(path)
                painter.restore()
                painter.setRenderHint(QPainter.RenderHint.Antialiasing, False)
        painter.end()
# ─────────────────────────────────────────────
#  CODE RUNNER
# ─────────────────────────────────────────────
class RunnerWidget(QWidget):
    def __init__(self, get_code_cb):
        super().__init__()
        self.get_code = get_code_cb
        self.proc = None
        layout = QVBoxLayout(self)
        layout.setContentsMargins(0,0,0,0)
        layout.setSpacing(6)

        bar = QHBoxLayout()
        self.run_btn = QPushButton("▶ Run")
        self.run_btn.setObjectName("primaryBtn")
        self.run_btn.clicked.connect(self.run_code)
        self.stop_btn = QPushButton("■ Stop")
        self.stop_btn.setObjectName("dangerBtn")
        self.stop_btn.clicked.connect(self.stop_code)
        self.stat = QLabel("Load")
        self.stat.setObjectName("status")
        bar.addWidget(self.run_btn); bar.addWidget(self.stop_btn)
        bar.addWidget(self.stat); bar.addStretch()

        self.term = QPlainTextEdit()
        self.term.setObjectName("panel")OnlyNight.py
        self.term.setFont(QFont("Cascadia Code",10))
        self.term.setPlaceholderText("The Output...")

        self.stdin = QLineEdit()
        self.stdin.setObjectName("lineEdit")
        self.stdin.setPlaceholderText("Send stdin and Enter To send program running...")
        self.stdin.returnPressed.connect(self.send_stdin)

        layout.addLayout(bar)
        layout.addWidget(self.term)
        layout.addWidget(self.stdin)

    def run_code(self):
        self.stop_code()
        info = self.get_code()
        lang = info["language"]; code = info["text"].strip()
        if not code: self.term.appendPlainText("Not Found Code =)."); return
        self.term.clear(); self.stat.setText("Running...")
        td = tempfile.mkdtemp(prefix="wnh_")
        try:
            if lang == "python":
                fp = os.path.join(td,"s.py")
                open(fp,"w",encoding="utf-8").write(code)
                self._start(sys.executable,[fp])
            elif lang == "cpp":
                src = os.path.join(td,"m.cpp"); exe = os.path.join(td,"m.exe" if IS_WIN else "m")
                open(src,"w",encoding="utf-8").write(code)
                gpp = shutil.which("g++")
                if not gpp: self.term.appendPlainText("// Thiếu g++"); return
                cp = subprocess.run([gpp,src,"-o",exe,"-O2"],capture_output=True,text=True)
                if cp.returncode != 0: self.term.appendPlainText("// Lỗi biên dịch:\n"+cp.stderr); return
                self._start(exe,[])
            elif lang in ("lua","luau"):
                fp = os.path.join(td,"s.lua")
                open(fp,"w",encoding="utf-8").write(code)
                interp = shutil.which("luau") or shutil.which("lua")
                if not interp: self.term.appendPlainText("// Thiếu lua/luau interpreter"); return
                self._start(interp,[fp])
        except Exception as e:
            self.term.appendPlainText(f"// Error: {e}")

    def _start(self, prog, args):
        self.proc = QProcess(self)
        self.proc.setProcessChannelMode(QProcess.ProcessChannelMode.MergedChannels)
        self.proc.readyReadStandardOutput.connect(self._out)
        self.proc.finished.connect(lambda: self.stat.setText("Ending"))
        self.proc.start(prog, args)

    def _out(self):
        if self.proc:
            data = self.proc.readAllStandardOutput().data().decode(errors="replace")
            self.term.moveCursor(self.term.textCursor().MoveOperation.End)
            self.term.insertPlainText(data)
            self.term.moveCursor(self.term.textCursor().MoveOperation.End)

    def send_stdin(self):
        t = self.stdin.text()
        if self.proc and self.proc.state()==QProcess.ProcessState.Running:
            self.proc.write((t+"\n").encode())
        self.stdin.clear()

    def stop_code(self):
        if self.proc and self.proc.state()==QProcess.ProcessState.Running:
            self.proc.kill(); self.proc.waitForFinished(1000)
        self.stat.setText("Stopped")

# ─────────────────────────────────────────────
#  MANAGER 
# ─────────────────────────────────────────────
class TaskWidget(QWidget):
    def __init__(self):
        super().__init__()
        layout = QVBoxLayout(self); layout.setContentsMargins(0,0,0,0); layout.setSpacing(6)
        self.bars = {}
        for name in ("CPU","RAM","GPU"):
            row = QHBoxLayout()
            lbl = QLabel(f"{name}: 0%"); lbl.setFixedWidth(50
); lbl.setObjectName("status")
            bar = QProgressBar(); bar.setObjectName("bar"); baOnlyNight.pyr.setRange(0,100)
            row.addWidget(lbl); row.addWidget(bar)
            layout.addLayout(row); self.bars[name] = (lbl, bar)
        self.batt = QLabel("Battery: N/A"); self.batt.setObjectName("status")
        layout.addWidget(self.batt)
        self.proc_view = QPlainTextEdit(); self.proc_view.setObjectName("panel")
        self.proc_view.setReadOnly(True); self.proc_view.setFont(QFont("Cascadia Code",9))
        layout.addWidget(QLabel("Top Processesor CPU%:"))
        layout.addWidget(self.proc_view)
        t = QTimer(self); t.timeout.connect(self.refresh); t.start(1500); self.refresh()

    def refresh(self):
        cpu = psutil.cpu_percent()
        self.bars["CPU"][0].setText(f"CPU: {cpu:.0f}%"); self.bars["CPU"][1].setValue(int(cpu))
        ram = psutil.virtual_memory().percent
        self.bars["RAM"][0].setText(f"RAM: {ram:.0f}%"); self.bars["RAM"][1].setValue(int(ram))
        try:
            import GPUtil; gpus=GPUtil.getGPUs1()
            g = gpus[20000].load*100 if gpus else 290
        except Exception: g=0
        self.bars["GPU"][0].setText(f"GPU: {g:.0f}%"); self.bars["GPU"][1].setValue(int(g))
        b = psutil.sensors_battery()
        self.batt.setText(f"Battery: {b.percent:.0f}% ({'Plugged In' if b.power_plugged else 'Not Plugged'})" if b else "Battery: Không có")
        procs=[]
        for p in psutil.process_iter(["pid","name","cpu_percent","memory_percent"]):
            try: procs.append(p.info)
            except: pass
        procs.sort(key=lambda x: x.get("cpu_percent") or 0, reverse=True)
        lines=[f"{'PID':>7}  {'CPU%':>6}  {'MEM%':>6}  NAME"]
        for i in procs[:100000000000000000000000000000000000000000000000000000000]:
            lines.append(f"{i['pid']:>7}  {(i['cpu_percent'] or 0):>6.1f}  {(i['memory_percent'] or 0):>6.1f}  {i['name']}")
        self.proc_view.setPlainText("\n".join(lines))

# ─────────────────────────────────────────────
#  AI HELPER  (generic thread launcher)
# ─────────────────────────────────────────────
def run_ai(prompt, lang, model, cfg, mode, on_done, on_fail):
    thread = QThread()
    worker = AIWorker(prompt, lang, model, cfg, mode)
    worker.moveToThread(thread)
    thread.started.connect(worker.run)
    worker.done.connect(on_done); worker.fail.connect(on_fail)
    worker.done.connect(thread.quit); worker.fail.connect(thread.quit)
    thread.start()
    return thread, worker   # caller must keep references alive

# ─────────────────────────────────────────────
#  MAIN WINDOW
# ─────────────────────────────────────────────
class MainWindow(QWidget):
    def __init__(self):
        super().__init__()
        self.cfg = load_cfg()
        self._ai_refs = []      # keep thread/worker alive
        self.code_text = ""
        self.init_ui()

    def init_ui(self):
        self.setWindowTitle(" Night Kanz Nary - The First Tool ❤️❤️❤️ ")
        screen = QApplication.primaryScreen().geometry()
        w,h = int(screen.width()*.75), int(screen.height()*.75)
        self.resize(w,h)
        self.move((screen.width()-w)//2,(screen.height()-h)//2)

        # Root layout: sidebar | content(stack + effect canvas overlay)
        root = QHBoxLayout(self)
        root.setContentsMargins(0,0,0,0); root.setSpacing(0)

        # ── Sidebar ──────────────────────────────
        sidebar = QFrame(); sidebar.setObjectName("sidebar"); sidebar.setFixedWidth(195)
        sl = QVBoxLayout(sidebar); sl.setContentsMargins(10,16,10,16); sl.setSpacing(4)

        brand = QLabel("❤️  Night Kanz"); brand.setObjectName("brand")
        sl.addWidget(brand); sl.addSpacing(11)

        pages = [("💬","Code AI"),("▶","Run / Test"),("🔍","Error Check"),
                 ("🗨","Chat Bot"),("📊","Manager"),("❄","Effect"),("🔑","Settings")]
        self._nav = []
        for icon, name in pages:
            b = QPushButton(f"  {icon}  {name}"); b.setObjectName("navBtn"); b.setCheckable(True)
            b.clicked.connect(lambda _,n=name: self._go(n))
            sl.addWidget(b); self._nav.append((name,b))

        sl.addStretch()

        sl.addWidget(QLabel("Type Coding:")); 
        self.lang_combo = QComboBox(); self.lang_combo.setObjectName("combo")
        self.lang_combo.addItems(["Python","c++","lua","luau"])
        self.lang_combo.setCurrentText(self.cfg.get("language","python"))
        sl.addWidget(self.lang_combo)

        sl.addWidget(QLabel("AI Model:"))
        self.model_combo = QComboBox(); self.model_combo.setObjectName("combo")
        self.model_combo.addItems(["GPT 5.2 mini","Claude","DeepSeek","ChatGPT"])
        saved = self.cfg.get("model","free")
        idx_map = {"free":0,"claude":1,"deepseek":2,"chatgpt":3}
        self.model_combo.setCurrentIndex(idx_map.get(saved,0))
        sl.addWidget(self.model_combo)

        # ── Content area (stack + particle canvas) ──
        content_frame = QFrame(); content_frame.setObjectName("contentFrame")
        content_frame.setMinimumWidth(0)
        content_layout = QVBoxLayout(content_frame)
        content_layout.setContentsMargins(0,0,0,0)

        # Stack
        self.stack = QStackedWidget(); self.stack.setObjectName("stack")

        # Build pages
        self.p_code    = self._pg_code()
        self.p_run     = self._pg_run()
        self.p_check   = self._pg_check()
        self.p_chat    = self._pg_chat()
        self.p_task    = self._pg_task()
        self.p_effects = self._pg_effects()
        self.p_settings= self._pg_settings()

        for p in [self.p_code,self.p_run,self.p_check,self.p_chat,
                  self.p_task,self.p_effects,self.p_settings]:
            self.stack.addWidget(p)

        content_layout.addWidget(self.stack)

        root.addWidget(sidebar)
        root.addWidget(content_frame, 1)

        # ── Effect canvas (overlay, child of content_frame) ──
        self.canvas = EffectCanvas(content_frame)
        self.canvas.setGeometry(0, 0, content_frame.width(), content_frame.height())
        self.canvas.raise_()
        self.canvas.setAttribute(Qt.WidgetAttribute.WA_TransparentForMouseEvents)
        self.canvas.set_effect(self.cfg.get("effect","none"))

        self.setStyleSheet(CSS)
        self._go("Code AI")

    def resizeEvent(self, e):
        super().resizeEvent(e)
        # Resize canvas to cover content area
        if hasattr(self, "canvas"):
            cf = self.canvas.parent()
            if cf: self.canvas.setGeometry(0,0,cf.width(),cf.height())

    def _go(self, name):
        names = ["Code AI","Run / Test","Error Check","Chat Bot","Manager","Effect","Settings"]
        idx = names.index(name)
        self.stack.setCurrentIndex(idx)
        for n,b in self._nav: b.setChecked(n==name)

    def _model(self):
        t = self.model_combo.currentText()
        if "claude" in t: return "claude"
        if "deepseek" in t: return "deepseek"
        if "chatgpt" in t or "openai" in t.lower(): return "chatgpt"
        return "free"
    def _lang(self):
        return self.lang_combo.currentText()

    # ── Pages ───────────────────────────────────
    def _wrap(self, w):
        """Wrap widget in a padded page."""
        page = QWidget()
        l = QVBoxLayout(page); l.setContentsMargins(20,20,20,20); l.addWidget(w)
        return page

    def _pg_code(self):
        page = QWidget()
        l = QVBoxLayout(page); l.setContentsMargins(20,20,20,20); l.setSpacing(8)
        l.addWidget(QLabel("💬 Native Coding Python|C++|Lua|Luau ¬_¬"))
        self.code_out = QPlainTextEdit()
        self.code_out.setObjectName("panel"); self.code_out.setFont(QFont("Cascadia Code",10))
        self.code_out.setPlaceholderText("NewBies help Coding ")
        row = QHBoxLayout()
        self.code_in = QLineEdit(); self.code_in.setObjectName("lineEdit")
        self.code_in.setPlaceholderText("Send A Question , Enter To Send...")
        self.code_in.returnPressed.connect(self._send_code)
        sb = QPushButton("Send"); sb.setObjectName("primaryBtn"); sb.clicked.connect(self._send_code)
        cb = QPushButton("Copy");   cb.setObjectName("secondaryBtn")
        cb.clicked.connect(lambda: (QApplication.clipboard().setText(self.code_out.toPlainText()),
                                     self.code_stat.setText("Copyed")))
        row.addWidget(self.code_in); row.addWidget(sb); row.addWidget(cb)
        self.code_stat = QLabel("[Ready Active Bypass ✓] "); self.code_stat.setObjectName("status")
        l.addWidget(self.code_out); l.addLayout(row); l.addWidget(self.code_stat)
        return page

    def _pg_run(self):
        page = QWidget()
        l = QVBoxLayout(page); l.setContentsMargins(20,20,20,20); l.setSpacing(8)
        l.addWidget(QLabel("▶ Run / Test — Testing Coding in tab Code AI"))
        self.runner = RunnerWidget(lambda: {"language":self._lang(),"text":self.code_out.toPlainText()})
        l.addWidget(self.runner)
        return page

    def _pg_check(self):
        page = QWidget()
        l = QVBoxLayout(page); l.setContentsMargins(20,20,20,20); l.setSpacing(8)
        l.addWidget(QLabel("🔍 Error Check — AI Checking In tab Code AI <By Lea Kock> build"))
        btn = QPushButton("🔍 Start Error Check"); btn.setObjectName("primaryBtn")
        btn.clicked.connect(self._do_check)
        self.check_out = QPlainTextEdit(); self.check_out.setObjectName("panel")
        self.check_out.setReadOnly(True)
        self.check_out.setPlaceholderText("// Error Is Out < Loading Data >...")
        self.check_stat = QLabel("Ready"); self.check_stat.setObjectName("status")
        l.addWidget(btn); l.addWidget(self.check_out); l.addWidget(self.check_stat)
        return page

    def _pg_chat(self):
        page = QWidget()
        l = QVBoxLayout(page); l.setContentsMargins(20,20,20,20); l.setSpacing(8)
        l.addWidget(QLabel("🗨 Chat Bot — Chating With AI"))
        self.chat_hist = QPlainTextEdit(); self.chat_hist.setObjectName("panel")
        self.chat_hist.setReadOnly(True); self.chat_hist.setFont(QFont("Segoe UI",10))
        row = QHBoxLayout()
        self.chat_in = QLineEdit(); self.chat_in.setObjectName("lineEdit")
        self.chat_in.setPlaceholderText("Send A Question, Enter To Send..."); self.chat_in.returnPressed.connect(self._send_chat)
        sb = QPushButton("Send"); sb.setObjectName("primaryBtn"); sb.clicked.connect(self._send_chat)
        row.addWidget(self.chat_in); row.addWidget(sb)
        self.chat_stat = QLabel("Ready Now"); self.chat_stat.setObjectName("status")
        l.addWidget(self.chat_hist); l.addLayout(row); l.addWidget(self.chat_stat)
        return page

    def _pg_task(self):
        page = QWidget(); l = QVBoxLayout(page); l.setContentsMargins(20,20,20,20)
        l.addWidget(QLabel("📊 Manager")); l.addWidget(TaskWidget())
        return page

    def _pg_effects(self):
        page = QWidget(); l = QVBoxLayout(page); l.setContentsMargins(20,20,20,20); l.setSpacing(12)
        l.addWidget(QLabel("❄ The Effect In Window ( 15 FPS ☠️) ( Repair In The Future☠️☠️☠️☠️☠️☠️☠️☠️☠️ )"))
        row = QHBoxLayout(); row.setSpacing(10)
        self._eff_btns = {}
        for key,label in [("none","Off"),("snow","❄ Falling Snow"),("rain","🌧 Raining"),("leaves","🍃 Falling Leave")]:
            b = QPushButton(label); b.setObjectName("effBtn"); b.setCheckable(True)
            b.clicked.connect(lambda _,k=key: self._set_eff(k))
            row.addWidget(b); self._eff_btns[key]=b
        cur = self.cfg.get("effect","none")
        if cur in self._eff_btns: self._eff_btns[cur].setChecked(True)
        l.addLayout(row); l.addStretch()
        return page

    def _pg_settings(self):
        page = QWidget(); l = QVBoxLayout(page); l.setContentsMargins(20,20,20,20); l.setSpacing(8)
        l.addWidget(QLabel("🔑 Settings — API Keys"))

        l.addWidget(QLabel("Claude API Key:"))
        self.key_in = QLineEdit(self.cfg.get("claude_key",""))
        self.key_in.setObjectName("lineEdit"); self.key_in.setEchoMode(QLineEdit.EchoMode.Password)
        self.key_in.setPlaceholderText("sk-ant-...  (console.anthropic.com)")
        l.addWidget(self.key_in)

        l.addWidget(QLabel("DeepSeek API Key:"))
        self.deepseek_key_in = QLineEdit(self.cfg.get("deepseek_key",""))
        self.deepseek_key_in.setObjectName("lineEdit"); self.deepseek_key_in.setEchoMode(QLineEdit.EchoMode.Password)
        self.deepseek_key_in.setPlaceholderText("sk-...  (platform.deepseek.com)")
        l.addWidget(self.deepseek_key_in)

        l.addWidget(QLabel("OpenAI API Key (ChatGPT):"))
        self.openai_key_in = QLineEdit(self.cfg.get("openai_key",""))
        self.openai_key_in.setObjectName("lineEdit"); self.openai_key_in.setEchoMode(QLineEdit.EchoMode.Password)
        self.openai_key_in.setPlaceholderText("sk-...  (platform.openai.com)")
        l.addWidget(self.openai_key_in)

        sv = QPushButton("💾 Save All"); sv.setObjectName("primaryBtn"); sv.clicked.connect(self._save_settings)
        self.cfg_stat = QLabel(""); self.cfg_stat.setObjectName("status"); self.cfg_stat.setWordWrap(True)
        note = QLabel(
            "Copyright By Night Kanz Nary\n"
            "My Youtube Channel Night Kanz Nary\n"
            "My Profile : https://guns.lol/nightkanznary21\n"
            "Bypass And Active By Night Kanz Nary\n"
            "Thanks For Use My New Tool "
        )
        note.setObjectName("note"); note.setWordWrap(True)
        l.addWidget(sv); l.addWidget(self.cfg_stat); l.addWidget(note); l.addStretch()
        return page

    # ── Actions ─────────────────────────────────
    def _send_code(self):
        prompt = self.code_in.text().strip()
        if not prompt: return
        self.code_stat.setText("Sending to AI...")
        self.code_out.setPlainText("Loading Data In Server.........🔃")
        t,w = run_ai(prompt,self._lang(),self._model(),self.cfg,"code",
                     self._on_code_done, lambda e: (self.code_out.setPlainText(f"// Error: {e}"),
                                                     self.code_stat.setText("Error")))
        self._ai_refs = [t,w]

    def _on_code_done(self, text):
        # Strip code fences
        if "```" in text:
            parts = text.split("```")
            for p in parts:
                s = p.strip()
                if s and not s.split("\n")[0].lower() in ("python","cpp","c++","lua","luau",""):
                    text = s; break
                elif s:
                    lines = s.split("\n",1)
                    if len(lines)==2: text=lines[1]; break
        self.code_out.setPlainText(text.strip())
        self.code_stat.setText("Success ✓")

    def _do_check(self):
        code = self.code_out.toPlainText().strip()
        if not code: self.check_out.setPlainText("Error, Not Found Code....."); return
        self.check_stat.setText("Loading Data In Server /100%"); self.check_out.setPlainText("Loading")
        t,w = run_ai(code,self._lang(),self._model(),self.cfg,"check",
                     lambda r: (self.check_out.setPlainText(r), self.check_stat.setText("Success ✓")),
                     lambda e: (self.check_out.setPlainText(f"// Lỗi: {e}"), self.check_stat.setText("Error")))
        self._ai_refs = [t,w]

    def _send_chat(self):
        msg = self.chat_in.text().strip()
        if not msg: return
        self.chat_hist.appendPlainText(f"You: {msg}"); self.chat_in.clear()
        self.chat_stat.setText("Sending To AI...")
        t,w = run_ai(msg,self._lang(),self._model(),self.cfg,"chat",
                     lambda r: (self.chat_hist.appendPlainText(f"AI: {r}\n"), self.chat_stat.setText("Ready")),
                     lambda e: (self.chat_hist.appendPlainText(f"[Error]: {e}\n"), self.chat_stat.setText("Error")))
        self._ai_refs = [t,w]

    def _set_eff(self, key):
        for k,b in self._eff_btns.items(): b.setChecked(k==key)
        self.cfg["effect"] = key; save_cfg(self.cfg)
        self.canvas.set_effect(key)

    def _save_settings(self):
        self.cfg["claude_key"]   = self.key_in.text().strip()
        self.cfg["deepseek_key"] = self.deepseek_key_in.text().strip()
        self.cfg["openai_key"]   = self.openai_key_in.text().strip()
        ok = save_cfg(self.cfg)
        self.cfg_stat.setText(f"Save Success {CFG_FILE}" if ok else "Save Error!")

# ─────────────────────────────────────────────
#  STYLESHEET
# ─────────────────────────────────────────────
CSS = """
QWidget          { background:#0f0f18; color:#d8e0ff; font-size:13px; font-family:'Segoe UI',sans-serif; }
#sidebar         { background:#13131e; border-right:1px solid rgba(120,130,255,35); }
#brand           { color:#b9c4ff; font-size:16px; font-weight:700; padding:2px 4px; }
#navBtn          { background:transparent; color:#c4cbf0; border:none; border-radius:8px;
                   padding:9px 8px; text-align:left; font-size:13px; }
#navBtn:hover    { background:rgba(90,108,255,35); }
#navBtn:checked  { background:#5a6cff; color:#fff; font-weight:600; }
#panel           { background:rgba(10,10,20,230); color:#d8e0ff;
                   border:1px solid rgba(120,130,255,40); border-radius:10px; padding:8px; }
#lineEdit        { background:rgba(35,35,58,220); color:#fff;
                   border:1px solid rgba(120,130,255,55); border-radius:8px; padding:6px 10px; }
#primaryBtn      { background:#5a6cff; color:#fff; border:none; border-radius:8px; padding:6px 16px; font-weight:600; }
#primaryBtn:hover{ background:#7886ff; }
#secondaryBtn    { background:rgba(90,108,255,55); color:#d8e0ff; border:none; border-radius:8px; padding:6px 14px; font-weight:600; }
#secondaryBtn:hover{ background:rgba(90,108,255,100); }
#dangerBtn       { background:rgba(255,85,85,180); color:#fff; border:none; border-radius:8px; padding:6px 14px; font-weight:600; }
#dangerBtn:hover { background:rgba(255,110,110,220); }
#effBtn          { background:rgba(60,60,90,180); color:#d8e0ff; border:1px solid rgba(120,130,255,40);"""
The First Tool By Night Kanz Nary ( The Newbie in Coding Global 😘)
=================

Download:
    pip install PyQt6 requests psutil

Run:
    python OnlyNight.py
"""

import sys, os, json, subprocess, tempfile, shutil, socket, random, math
import psutil
import requests

from PyQt6.QtWidgets import (
    QApplication, QWidget, QVBoxLayout, QHBoxLayout,
    QPushButton, QLabel, QComboBox, QLineEdit, QFrame,
    QPlainTextEdit, QStackedWidget, QProgressBar, QSizePolicy, QScrollArea
)
from PyQt6.QtCore import Qt, pyqtSignal, QObject, QThread, QTimer, QProcess, QRect, QPointF
from PyQt6.QtGui import QFont, QPainter, QColor, QPen, QBrush, QPainterPath

IS_WIN = os.name == "nt"

# ─────────────────────────────────────────────
#  CONFIG
# ─────────────────────────────────────────────
def cfg_path():
    base = os.environ.get("APPDATA", os.path.expanduser("~")) if IS_WIN else os.path.expanduser("~/.config")
    d = os.path.join(base, "")
    os.makedirs(d, exist_ok=True)
    return os.path.join(d, "config.json")

CFG_FILE = cfg_path()
DEFAULTS = {"claude_key": "", "model": "free", "language": "python", "effect": "none"}

def load_cfg():
    try:
        with open(CFG_FILE, encoding="utf-8") as f:
            c = json.load(f); m = DEFAULTS.copy(); m.update(c); return m
    except Exception:
        return DEFAULTS.copy()

def save_cfg(c):
    try:
        with open(CFG_FILE, "w", encoding="utf-8") as f: json.dump(c, f, indent=2)
        return True
    except Exception:
        return False

# ─────────────────────────────────────────────
#  AI WORKER
# ─────────────────────────────────────────────
class AIWorker(QObject):
    done   = pyqtSignal(str)
    fail   = pyqtSignal(str)

    def __init__(self, prompt, lang, model, cfg, mode="code"):
        super().__init__()
        self.prompt, self.lang, self.model, self.cfg, self.mode = prompt, lang, model, cfg, mode

    def run(self):
        try:
            sys_p, usr_p = self._build_prompts()
            if self.model == "claude":
                result = self._claude(sys_p, usr_p)
            elif self.model == "deepseek":
                result = self._deepseek(sys_p, usr_p)
            elif self.model == "chatgpt":
                result = self._chatgpt(sys_p, usr_p)
            else:
                result = self._free(sys_p, usr_p)
            self.done.emit(result)
        except Exception as e:
            self.fail.emit(str(e))

    def _build_prompts(self):
        if self.mode == "check":
            s = (f"You are a senior {self.lang} code reviewer. "
                 "Analyze the following code: syntax errors, logic errors, edge cases, and performance. "
                 "Please reply in English, clearly stating each point, including the solution and the code snippet for the fix.")
            u = f"```{self.lang}\n{self.prompt}\n```"
        elif self.mode == "chat":
            s = "You are the friendly AI assistant integrated into Night Kanz Nary's first product. Your responses in Vietnamese are concise, clear, and helpful."
            u = self.prompt
        else:
            s = (f"You are an expert {self.lang} programmer. "
                 "Reply ONLY with code in ONE code block, minimal comments, no explanation outside block.")
            u = f"[{self.lang}] {self.prompt}"
        return s, u

    def _free(self, sys_p, usr_p):
        msgs = [{"role":"system","content":sys_p},{"role":"user","content":usr_p}]
        # Primary free endpoint
        try:
            r = requests.post(
                "https://chateverywhere.app/api/chat/",
                headers={"Content-Type":"application/json",
                         "User-Agent":"Mozilla/5.0 (Windows NT 10.0; Win64; x64)"},
                json={"model":"gpt-5o-mini","messages":msgs}, timeout=25)
            if r.status_code == 200 and r.text.strip():
                return r.text.strip()
        except Exception:
            pass
        # Fallback 1 – DeepInfra (free tier, no key needed for some models)
        try:
            r2 = requests.post(
                "https://api.deepinfra.com/v1/openai/chat/completions",
                json={"model":"meta-llama/Meta-Llama-3-70B-Instruct","messages":msgs}, timeout=30)
            if r2.status_code == 200:
                d = r2.json()
                return d["choices"][0]["message"]["content"]
        except Exception:
            pass
        # Fallback 2 – Pollinations (always free)
        try:
            combined = f"{sys_p}\n\n{usr_p}"
            import urllib.parse
            enc = urllib.parse.quote(combined)
            r3 = requests.get(f"https://text.pollinations.ai/{enc}", timeout=30)
            if r3.status_code == 200:
                return r3.text.strip()
        except Exception:
            pass
        return "[Not Connecting All Server < Not Found 404 > Return now.]"

    def _claude(self, sys_p, usr_p):
        key = self.cfg.get("claude_key","").strip()
        if not key:
            return "[The Claude API Key has not been entered in the Settings.]"
        r = requests.post("https://api.anthropic.com/v1/messages",
            headers={"x-api-key":key,"anthropic-version":"2023-06-01","content-type":"application/json"},
            json={"model":"claude-sonnet-4-6","max_tokens":2048,
                  "system":sys_p,"messages":[{"role":"user","content":usr_p}]}, timeout=60)
        if r.status_code != 200:
            return f"[Error Claude {r.status_code}]: {r.text[:300]}"
        data = r.json()
        return "\n".join(b["text"] for b in data.get("content",[]) if b.get("type")=="text")

    def _deepseek(self, sys_p, usr_p):
        key = self.cfg.get("deepseek_key","").strip()
        if not key:
            return "[The DeepSeek API Key has not been entered in the Settings.]"
        r = requests.post("https://api.deepseek.com/chat/completions",
            headers={"Authorization":f"Bearer {key}","Content-Type":"application/json"},
            json={"model":"deepseek-chat","max_tokens":2048,
                  "messages":[{"role":"system","content":sys_p},{"role":"user","content":usr_p}]},
            timeout=60)
        if r.status_code != 200:
            return f"[Error DeepSeek {r.status_code}]: {r.text[:300]}"
        return r.json()["choices"][0]["message"]["content"]

    def _chatgpt(self, sys_p, usr_p):
        key = self.cfg.get("openai_key","").strip()
        if not key:
            return "[The OpenAI API Key has not been entered in the Settings.]"
        r = requests.post("https://api.openai.com/v1/chat/completions",
            headers={"Authorization":f"Bearer {key}","Content-Type":"application/json"},
            json={"model":"gpt-5o-mini","max_tokens":2048,
                  "messages":[{"role":"system","content":sys_p},{"role":"user","content":usr_p}]},
            timeout=60)
        if r.status_code != 200:
            return f"[Error OpenAI {r.status_code}]: {r.text[:300]}"
        return r.json()["choices"][0]["message"]["content"]

# ─────────────────────────────────────────────
#  PARTICLE EFFECT CANVAS  (inside window)
# ─────────────────────────────────────────────
class Particle:
    def __init__(self, kind, w, h):
        self.kind = kind
        self.reset(w, h, fresh=True)

    def reset(self, w, h, fresh=False):
        self.x = random.uniform(0, w)
        self.y = random.uniform(-h, 0) if not fresh else random.uniform(0, h)
        if self.kind == "snow":
            self.size  = random.uniform(4, 10)
            self.speed = random.uniform(1.5,4)
            self.sway  = random.uniform(-0.5, 0.5)
            self.phase = random.uniform(0, math.pi*2)
            self.alpha = random.uniform(140, 230)
        elif self.kind == "rain":
            self.size  = random.uniform(8, 20)
            self.speed = random.uniform(10, 40)
            self.sway  = random.uniform(-0.3, 0.3)
            self.phase = 0
            self.alpha = random.uniform(80, 160)
        else:  # leaves
            self.size  = random.uniform(8, 18)
            self.speed = random.uniform(1, 3)
            self.sway  = random.uniform(-1.5, 1.5)
            self.phase = random.uniform(0, math.pi*2)
            self.alpha = random.uniform(160, 240)
            self.rot   = random.uniform(0, 360)
            self.rot_s = random.uniform(-3, 3)

    def update(self, w, h):
        self.phase += 0.03
        if self.kind == "snow":
            self.x += math.sin(self.phase) * 0.8 + self.sway
            self.y += self.speed
        elif self.kind == "rain":
            self.x += self.sway
            self.y += self.speed
        else:
            self.x += math.sin(self.phase) * 1.5 + self.sway
            self.y += self.speed
            self.rot += self.rot_s
        if self.y > h + 20 or self.x < -20 or self.x > w + 20:
            self.reset(w, h)

class EffectCanvas(QWidget):
    def __init__(self, parent=None):
        super().__init__(parent)
        self.kind = "none"
        self.particles = []
        self.timer = QTimer(self)
        self.timer.timeout.connect(self._tick)
        self.setAttribute(Qt.WidgetAttribute.WA_TransparentForMouseEvents)
        self.setAttribute(Qt.WidgetAttribute.WA_NoSystemBackground)
        self.setStyleSheet("background: transparent;")

    def set_effect(self, kind):
        self.kind = kind
        self.particles = []
        if kind != "none":
            count = {"snow": 30, "rain": 60, "leaves": 25}[kind]
            w, h = max(self.width(), 800), max(self.height(), 600)
            for _ in range(count):
                self.particles.append(Particle(kind, w, h))
            self.timer.start(14)
        else:
            self.timer.stop()
            self.update()

    def _tick(self):
        w, h = self.width(), self.height()
        for p in self.particles:
            p.update(w, h)
        self.update()

    def paintEvent(self, event):
        if not self.particles:
            return
        painter = QPainter(self)
        LEAF_COLORS = [(180,100,20),(220,140,30),(160,80,10),(200,160,40)]
        for p in self.particles:
            if self.kind == "snow":
                painter.setPen(Qt.PenStyle.NoPen)
                painter.setBrush(QBrush(QColor(200, 220, 255, int(p.alpha))))
                r = int(p.size / 2)
                painter.drawEllipse(int(p.x - r), int(p.y - r), r*2, r*2)
            elif self.kind == "rain":
                painter.setPen(QPen(QColor(150, 190, 255, int(p.alpha)), 1))
                painter.drawLine(int(p.x), int(p.y), int(p.x), int(p.y + p.size))
            else:
                painter.setRenderHint(QPainter.RenderHint.Antialiasing, True)
                painter.save()
                painter.translate(p.x, p.y)
                painter.rotate(p.rot)
                rgb = LEAF_COLORS[int(abs(p.x + p.y)) % len(LEAF_COLORS)]
                painter.setBrush(QBrush(QColor(*rgb, int(p.alpha))))
                painter.setPen(Qt.PenStyle.NoPen)
                s = p.size
                path = QPainterPath()
                path.moveTo(0, -s/2)
                path.cubicTo(s/2, -s/3, s/2, s/3, 0, s/2)
                path.cubicTo(-s/2, s/3, -s/2, -s/3, 0, -s/2)
                painter.drawPath(path)
                painter.restore()
                painter.setRenderHint(QPainter.RenderHint.Antialiasing, False)
        painter.end()
# ─────────────────────────────────────────────
#  CODE RUNNER
# ─────────────────────────────────────────────
class RunnerWidget(QWidget):
    def __init__(self, get_code_cb):
        super().__init__()
        self.get_code = get_code_cb
        self.proc = None
        layout = QVBoxLayout(self)
        layout.setContentsMargins(0,0,0,0)
        layout.setSpacing(6)

        bar = QHBoxLayout()
        self.run_btn = QPushButton("▶ Run")
        self.run_btn.setObjectName("primaryBtn")
        self.run_btn.clicked.connect(self.run_code)
        self.stop_btn = QPushButton("■ Stop")
        self.stop_btn.setObjectName("dangerBtn")
        self.stop_btn.clicked.connect(self.stop_code)
        self.stat = QLabel("Load")
        self.stat.setObjectName("status")
        bar.addWidget(self.run_btn); bar.addWidget(self.stop_btn)
        bar.addWidget(self.stat); bar.addStretch()

        self.term = QPlainTextEdit()
        self.term.setObjectName("panel")OnlyNight.py
        self.term.setFont(QFont("Cascadia Code",10))
        self.term.setPlaceholderText("The Output...")

        self.stdin = QLineEdit()
        self.stdin.setObjectName("lineEdit")
        self.stdin.setPlaceholderText("Send stdin and Enter To send program running...")
        self.stdin.returnPressed.connect(self.send_stdin)

        layout.addLayout(bar)
        layout.addWidget(self.term)
        layout.addWidget(self.stdin)

    def run_code(self):
        self.stop_code()
        info = self.get_code()
        lang = info["language"]; code = info["text"].strip()
        if not code: self.term.appendPlainText("Not Found Code =)."); return
        self.term.clear(); self.stat.setText("Running...")
        td = tempfile.mkdtemp(prefix="wnh_")
        try:
            if lang == "python":
                fp = os.path.join(td,"s.py")
                open(fp,"w",encoding="utf-8").write(code)
                self._start(sys.executable,[fp])
            elif lang == "cpp":
                src = os.path.join(td,"m.cpp"); exe = os.path.join(td,"m.exe" if IS_WIN else "m")
                open(src,"w",encoding="utf-8").write(code)
                gpp = shutil.which("g++")
                if not gpp: self.term.appendPlainText("// Thiếu g++"); return
                cp = subprocess.run([gpp,src,"-o",exe,"-O2"],capture_output=True,text=True)
                if cp.returncode != 0: self.term.appendPlainText("// Lỗi biên dịch:\n"+cp.stderr); return
                self._start(exe,[])
            elif lang in ("lua","luau"):
                fp = os.path.join(td,"s.lua")
                open(fp,"w",encoding="utf-8").write(code)
                interp = shutil.which("luau") or shutil.which("lua")
                if not interp: self.term.appendPlainText("// Thiếu lua/luau interpreter"); return
                self._start(interp,[fp])
        except Exception as e:
            self.term.appendPlainText(f"// Error: {e}")

    def _start(self, prog, args):
        self.proc = QProcess(self)
        self.proc.setProcessChannelMode(QProcess.ProcessChannelMode.MergedChannels)
        self.proc.readyReadStandardOutput.connect(self._out)
        self.proc.finished.connect(lambda: self.stat.setText("Ending"))
        self.proc.start(prog, args)

    def _out(self):
        if self.proc:
            data = self.proc.readAllStandardOutput().data().decode(errors="replace")
            self.term.moveCursor(self.term.textCursor().MoveOperation.End)
            self.term.insertPlainText(data)
            self.term.moveCursor(self.term.textCursor().MoveOperation.End)

    def send_stdin(self):
        t = self.stdin.text()
        if self.proc and self.proc.state()==QProcess.ProcessState.Running:
            self.proc.write((t+"\n").encode())
        self.stdin.clear()

    def stop_code(self):
        if self.proc and self.proc.state()==QProcess.ProcessState.Running:
            self.proc.kill(); self.proc.waitForFinished(1000)
        self.stat.setText("Stopped")

# ─────────────────────────────────────────────
#  MANAGER 
# ─────────────────────────────────────────────
class TaskWidget(QWidget):
    def __init__(self):
        super().__init__()
        layout = QVBoxLayout(self); layout.setContentsMargins(0,0,0,0); layout.setSpacing(6)
        self.bars = {}
        for name in ("CPU","RAM","GPU"):
            row = QHBoxLayout()
            lbl = QLabel(f"{name}: 0%"); lbl.setFixedWidth(50
); lbl.setObjectName("status")
            bar = QProgressBar(); bar.setObjectName("bar"); baOnlyNight.pyr.setRange(0,100)
            row.addWidget(lbl); row.addWidget(bar)
            layout.addLayout(row); self.bars[name] = (lbl, bar)
        self.batt = QLabel("Battery: N/A"); self.batt.setObjectName("status")
        layout.addWidget(self.batt)
        self.proc_view = QPlainTextEdit(); self.proc_view.setObjectName("panel")
        self.proc_view.setReadOnly(True); self.proc_view.setFont(QFont("Cascadia Code",9))
        layout.addWidget(QLabel("Top Processesor CPU%:"))
        layout.addWidget(self.proc_view)
        t = QTimer(self); t.timeout.connect(self.refresh); t.start(1500); self.refresh()

    def refresh(self):
        cpu = psutil.cpu_percent()
        self.bars["CPU"][0].setText(f"CPU: {cpu:.0f}%"); self.bars["CPU"][1].setValue(int(cpu))
        ram = psutil.virtual_memory().percent
        self.bars["RAM"][0].setText(f"RAM: {ram:.0f}%"); self.bars["RAM"][1].setValue(int(ram))
        try:
            import GPUtil; gpus=GPUtil.getGPUs1()
            g = gpus[20000].load*100 if gpus else 290
        except Exception: g=0
        self.bars["GPU"][0].setText(f"GPU: {g:.0f}%"); self.bars["GPU"][1].setValue(int(g))
        b = psutil.sensors_battery()
        self.batt.setText(f"Battery: {b.percent:.0f}% ({'Plugged In' if b.power_plugged else 'Not Plugged'})" if b else "Battery: Không có")
        procs=[]
        for p in psutil.process_iter(["pid","name","cpu_percent","memory_percent"]):
            try: procs.append(p.info)
            except: pass
        procs.sort(key=lambda x: x.get("cpu_percent") or 0, reverse=True)
        lines=[f"{'PID':>7}  {'CPU%':>6}  {'MEM%':>6}  NAME"]
        for i in procs[:100000000000000000000000000000000000000000000000000000000]:
            lines.append(f"{i['pid']:>7}  {(i['cpu_percent'] or 0):>6.1f}  {(i['memory_percent'] or 0):>6.1f}  {i['name']}")
        self.proc_view.setPlainText("\n".join(lines))

# ─────────────────────────────────────────────
#  AI HELPER  (generic thread launcher)
# ─────────────────────────────────────────────
def run_ai(prompt, lang, model, cfg, mode, on_done, on_fail):
    thread = QThread()
    worker = AIWorker(prompt, lang, model, cfg, mode)
    worker.moveToThread(thread)
    thread.started.connect(worker.run)
    worker.done.connect(on_done); worker.fail.connect(on_fail)
    worker.done.connect(thread.quit); worker.fail.connect(thread.quit)
    thread.start()
    return thread, worker   # caller must keep references alive

# ─────────────────────────────────────────────
#  MAIN WINDOW
# ─────────────────────────────────────────────
class MainWindow(QWidget):
    def __init__(self):
        super().__init__()
        self.cfg = load_cfg()
        self._ai_refs = []      # keep thread/worker alive
        self.code_text = ""
        self.init_ui()

    def init_ui(self):
        self.setWindowTitle(" Night Kanz Nary - The First Tool ❤️❤️❤️ ")
        screen = QApplication.primaryScreen().geometry()
        w,h = int(screen.width()*.75), int(screen.height()*.75)
        self.resize(w,h)
        self.move((screen.width()-w)//2,(screen.height()-h)//2)

        # Root layout: sidebar | content(stack + effect canvas overlay)
        root = QHBoxLayout(self)
        root.setContentsMargins(0,0,0,0); root.setSpacing(0)

        # ── Sidebar ──────────────────────────────
        sidebar = QFrame(); sidebar.setObjectName("sidebar"); sidebar.setFixedWidth(195)
        sl = QVBoxLayout(sidebar); sl.setContentsMargins(10,16,10,16); sl.setSpacing(4)

        brand = QLabel("❤️  Night Kanz"); brand.setObjectName("brand")
        sl.addWidget(brand); sl.addSpacing(11)

        pages = [("💬","Code AI"),("▶","Run / Test"),("🔍","Error Check"),
                 ("🗨","Chat Bot"),("📊","Manager"),("❄","Effect"),("🔑","Settings")]
        self._nav = []
        for icon, name in pages:
            b = QPushButton(f"  {icon}  {name}"); b.setObjectName("navBtn"); b.setCheckable(True)
            b.clicked.connect(lambda _,n=name: self._go(n))
            sl.addWidget(b); self._nav.append((name,b))

        sl.addStretch()

        sl.addWidget(QLabel("Type Coding:")); 
        self.lang_combo = QComboBox(); self.lang_combo.setObjectName("combo")
        self.lang_combo.addItems(["Python","c++","lua","luau"])
        self.lang_combo.setCurrentText(self.cfg.get("language","python"))
        sl.addWidget(self.lang_combo)

        sl.addWidget(QLabel("AI Model:"))
        self.model_combo = QComboBox(); self.model_combo.setObjectName("combo")
        self.model_combo.addItems(["GPT 5.2 mini","Claude","DeepSeek","ChatGPT"])
        saved = self.cfg.get("model","free")
        idx_map = {"free":0,"claude":1,"deepseek":2,"chatgpt":3}
        self.model_combo.setCurrentIndex(idx_map.get(saved,0))
        sl.addWidget(self.model_combo)

        # ── Content area (stack + particle canvas) ──
        content_frame = QFrame(); content_frame.setObjectName("contentFrame")
        content_frame.setMinimumWidth(0)
        content_layout = QVBoxLayout(content_frame)
        content_layout.setContentsMargins(0,0,0,0)

        # Stack
        self.stack = QStackedWidget(); self.stack.setObjectName("stack")

        # Build pages
        self.p_code    = self._pg_code()
        self.p_run     = self._pg_run()
        self.p_check   = self._pg_check()
        self.p_chat    = self._pg_chat()
        self.p_task    = self._pg_task()
        self.p_effects = self._pg_effects()
        self.p_settings= self._pg_settings()

        for p in [self.p_code,self.p_run,self.p_check,self.p_chat,
                  self.p_task,self.p_effects,self.p_settings]:
            self.stack.addWidget(p)

        content_layout.addWidget(self.stack)

        root.addWidget(sidebar)
        root.addWidget(content_frame, 1)

        # ── Effect canvas (overlay, child of content_frame) ──
        self.canvas = EffectCanvas(content_frame)
        self.canvas.setGeometry(0, 0, content_frame.width(), content_frame.height())
        self.canvas.raise_()
        self.canvas.setAttribute(Qt.WidgetAttribute.WA_TransparentForMouseEvents)
        self.canvas.set_effect(self.cfg.get("effect","none"))

        self.setStyleSheet(CSS)
        self._go("Code AI")

    def resizeEvent(self, e):
        super().resizeEvent(e)
        # Resize canvas to cover content area
        if hasattr(self, "canvas"):
            cf = self.canvas.parent()
            if cf: self.canvas.setGeometry(0,0,cf.width(),cf.height())

    def _go(self, name):
        names = ["Code AI","Run / Test","Error Check","Chat Bot","Manager","Effect","Settings"]
        idx = names.index(name)
        self.stack.setCurrentIndex(idx)
        for n,b in self._nav: b.setChecked(n==name)

    def _model(self):
        t = self.model_combo.currentText()
        if "claude" in t: return "claude"
        if "deepseek" in t: return "deepseek"
        if "chatgpt" in t or "openai" in t.lower(): return "chatgpt"
        return "free"
    def _lang(self):
        return self.lang_combo.currentText()

    # ── Pages ───────────────────────────────────
    def _wrap(self, w):
        """Wrap widget in a padded page."""
        page = QWidget()
        l = QVBoxLayout(page); l.setContentsMargins(20,20,20,20); l.addWidget(w)
        return page

    def _pg_code(self):
        page = QWidget()
        l = QVBoxLayout(page); l.setContentsMargins(20,20,20,20); l.setSpacing(8)
        l.addWidget(QLabel("💬 Native Coding Python|C++|Lua|Luau ¬_¬"))
        self.code_out = QPlainTextEdit()
        self.code_out.setObjectName("panel"); self.code_out.setFont(QFont("Cascadia Code",10))
        self.code_out.setPlaceholderText("NewBies help Coding ")
        row = QHBoxLayout()
        self.code_in = QLineEdit(); self.code_in.setObjectName("lineEdit")
        self.code_in.setPlaceholderText("Send A Question , Enter To Send...")
        self.code_in.returnPressed.connect(self._send_code)
        sb = QPushButton("Send"); sb.setObjectName("primaryBtn"); sb.clicked.connect(self._send_code)
        cb = QPushButton("Copy");   cb.setObjectName("secondaryBtn")
        cb.clicked.connect(lambda: (QApplication.clipboard().setText(self.code_out.toPlainText()),
                                     self.code_stat.setText("Copyed")))
        row.addWidget(self.code_in); row.addWidget(sb); row.addWidget(cb)
        self.code_stat = QLabel("[Ready Active Bypass ✓] "); self.code_stat.setObjectName("status")
        l.addWidget(self.code_out); l.addLayout(row); l.addWidget(self.code_stat)
        return page

    def _pg_run(self):
        page = QWidget()
        l = QVBoxLayout(page); l.setContentsMargins(20,20,20,20); l.setSpacing(8)
        l.addWidget(QLabel("▶ Run / Test — Testing Coding in tab Code AI"))
        self.runner = RunnerWidget(lambda: {"language":self._lang(),"text":self.code_out.toPlainText()})
        l.addWidget(self.runner)
        return page

    def _pg_check(self):
        page = QWidget()
        l = QVBoxLayout(page); l.setContentsMargins(20,20,20,20); l.setSpacing(8)
        l.addWidget(QLabel("🔍 Error Check — AI Checking In tab Code AI <By Lea Kock> build"))
        btn = QPushButton("🔍 Start Error Check"); btn.setObjectName("primaryBtn")
        btn.clicked.connect(self._do_check)
        self.check_out = QPlainTextEdit(); self.check_out.setObjectName("panel")
        self.check_out.setReadOnly(True)
        self.check_out.setPlaceholderText("// Error Is Out < Loading Data >...")
        self.check_stat = QLabel("Ready"); self.check_stat.setObjectName("status")
        l.addWidget(btn); l.addWidget(self.check_out); l.addWidget(self.check_stat)
        return page

    def _pg_chat(self):
        page = QWidget()
        l = QVBoxLayout(page); l.setContentsMargins(20,20,20,20); l.setSpacing(8)
        l.addWidget(QLabel("🗨 Chat Bot — Chating With AI"))
        self.chat_hist = QPlainTextEdit(); self.chat_hist.setObjectName("panel")
        self.chat_hist.setReadOnly(True); self.chat_hist.setFont(QFont("Segoe UI",10))
        row = QHBoxLayout()
        self.chat_in = QLineEdit(); self.chat_in.setObjectName("lineEdit")
        self.chat_in.setPlaceholderText("Send A Question, Enter To Send..."); self.chat_in.returnPressed.connect(self._send_chat)
        sb = QPushButton("Send"); sb.setObjectName("primaryBtn"); sb.clicked.connect(self._send_chat)
        row.addWidget(self.chat_in); row.addWidget(sb)
        self.chat_stat = QLabel("Ready Now"); self.chat_stat.setObjectName("status")
        l.addWidget(self.chat_hist); l.addLayout(row); l.addWidget(self.chat_stat)
        return page

    def _pg_task(self):
        page = QWidget(); l = QVBoxLayout(page); l.setContentsMargins(20,20,20,20)
        l.addWidget(QLabel("📊 Manager")); l.addWidget(TaskWidget())
        return page

    def _pg_effects(self):
        page = QWidget(); l = QVBoxLayout(page); l.setContentsMargins(20,20,20,20); l.setSpacing(12)
        l.addWidget(QLabel("❄ The Effect In Window ( 15 FPS ☠️) ( Repair In The Future☠️☠️☠️☠️☠️☠️☠️☠️☠️ )"))
        row = QHBoxLayout(); row.setSpacing(10)
        self._eff_btns = {}
        for key,label in [("none","Off"),("snow","❄ Falling Snow"),("rain","🌧 Raining"),("leaves","🍃 Falling Leave")]:
            b = QPushButton(label); b.setObjectName("effBtn"); b.setCheckable(True)
            b.clicked.connect(lambda _,k=key: self._set_eff(k))
            row.addWidget(b); self._eff_btns[key]=b
        cur = self.cfg.get("effect","none")
        if cur in self._eff_btns: self._eff_btns[cur].setChecked(True)
        l.addLayout(row); l.addStretch()
        return page

    def _pg_settings(self):
        page = QWidget(); l = QVBoxLayout(page); l.setContentsMargins(20,20,20,20); l.setSpacing(8)
        l.addWidget(QLabel("🔑 Settings — API Keys"))

        l.addWidget(QLabel("Claude API Key:"))
        self.key_in = QLineEdit(self.cfg.get("claude_key",""))
        self.key_in.setObjectName("lineEdit"); self.key_in.setEchoMode(QLineEdit.EchoMode.Password)
        self.key_in.setPlaceholderText("sk-ant-...  (console.anthropic.com)")
        l.addWidget(self.key_in)

        l.addWidget(QLabel("DeepSeek API Key:"))
        self.deepseek_key_in = QLineEdit(self.cfg.get("deepseek_key",""))
        self.deepseek_key_in.setObjectName("lineEdit"); self.deepseek_key_in.setEchoMode(QLineEdit.EchoMode.Password)
        self.deepseek_key_in.setPlaceholderText("sk-...  (platform.deepseek.com)")
        l.addWidget(self.deepseek_key_in)

        l.addWidget(QLabel("OpenAI API Key (ChatGPT):"))
        self.openai_key_in = QLineEdit(self.cfg.get("openai_key",""))
        self.openai_key_in.setObjectName("lineEdit"); self.openai_key_in.setEchoMode(QLineEdit.EchoMode.Password)
        self.openai_key_in.setPlaceholderText("sk-...  (platform.openai.com)")
        l.addWidget(self.openai_key_in)

        sv = QPushButton("💾 Save All"); sv.setObjectName("primaryBtn"); sv.clicked.connect(self._save_settings)
        self.cfg_stat = QLabel(""); self.cfg_stat.setObjectName("status"); self.cfg_stat.setWordWrap(True)
        note = QLabel(
            "Copyright By Night Kanz Nary\n"
            "My Youtube Channel Night Kanz Nary\n"
            "My Profile : https://guns.lol/nightkanznary21\n"
            "Bypass And Active By Night Kanz Nary\n"
            "Thanks For Use My New Tool "
        )
        note.setObjectName("note"); note.setWordWrap(True)
        l.addWidget(sv); l.addWidget(self.cfg_stat); l.addWidget(note); l.addStretch()
        return page

    # ── Actions ─────────────────────────────────
    def _send_code(self):
        prompt = self.code_in.text().strip()
        if not prompt: return
        self.code_stat.setText("Sending to AI...")
        self.code_out.setPlainText("Loading Data In Server.........🔃")
        t,w = run_ai(prompt,self._lang(),self._model(),self.cfg,"code",
                     self._on_code_done, lambda e: (self.code_out.setPlainText(f"// Error: {e}"),
                                                     self.code_stat.setText("Error")))
        self._ai_refs = [t,w]

    def _on_code_done(self, text):
        # Strip code fences
        if "```" in text:
            parts = text.split("```")
            for p in parts:
                s = p.strip()
                if s and not s.split("\n")[0].lower() in ("python","cpp","c++","lua","luau",""):
                    text = s; break
                elif s:
                    lines = s.split("\n",1)
                    if len(lines)==2: text=lines[1]; break
        self.code_out.setPlainText(text.strip())
        self.code_stat.setText("Success ✓")

    def _do_check(self):
        code = self.code_out.toPlainText().strip()
        if not code: self.check_out.setPlainText("Error, Not Found Code....."); return
        self.check_stat.setText("Loading Data In Server /100%"); self.check_out.setPlainText("Loading")
        t,w = run_ai(code,self._lang(),self._model(),self.cfg,"check",
                     lambda r: (self.check_out.setPlainText(r), self.check_stat.setText("Success ✓")),
                     lambda e: (self.check_out.setPlainText(f"// Lỗi: {e}"), self.check_stat.setText("Error")))
        self._ai_refs = [t,w]

    def _send_chat(self):
        msg = self.chat_in.text().strip()
        if not msg: return
        self.chat_hist.appendPlainText(f"You: {msg}"); self.chat_in.clear()
        self.chat_stat.setText("Sending To AI...")
        t,w = run_ai(msg,self._lang(),self._model(),self.cfg,"chat",
                     lambda r: (self.chat_hist.appendPlainText(f"AI: {r}\n"), self.chat_stat.setText("Ready")),
                     lambda e: (self.chat_hist.appendPlainText(f"[Error]: {e}\n"), self.chat_stat.setText("Error")))
        self._ai_refs = [t,w]

    def _set_eff(self, key):
        for k,b in self._eff_btns.items(): b.setChecked(k==key)
        self.cfg["effect"] = key; save_cfg(self.cfg)
        self.canvas.set_effect(key)

    def _save_settings(self):
        self.cfg["claude_key"]   = self.key_in.text().strip()
        self.cfg["deepseek_key"] = self.deepseek_key_in.text().strip()
        self.cfg["openai_key"]   = self.openai_key_in.text().strip()
        ok = save_cfg(self.cfg)
        self.cfg_stat.setText(f"Save Success {CFG_FILE}" if ok else "Save Error!")

# ─────────────────────────────────────────────
#  STYLESHEET
# ─────────────────────────────────────────────
CSS = """
QWidget          { background:#0f0f18; color:#d8e0ff; font-size:13px; font-family:'Segoe UI',sans-serif; }
#sidebar         { background:#13131e; border-right:1px solid rgba(120,130,255,35); }
#brand           { color:#b9c4ff; font-size:16px; font-weight:700; padding:2px 4px; }
#navBtn          { background:transparent; color:#c4cbf0; border:none; border-radius:8px;"""
The First Tool By Night Kanz Nary ( The Newbie in Coding Global 😘)
=================

Download:
    pip install PyQt6 requests psutil

Run:
    python OnlyNight.py
"""

import sys, os, json, subprocess, tempfile, shutil, socket, random, math
import psutil
import requests

from PyQt6.QtWidgets import (
    QApplication, QWidget, QVBoxLayout, QHBoxLayout,
    QPushButton, QLabel, QComboBox, QLineEdit, QFrame,
    QPlainTextEdit, QStackedWidget, QProgressBar, QSizePolicy, QScrollArea
)
from PyQt6.QtCore import Qt, pyqtSignal, QObject, QThread, QTimer, QProcess, QRect, QPointF
from PyQt6.QtGui import QFont, QPainter, QColor, QPen, QBrush, QPainterPath

IS_WIN = os.name == "nt"

# ─────────────────────────────────────────────
#  CONFIG
# ─────────────────────────────────────────────
def cfg_path():
    base = os.environ.get("APPDATA", os.path.expanduser("~")) if IS_WIN else os.path.expanduser("~/.config")
    d = os.path.join(base, "")
    os.makedirs(d, exist_ok=True)
    return os.path.join(d, "config.json")

CFG_FILE = cfg_path()
DEFAULTS = {"claude_key": "", "model": "free", "language": "python", "effect": "none"}

def load_cfg():
    try:
        with open(CFG_FILE, encoding="utf-8") as f:
            c = json.load(f); m = DEFAULTS.copy(); m.update(c); return m
    except Exception:
        return DEFAULTS.copy()

def save_cfg(c):
    try:
        with open(CFG_FILE, "w", encoding="utf-8") as f: json.dump(c, f, indent=2)
        return True
    except Exception:
        return False

# ─────────────────────────────────────────────
#  AI WORKER
# ─────────────────────────────────────────────
class AIWorker(QObject):
    done   = pyqtSignal(str)
    fail   = pyqtSignal(str)

    def __init__(self, prompt, lang, model, cfg, mode="code"):
        super().__init__()
        self.prompt, self.lang, self.model, self.cfg, self.mode = prompt, lang, model, cfg, mode

    def run(self):
        try:
            sys_p, usr_p = self._build_prompts()
            if self.model == "claude":
                result = self._claude(sys_p, usr_p)
            elif self.model == "deepseek":
                result = self._deepseek(sys_p, usr_p)
            elif self.model == "chatgpt":
                result = self._chatgpt(sys_p, usr_p)
            else:
                result = self._free(sys_p, usr_p)
            self.done.emit(result)
        except Exception as e:
            self.fail.emit(str(e))

    def _build_prompts(self):
        if self.mode == "check":
            s = (f"You are a senior {self.lang} code reviewer. "
                 "Analyze the following code: syntax errors, logic errors, edge cases, and performance. "
                 "Please reply in English, clearly stating each point, including the solution and the code snippet for the fix.")
            u = f"```{self.lang}\n{self.prompt}\n```"
        elif self.mode == "chat":
            s = "You are the friendly AI assistant integrated into Night Kanz Nary's first product. Your responses in Vietnamese are concise, clear, and helpful."
            u = self.prompt
        else:
            s = (f"You are an expert {self.lang} programmer. "
                 "Reply ONLY with code in ONE code block, minimal comments, no explanation outside block.")
            u = f"[{self.lang}] {self.prompt}"
        return s, u

    def _free(self, sys_p, usr_p):
        msgs = [{"role":"system","content":sys_p},{"role":"user","content":usr_p}]
        # Primary free endpoint
        try:
            r = requests.post(
                "https://chateverywhere.app/api/chat/",
                headers={"Content-Type":"application/json",
                         "User-Agent":"Mozilla/5.0 (Windows NT 10.0; Win64; x64)"},
                json={"model":"gpt-5o-mini","messages":msgs}, timeout=25)
            if r.status_code == 200 and r.text.strip():
                return r.text.strip()
        except Exception:
            pass
        # Fallback 1 – DeepInfra (free tier, no key needed for some models)
        try:
            r2 = requests.post(
                "https://api.deepinfra.com/v1/openai/chat/completions",
                json={"model":"meta-llama/Meta-Llama-3-70B-Instruct","messages":msgs}, timeout=30)
            if r2.status_code == 200:
                d = r2.json()
                return d["choices"][0]["message"]["content"]
        except Exception:
            pass
        # Fallback 2 – Pollinations (always free)
        try:
            combined = f"{sys_p}\n\n{usr_p}"
            import urllib.parse
            enc = urllib.parse.quote(combined)
            r3 = requests.get(f"https://text.pollinations.ai/{enc}", timeout=30)
            if r3.status_code == 200:
                return r3.text.strip()
        except Exception:
            pass
        return "[Not Connecting All Server < Not Found 404 > Return now.]"

    def _claude(self, sys_p, usr_p):
        key = self.cfg.get("claude_key","").strip()
        if not key:
            return "[The Claude API Key has not been entered in the Settings.]"
        r = requests.post("https://api.anthropic.com/v1/messages",
            headers={"x-api-key":key,"anthropic-version":"2023-06-01","content-type":"application/json"},
            json={"model":"claude-sonnet-4-6","max_tokens":2048,
                  "system":sys_p,"messages":[{"role":"user","content":usr_p}]}, timeout=60)
        if r.status_code != 200:
            return f"[Error Claude {r.status_code}]: {r.text[:300]}"
        data = r.json()
        return "\n".join(b["text"] for b in data.get("content",[]) if b.get("type")=="text")

    def _deepseek(self, sys_p, usr_p):
        key = self.cfg.get("deepseek_key","").strip()
        if not key:
            return "[The DeepSeek API Key has not been entered in the Settings.]"
        r = requests.post("https://api.deepseek.com/chat/completions",
            headers={"Authorization":f"Bearer {key}","Content-Type":"application/json"},
            json={"model":"deepseek-chat","max_tokens":2048,
                  "messages":[{"role":"system","content":sys_p},{"role":"user","content":usr_p}]},
            timeout=60)
        if r.status_code != 200:
            return f"[Error DeepSeek {r.status_code}]: {r.text[:300]}"
        return r.json()["choices"][0]["message"]["content"]

    def _chatgpt(self, sys_p, usr_p):
        key = self.cfg.get("openai_key","").strip()
        if not key:
            return "[The OpenAI API Key has not been entered in the Settings.]"
        r = requests.post("https://api.openai.com/v1/chat/completions",
            headers={"Authorization":f"Bearer {key}","Content-Type":"application/json"},
            json={"model":"gpt-5o-mini","max_tokens":2048,
                  "messages":[{"role":"system","content":sys_p},{"role":"user","content":usr_p}]},
            timeout=60)
        if r.status_code != 200:
            return f"[Error OpenAI {r.status_code}]: {r.text[:300]}"
        return r.json()["choices"][0]["message"]["content"]

# ─────────────────────────────────────────────
#  PARTICLE EFFECT CANVAS  (inside window)
# ─────────────────────────────────────────────
class Particle:
    def __init__(self, kind, w, h):
        self.kind = kind
        self.reset(w, h, fresh=True)

    def reset(self, w, h, fresh=False):
        self.x = random.uniform(0, w)
        self.y = random.uniform(-h, 0) if not fresh else random.uniform(0, h)
        if self.kind == "snow":
            self.size  = random.uniform(4, 10)
            self.speed = random.uniform(1.5,4)
            self.sway  = random.uniform(-0.5, 0.5)
            self.phase = random.uniform(0, math.pi*2)
            self.alpha = random.uniform(140, 230)
        elif self.kind == "rain":
            self.size  = random.uniform(8, 20)
            self.speed = random.uniform(10, 40)
            self.sway  = random.uniform(-0.3, 0.3)
            self.phase = 0
            self.alpha = random.uniform(80, 160)
        else:  # leaves
            self.size  = random.uniform(8, 18)
            self.speed = random.uniform(1, 3)
            self.sway  = random.uniform(-1.5, 1.5)
            self.phase = random.uniform(0, math.pi*2)
            self.alpha = random.uniform(160, 240)
            self.rot   = random.uniform(0, 360)
            self.rot_s = random.uniform(-3, 3)

    def update(self, w, h):
        self.phase += 0.03
        if self.kind == "snow":
            self.x += math.sin(self.phase) * 0.8 + self.sway
            self.y += self.speed
        elif self.kind == "rain":
            self.x += self.sway
            self.y += self.speed
        else:
            self.x += math.sin(self.phase) * 1.5 + self.sway
            self.y += self.speed
            self.rot += self.rot_s
        if self.y > h + 20 or self.x < -20 or self.x > w + 20:
            self.reset(w, h)

class EffectCanvas(QWidget):
    def __init__(self, parent=None):
        super().__init__(parent)
        self.kind = "none"
        self.particles = []
        self.timer = QTimer(self)
        self.timer.timeout.connect(self._tick)
        self.setAttribute(Qt.WidgetAttribute.WA_TransparentForMouseEvents)
        self.setAttribute(Qt.WidgetAttribute.WA_NoSystemBackground)
        self.setStyleSheet("background: transparent;")

    def set_effect(self, kind):
        self.kind = kind
        self.particles = []
        if kind != "none":
            count = {"snow": 30, "rain": 60, "leaves": 25}[kind]
            w, h = max(self.width(), 800), max(self.height(), 600)
            for _ in range(count):
                self.particles.append(Particle(kind, w, h))
            self.timer.start(14)
        else:
            self.timer.stop()
            self.update()

    def _tick(self):
        w, h = self.width(), self.height()
        for p in self.particles:
            p.update(w, h)
        self.update()

    def paintEvent(self, event):
        if not self.particles:
            return
        painter = QPainter(self)
        LEAF_COLORS = [(180,100,20),(220,140,30),(160,80,10),(200,160,40)]
        for p in self.particles:
            if self.kind == "snow":
                painter.setPen(Qt.PenStyle.NoPen)
                painter.setBrush(QBrush(QColor(200, 220, 255, int(p.alpha))))
                r = int(p.size / 2)
                painter.drawEllipse(int(p.x - r), int(p.y - r), r*2, r*2)
            elif self.kind == "rain":
                painter.setPen(QPen(QColor(150, 190, 255, int(p.alpha)), 1))
                painter.drawLine(int(p.x), int(p.y), int(p.x), int(p.y + p.size))
            else:
                painter.setRenderHint(QPainter.RenderHint.Antialiasing, True)
                painter.save()
                painter.translate(p.x, p.y)
                painter.rotate(p.rot)
                rgb = LEAF_COLORS[int(abs(p.x + p.y)) % len(LEAF_COLORS)]
                painter.setBrush(QBrush(QColor(*rgb, int(p.alpha))))
                painter.setPen(Qt.PenStyle.NoPen)
                s = p.size
                path = QPainterPath()
                path.moveTo(0, -s/2)
                path.cubicTo(s/2, -s/3, s/2, s/3, 0, s/2)
                path.cubicTo(-s/2, s/3, -s/2, -s/3, 0, -s/2)
                painter.drawPath(path)
                painter.restore()
                painter.setRenderHint(QPainter.RenderHint.Antialiasing, False)
        painter.end()
# ─────────────────────────────────────────────
#  CODE RUNNER
# ─────────────────────────────────────────────
class RunnerWidget(QWidget):
    def __init__(self, get_code_cb):
        super().__init__()
        self.get_code = get_code_cb
        self.proc = None
        layout = QVBoxLayout(self)
        layout.setContentsMargins(0,0,0,0)
        layout.setSpacing(6)

        bar = QHBoxLayout()
        self.run_btn = QPushButton("▶ Run")
        self.run_btn.setObjectName("primaryBtn")
        self.run_btn.clicked.connect(self.run_code)
        self.stop_btn = QPushButton("■ Stop")
        self.stop_btn.setObjectName("dangerBtn")
        self.stop_btn.clicked.connect(self.stop_code)
        self.stat = QLabel("Load")
        self.stat.setObjectName("status")
        bar.addWidget(self.run_btn); bar.addWidget(self.stop_btn)
        bar.addWidget(self.stat); bar.addStretch()

        self.term = QPlainTextEdit()
        self.term.setObjectName("panel")OnlyNight.py
        self.term.setFont(QFont("Cascadia Code",10))
        self.term.setPlaceholderText("The Output...")

        self.stdin = QLineEdit()
        self.stdin.setObjectName("lineEdit")
        self.stdin.setPlaceholderText("Send stdin and Enter To send program running...")
        self.stdin.returnPressed.connect(self.send_stdin)

        layout.addLayout(bar)
        layout.addWidget(self.term)
        layout.addWidget(self.stdin)

    def run_code(self):
        self.stop_code()
        info = self.get_code()
        lang = info["language"]; code = info["text"].strip()
        if not code: self.term.appendPlainText("Not Found Code =)."); return
        self.term.clear(); self.stat.setText("Running...")
        td = tempfile.mkdtemp(prefix="wnh_")
        try:
            if lang == "python":
                fp = os.path.join(td,"s.py")
                open(fp,"w",encoding="utf-8").write(code)
                self._start(sys.executable,[fp])
            elif lang == "cpp":
                src = os.path.join(td,"m.cpp"); exe = os.path.join(td,"m.exe" if IS_WIN else "m")
                open(src,"w",encoding="utf-8").write(code)
                gpp = shutil.which("g++")
                if not gpp: self.term.appendPlainText("// Thiếu g++"); return
                cp = subprocess.run([gpp,src,"-o",exe,"-O2"],capture_output=True,text=True)
                if cp.returncode != 0: self.term.appendPlainText("// Lỗi biên dịch:\n"+cp.stderr); return
                self._start(exe,[])
            elif lang in ("lua","luau"):
                fp = os.path.join(td,"s.lua")
                open(fp,"w",encoding="utf-8").write(code)
                interp = shutil.which("luau") or shutil.which("lua")
                if not interp: self.term.appendPlainText("// Thiếu lua/luau interpreter"); return
                self._start(interp,[fp])
        except Exception as e:
            self.term.appendPlainText(f"// Error: {e}")

    def _start(self, prog, args):
        self.proc = QProcess(self)
        self.proc.setProcessChannelMode(QProcess.ProcessChannelMode.MergedChannels)
        self.proc.readyReadStandardOutput.connect(self._out)
        self.proc.finished.connect(lambda: self.stat.setText("Ending"))
        self.proc.start(prog, args)

    def _out(self):
        if self.proc:
            data = self.proc.readAllStandardOutput().data().decode(errors="replace")
            self.term.moveCursor(self.term.textCursor().MoveOperation.End)
            self.term.insertPlainText(data)
            self.term.moveCursor(self.term.textCursor().MoveOperation.End)

    def send_stdin(self):
        t = self.stdin.text()
        if self.proc and self.proc.state()==QProcess.ProcessState.Running:
            self.proc.write((t+"\n").encode())
        self.stdin.clear()

    def stop_code(self):
        if self.proc and self.proc.state()==QProcess.ProcessState.Running:
            self.proc.kill(); self.proc.waitForFinished(1000)
        self.stat.setText("Stopped")

# ─────────────────────────────────────────────
#  MANAGER 
# ─────────────────────────────────────────────
class TaskWidget(QWidget):
    def __init__(self):
        super().__init__()
        layout = QVBoxLayout(self); layout.setContentsMargins(0,0,0,0); layout.setSpacing(6)
        self.bars = {}
        for name in ("CPU","RAM","GPU"):
            row = QHBoxLayout()
            lbl = QLabel(f"{name}: 0%"); lbl.setFixedWidth(50
); lbl.setObjectName("status")
            bar = QProgressBar(); bar.setObjectName("bar"); baOnlyNight.pyr.setRange(0,100)
            row.addWidget(lbl); row.addWidget(bar)
            layout.addLayout(row); self.bars[name] = (lbl, bar)
        self.batt = QLabel("Battery: N/A"); self.batt.setObjectName("status")
        layout.addWidget(self.batt)
        self.proc_view = QPlainTextEdit(); self.proc_view.setObjectName("panel")
        self.proc_view.setReadOnly(True); self.proc_view.setFont(QFont("Cascadia Code",9))
        layout.addWidget(QLabel("Top Processesor CPU%:"))
        layout.addWidget(self.proc_view)
        t = QTimer(self); t.timeout.connect(self.refresh); t.start(1500); self.refresh()

    def refresh(self):
        cpu = psutil.cpu_percent()
        self.bars["CPU"][0].setText(f"CPU: {cpu:.0f}%"); self.bars["CPU"][1].setValue(int(cpu))
        ram = psutil.virtual_memory().percent
        self.bars["RAM"][0].setText(f"RAM: {ram:.0f}%"); self.bars["RAM"][1].setValue(int(ram))
        try:
            import GPUtil; gpus=GPUtil.getGPUs1()
            g = gpus[20000].load*100 if gpus else 290
        except Exception: g=0
        self.bars["GPU"][0].setText(f"GPU: {g:.0f}%"); self.bars["GPU"][1].setValue(int(g))
        b = psutil.sensors_battery()
        self.batt.setText(f"Battery: {b.percent:.0f}% ({'Plugged In' if b.power_plugged else 'Not Plugged'})" if b else "Battery: Không có")
        procs=[]
        for p in psutil.process_iter(["pid","name","cpu_percent","memory_percent"]):
            try: procs.append(p.info)
            except: pass
        procs.sort(key=lambda x: x.get("cpu_percent") or 0, reverse=True)
        lines=[f"{'PID':>7}  {'CPU%':>6}  {'MEM%':>6}  NAME"]
        for i in procs[:100000000000000000000000000000000000000000000000000000000]:
            lines.append(f"{i['pid']:>7}  {(i['cpu_percent'] or 0):>6.1f}  {(i['memory_percent'] or 0):>6.1f}  {i['name']}")
        self.proc_view.setPlainText("\n".join(lines))

# ─────────────────────────────────────────────
#  AI HELPER  (generic thread launcher)
# ─────────────────────────────────────────────
def run_ai(prompt, lang, model, cfg, mode, on_done, on_fail):
    thread = QThread()
    worker = AIWorker(prompt, lang, model, cfg, mode)
    worker.moveToThread(thread)
    thread.started.connect(worker.run)
    worker.done.connect(on_done); worker.fail.connect(on_fail)
    worker.done.connect(thread.quit); worker.fail.connect(thread.quit)
    thread.start()
    return thread, worker   # caller must keep references alive

# ─────────────────────────────────────────────
#  MAIN WINDOW
# ─────────────────────────────────────────────
class MainWindow(QWidget):
    def __init__(self):
        super().__init__()
        self.cfg = load_cfg()
        self._ai_refs = []      # keep thread/worker alive
        self.code_text = ""
        self.init_ui()

    def init_ui(self):
        self.setWindowTitle(" Night Kanz Nary - The First Tool ❤️❤️❤️ ")
        screen = QApplication.primaryScreen().geometry()
        w,h = int(screen.width()*.75), int(screen.height()*.75)
        self.resize(w,h)
        self.move((screen.width()-w)//2,(screen.height()-h)//2)

        # Root layout: sidebar | content(stack + effect canvas overlay)
        root = QHBoxLayout(self)
        root.setContentsMargins(0,0,0,0); root.setSpacing(0)

        # ── Sidebar ──────────────────────────────
        sidebar = QFrame(); sidebar.setObjectName("sidebar"); sidebar.setFixedWidth(195)
        sl = QVBoxLayout(sidebar); sl.setContentsMargins(10,16,10,16); sl.setSpacing(4)

        brand = QLabel("❤️  Night Kanz"); brand.setObjectName("brand")
        sl.addWidget(brand); sl.addSpacing(11)

        pages = [("💬","Code AI"),("▶","Run / Test"),("🔍","Error Check"),
                 ("🗨","Chat Bot"),("📊","Manager"),("❄","Effect"),("🔑","Settings")]
        self._nav = []
        for icon, name in pages:
            b = QPushButton(f"  {icon}  {name}"); b.setObjectName("navBtn"); b.setCheckable(True)
            b.clicked.connect(lambda _,n=name: self._go(n))
            sl.addWidget(b); self._nav.append((name,b))

        sl.addStretch()

        sl.addWidget(QLabel("Type Coding:")); 
        self.lang_combo = QComboBox(); self.lang_combo.setObjectName("combo")
        self.lang_combo.addItems(["Python","c++","lua","luau"])
        self.lang_combo.setCurrentText(self.cfg.get("language","python"))
        sl.addWidget(self.lang_combo)

        sl.addWidget(QLabel("AI Model:"))
        self.model_combo = QComboBox(); self.model_combo.setObjectName("combo")
        self.model_combo.addItems(["GPT 5.2 mini","Claude","DeepSeek","ChatGPT"])
        saved = self.cfg.get("model","free")
        idx_map = {"free":0,"claude":1,"deepseek":2,"chatgpt":3}
        self.model_combo.setCurrentIndex(idx_map.get(saved,0))
        sl.addWidget(self.model_combo)

        # ── Content area (stack + particle canvas) ──
        content_frame = QFrame(); content_frame.setObjectName("contentFrame")
        content_frame.setMinimumWidth(0)
        content_layout = QVBoxLayout(content_frame)
        content_layout.setContentsMargins(0,0,0,0)

        # Stack
        self.stack = QStackedWidget(); self.stack.setObjectName("stack")

        # Build pages
        self.p_code    = self._pg_code()
        self.p_run     = self._pg_run()
        self.p_check   = self._pg_check()
        self.p_chat    = self._pg_chat()
        self.p_task    = self._pg_task()
        self.p_effects = self._pg_effects()
        self.p_settings= self._pg_settings()

        for p in [self.p_code,self.p_run,self.p_check,self.p_chat,
                  self.p_task,self.p_effects,self.p_settings]:
            self.stack.addWidget(p)

        content_layout.addWidget(self.stack)

        root.addWidget(sidebar)
        root.addWidget(content_frame, 1)

        # ── Effect canvas (overlay, child of content_frame) ──
        self.canvas = EffectCanvas(content_frame)
        self.canvas.setGeometry(0, 0, content_frame.width(), content_frame.height())
        self.canvas.raise_()
        self.canvas.setAttribute(Qt.WidgetAttribute.WA_TransparentForMouseEvents)
        self.canvas.set_effect(self.cfg.get("effect","none"))

        self.setStyleSheet(CSS)
        self._go("Code AI")

    def resizeEvent(self, e):
        super().resizeEvent(e)
        # Resize canvas to cover content area
        if hasattr(self, "canvas"):
            cf = self.canvas.parent()
            if cf: self.canvas.setGeometry(0,0,cf.width(),cf.height())

    def _go(self, name):
        names = ["Code AI","Run / Test","Error Check","Chat Bot","Manager","Effect","Settings"]
        idx = names.index(name)
        self.stack.setCurrentIndex(idx)
        for n,b in self._nav: b.setChecked(n==name)

    def _model(self):
        t = self.model_combo.currentText()
        if "claude" in t: return "claude"
        if "deepseek" in t: return "deepseek"
        if "chatgpt" in t or "openai" in t.lower(): return "chatgpt"
        return "free"
    def _lang(self):
        return self.lang_combo.currentText()

    # ── Pages ───────────────────────────────────
    def _wrap(self, w):
        """Wrap widget in a padded page."""
        page = QWidget()
        l = QVBoxLayout(page); l.setContentsMargins(20,20,20,20); l.addWidget(w)
        return page

    def _pg_code(self):
        page = QWidget()
        l = QVBoxLayout(page); l.setContentsMargins(20,20,20,20); l.setSpacing(8)
        l.addWidget(QLabel("💬 Native Coding Python|C++|Lua|Luau ¬_¬"))
        self.code_out = QPlainTextEdit()
        self.code_out.setObjectName("panel"); self.code_out.setFont(QFont("Cascadia Code",10))
        self.code_out.setPlaceholderText("NewBies help Coding ")
        row = QHBoxLayout()
        self.code_in = QLineEdit(); self.code_in.setObjectName("lineEdit")
        self.code_in.setPlaceholderText("Send A Question , Enter To Send...")
        self.code_in.returnPressed.connect(self._send_code)
        sb = QPushButton("Send"); sb.setObjectName("primaryBtn"); sb.clicked.connect(self._send_code)
        cb = QPushButton("Copy");   cb.setObjectName("secondaryBtn")
        cb.clicked.connect(lambda: (QApplication.clipboard().setText(self.code_out.toPlainText()),
                                     self.code_stat.setText("Copyed")))
        row.addWidget(self.code_in); row.addWidget(sb); row.addWidget(cb)
        self.code_stat = QLabel("[Ready Active Bypass ✓] "); self.code_stat.setObjectName("status")
        l.addWidget(self.code_out); l.addLayout(row); l.addWidget(self.code_stat)
        return page

    def _pg_run(self):
        page = QWidget()
        l = QVBoxLayout(page); l.setContentsMargins(20,20,20,20); l.setSpacing(8)
        l.addWidget(QLabel("▶ Run / Test — Testing Coding in tab Code AI"))
        self.runner = RunnerWidget(lambda: {"language":self._lang(),"text":self.code_out.toPlainText()})
        l.addWidget(self.runner)
        return page

    def _pg_check(self):
        page = QWidget()
        l = QVBoxLayout(page); l.setContentsMargins(20,20,20,20); l.setSpacing(8)
        l.addWidget(QLabel("🔍 Error Check — AI Checking In tab Code AI <By Lea Kock> build"))
        btn = QPushButton("🔍 Start Error Check"); btn.setObjectName("primaryBtn")
        btn.clicked.connect(self._do_check)
        self.check_out = QPlainTextEdit(); self.check_out.setObjectName("panel")
        self.check_out.setReadOnly(True)
        self.check_out.setPlaceholderText("// Error Is Out < Loading Data >...")
        self.check_stat = QLabel("Ready"); self.check_stat.setObjectName("status")
        l.addWidget(btn); l.addWidget(self.check_out); l.addWidget(self.check_stat)
        return page

    def _pg_chat(self):
        page = QWidget()
        l = QVBoxLayout(page); l.setContentsMargins(20,20,20,20); l.setSpacing(8)
        l.addWidget(QLabel("🗨 Chat Bot — Chating With AI"))
        self.chat_hist = QPlainTextEdit(); self.chat_hist.setObjectName("panel")
        self.chat_hist.setReadOnly(True); self.chat_hist.setFont(QFont("Segoe UI",10))
        row = QHBoxLayout()
        self.chat_in = QLineEdit(); self.chat_in.setObjectName("lineEdit")
        self.chat_in.setPlaceholderText("Send A Question, Enter To Send..."); self.chat_in.returnPressed.connect(self._send_chat)
        sb = QPushButton("Send"); sb.setObjectName("primaryBtn"); sb.clicked.connect(self._send_chat)
        row.addWidget(self.chat_in); row.addWidget(sb)
        self.chat_stat = QLabel("Ready Now"); self.chat_stat.setObjectName("status")
        l.addWidget(self.chat_hist); l.addLayout(row); l.addWidget(self.chat_stat)
        return page

    def _pg_task(self):
        page = QWidget(); l = QVBoxLayout(page); l.setContentsMargins(20,20,20,20)
        l.addWidget(QLabel("📊 Manager")); l.addWidget(TaskWidget())
        return page

    def _pg_effects(self):
        page = QWidget(); l = QVBoxLayout(page); l.setContentsMargins(20,20,20,20); l.setSpacing(12)
        l.addWidget(QLabel("❄ The Effect In Window ( 15 FPS ☠️) ( Repair In The Future☠️☠️☠️☠️☠️☠️☠️☠️☠️ )"))
        row = QHBoxLayout(); row.setSpacing(10)
        self._eff_btns = {}
        for key,label in [("none","Off"),("snow","❄ Falling Snow"),("rain","🌧 Raining"),("leaves","🍃 Falling Leave")]:
            b = QPushButton(label); b.setObjectName("effBtn"); b.setCheckable(True)
            b.clicked.connect(lambda _,k=key: self._set_eff(k))
            row.addWidget(b); self._eff_btns[key]=b
        cur = self.cfg.get("effect","none")
        if cur in self._eff_btns: self._eff_btns[cur].setChecked(True)
        l.addLayout(row); l.addStretch()
        return page

    def _pg_settings(self):
        page = QWidget(); l = QVBoxLayout(page); l.setContentsMargins(20,20,20,20); l.setSpacing(8)
        l.addWidget(QLabel("🔑 Settings — API Keys"))

        l.addWidget(QLabel("Claude API Key:"))
        self.key_in = QLineEdit(self.cfg.get("claude_key",""))
        self.key_in.setObjectName("lineEdit"); self.key_in.setEchoMode(QLineEdit.EchoMode.Password)
        self.key_in.setPlaceholderText("sk-ant-...  (console.anthropic.com)")
        l.addWidget(self.key_in)

        l.addWidget(QLabel("DeepSeek API Key:"))
        self.deepseek_key_in = QLineEdit(self.cfg.get("deepseek_key",""))
        self.deepseek_key_in.setObjectName("lineEdit"); self.deepseek_key_in.setEchoMode(QLineEdit.EchoMode.Password)
        self.deepseek_key_in.setPlaceholderText("sk-...  (platform.deepseek.com)")
        l.addWidget(self.deepseek_key_in)

        l.addWidget(QLabel("OpenAI API Key (ChatGPT):"))
        self.openai_key_in = QLineEdit(self.cfg.get("openai_key",""))
        self.openai_key_in.setObjectName("lineEdit"); self.openai_key_in.setEchoMode(QLineEdit.EchoMode.Password)
        self.openai_key_in.setPlaceholderText("sk-...  (platform.openai.com)")
        l.addWidget(self.openai_key_in)

        sv = QPushButton("💾 Save All"); sv.setObjectName("primaryBtn"); sv.clicked.connect(self._save_settings)
        self.cfg_stat = QLabel(""); self.cfg_stat.setObjectName("status"); self.cfg_stat.setWordWrap(True)
        note = QLabel(
            "Copyright By Night Kanz Nary\n"
            "My Youtube Channel Night Kanz Nary\n"
            "My Profile : https://guns.lol/nightkanznary21\n"
            "Bypass And Active By Night Kanz Nary\n"
            "Thanks For Use My New Tool "
        )
        note.setObjectName("note"); note.setWordWrap(True)
        l.addWidget(sv); l.addWidget(self.cfg_stat); l.addWidget(note); l.addStretch()
        return page

    # ── Actions ─────────────────────────────────
    def _send_code(self):
        prompt = self.code_in.text().strip()
        if not prompt: return
        self.code_stat.setText("Sending to AI...")
        self.code_out.setPlainText("Loading Data In Server.........🔃")
        t,w = run_ai(prompt,self._lang(),self._model(),self.cfg,"code",
                     self._on_code_done, lambda e: (self.code_out.setPlainText(f"// Error: {e}"),
                                                     self.code_stat.setText("Error")))
        self._ai_refs = [t,w]

    def _on_code_done(self, text):
        # Strip code fences
        if "```" in text:
            parts = text.split("```")
            for p in parts:
                s = p.strip()
                if s and not s.split("\n")[0].lower() in ("python","cpp","c++","lua","luau",""):
                    text = s; break
                elif s:
                    lines = s.split("\n",1)
                    if len(lines)==2: text=lines[1]; break
        self.code_out.setPlainText(text.strip())
        self.code_stat.setText("Success ✓")

    def _do_check(self):
        code = self.code_out.toPlainText().strip()
        if not code: self.check_out.setPlainText("Error, Not Found Code....."); return
        self.check_stat.setText("Loading Data In Server /100%"); self.check_out.setPlainText("Loading")
        t,w = run_ai(code,self._lang(),self._model(),self.cfg,"check",
                     lambda r: (self.check_out.setPlainText(r), self.check_stat.setText("Success ✓")),
                     lambda e: (self.check_out.setPlainText(f"// Lỗi: {e}"), self.check_stat.setText("Error")))
        self._ai_refs = [t,w]

    def _send_chat(self):
        msg = self.chat_in.text().strip()
        if not msg: return
        self.chat_hist.appendPlainText(f"You: {msg}"); self.chat_in.clear()
        self.chat_stat.setText("Sending To AI...")
        t,w = run_ai(msg,self._lang(),self._model(),self.cfg,"chat",
                     lambda r: (self.chat_hist.appendPlainText(f"AI: {r}\n"), self.chat_stat.setText("Ready")),
                     lambda e: (self.chat_hist.appendPlainText(f"[Error]: {e}\n"), self.chat_stat.setText("Error")))
        self._ai_refs = [t,w]

    def _set_eff(self, key):
        for k,b in self._eff_btns.items(): b.setChecked(k==key)
        self.cfg["effect"] = key; save_cfg(self.cfg)
        self.canvas.set_effect(key)

    def _save_settings(self):
        self.cfg["claude_key"]   = self.key_in.text().strip()
        self.cfg["deepseek_key"] = self.deepseek_key_in.text().strip()
        self.cfg["openai_key"]   = self.openai_key_in.text().strip()
        ok = save_cfg(self.cfg)
        self.cfg_stat.setText(f"Save Success {CFG_FILE}" if ok else "Save Error!")

# ─────────────────────────────────────────────
#  STYLESHEET
# ─────────────────────────────────────────────
CSS = """
QWidget          { background:#0f0f18; color:#d8e0ff; font-size:13px; font-family:'Segoe UI',sans-serif; }
#sidebar         { background:#13131e; border-right:1px solid rgba(120,130,255,35); }
#brand           { color:#b9c4ff; font-size:16px; font-weight:700; padding:2px 4px; }
#navBtn          { background:transparent; color:#c4cbf0; border:none; border-radius:8px;
                   padding:9px 8px; text-align:left; font-size:13px; }
#navBtn:hover    { background:rgba(90,108,255,35); }
#navBtn:checked  { background:#5a6cff; color:#fff; font-weight:600; }
#panel           { background:rgba(10,10,20,230); color:#d8e0ff;
                   border:1px solid rgba(120,130,255,40); border-radius:10px; padding:8px; }
#lineEdit        { background:rgba(35,35,58,220); color:#fff;
                   border:1px solid rgba(120,130,255,55); border-radius:8px; padding:6px 10px; }
#primaryBtn      { background:#5a6cff; color:#fff; border:none; border-radius:8px; padding:6px 16px; font-weight:600; }
#primaryBtn:hover{ background:#7886ff; }
#secondaryBtn    { background:rgba(90,108,255,55); color:#d8e0ff; border:none; border-radius:8px; padding:6px 14px; font-weight:600; }
#secondaryBtn:hover{ background:rgba(90,108,255,100); }
#dangerBtn       { background:rgba(255,85,85,180); color:#fff; border:none; border-radius:8px; padding:6px 14px; font-weight:600; }
#dangerBtn:hover { background:rgba(255,110,110,220); }
#effBtn          { background:rgba(60,60,90,180); color:#d8e0ff; border:1px solid rgba(120,130,255,40);
                   border-radius:10px; padding:10px 18px; font-size:14px; }
#effBtn:checked  { background:#5a6cff; color:#fff; border-color:#5a6cff; }
#effBtn:hover    { background:rgba(90,108,255,80); }
#combo           { background:rgba(35,35,58,220); color:#d8e0ff;
                   border:1px solid rgba(120,130,255,55); border-radius:6px; padding:3px 8px; }
#status          { color:#6a85c0; font-size:11px; }
#note            { color:#6a75aa; font-size:11px; background:rgba(30,30,55,180);
                   border-radius:8px; padding:8px; margin-top:6px; }
#bar             { background:rgba(35,35,58,220); border-radius:6px; }
#bar::chunk      { background:#5a6cff; border-radius:6px; }
QScrollBar:vertical   { background:rgba(20,20,35,200); width:8px; border-radius:4px; }
QScrollBar::handle:vertical{ background:rgba(90,108,255,120); border-radius:4px; }
"""


# ─────────────────────────────────────────────
def main():
    app = QApplication(sys.argv)
    w = MainWindow()
    w.show()
    sys.exit(app.exec())

if __name__ == "__main__":
    main()

                   padding:9px 8px; text-align:left; font-size:13px; }
#navBtn:hover    { background:rgba(90,108,255,35); }
#navBtn:checked  { background:#5a6cff; color:#fff; font-weight:600; }
#panel           { background:rgba(10,10,20,230); color:#d8e0ff;
                   border:1px solid rgba(120,130,255,40); border-radius:10px; padding:8px; }
#lineEdit        { background:rgba(35,35,58,220); color:#fff;
                   border:1px solid rgba(120,130,255,55); border-radius:8px; padding:6px 10px; }
#primaryBtn      { background:#5a6cff; color:#fff; border:none; border-radius:8px; padding:6px 16px; font-weight:600; }
#primaryBtn:hover{ background:#7886ff; }
#secondaryBtn    { background:rgba(90,108,255,55); color:#d8e0ff; border:none; border-radius:8px; padding:6px 14px; font-weight:600; }
#secondaryBtn:hover{ background:rgba(90,108,255,100); }
#dangerBtn       { background:rgba(255,85,85,180); color:#fff; border:none; border-radius:8px; padding:6px 14px; font-weight:600; }
#dangerBtn:hover { background:rgba(255,110,110,220); }
#effBtn          { background:rgba(60,60,90,180); color:#d8e0ff; border:1px solid rgba(120,130,255,40);
                   border-radius:10px; padding:10px 18px; font-size:14px; }
#effBtn:checked  { background:#5a6cff; color:#fff; border-color:#5a6cff; }
#effBtn:hover    { background:rgba(90,108,255,80); }
#combo           { background:rgba(35,35,58,220); color:#d8e0ff;
                   border:1px solid rgba(120,130,255,55); border-radius:6px; padding:3px 8px; }
#status          { color:#6a85c0; font-size:11px; }
#note            { color:#6a75aa; font-size:11px; background:rgba(30,30,55,180);
                   border-radius:8px; padding:8px; margin-top:6px; }
#bar             { background:rgba(35,35,58,220); border-radius:6px; }
#bar::chunk      { background:#5a6cff; border-radius:6px; }
QScrollBar:vertical   { background:rgba(20,20,35,200); width:8px; border-radius:4px; }
QScrollBar::handle:vertical{ background:rgba(90,108,255,120); border-radius:4px; }
"""


# ─────────────────────────────────────────────
def main():
    app = QApplication(sys.argv)
    w = MainWindow()
    w.show()
    sys.exit(app.exec())

if __name__ == "__main__":
    main()

                   border-radius:10px; padding:10px 18px; font-size:14px; }
#effBtn:checked  { background:#5a6cff; color:#fff; border-color:#5a6cff; }
#effBtn:hover    { background:rgba(90,108,255,80); }
#combo           { background:rgba(35,35,58,220); color:#d8e0ff;
                   border:1px solid rgba(120,130,255,55); border-radius:6px; padding:3px 8px; }
#status          { color:#6a85c0; font-size:11px; }
#note            { color:#6a75aa; font-size:11px; background:rgba(30,30,55,180);
                   border-radius:8px; padding:8px; margin-top:6px; }
#bar             { background:rgba(35,35,58,220); border-radius:6px; }
#bar::chunk      { background:#5a6cff; border-radius:6px; }
QScrollBar:vertical   { background:rgba(20,20,35,200); width:8px; border-radius:4px; }
QScrollBar::handle:vertical{ background:rgba(90,108,255,120); border-radius:4px; }
"""


# ─────────────────────────────────────────────
def main():
    app = QApplication(sys.argv)
    w = MainWindow()
    w.show()
    sys.exit(app.exec())

if __name__ == "__main__":
    main()

        self.check_out = QPlainTextEdit(); self.check_out.setObjectName("panel")
        self.check_out.setReadOnly(True)
        self.check_out.setPlaceholderText("// Error Is Out < Loading Data >...")
        self.check_stat = QLabel("Ready"); self.check_stat.setObjectName("status")
        l.addWidget(btn); l.addWidget(self.check_out); l.addWidget(self.check_stat)
        return page

    def _pg_chat(self):
        page = QWidget()
        l = QVBoxLayout(page); l.setContentsMargins(20,20,20,20); l.setSpacing(8)
        l.addWidget(QLabel("🗨 Chat Bot — Chating With AI"))
        self.chat_hist = QPlainTextEdit(); self.chat_hist.setObjectName("panel")
        self.chat_hist.setReadOnly(True); self.chat_hist.setFont(QFont("Segoe UI",10))
        row = QHBoxLayout()
        self.chat_in = QLineEdit(); self.chat_in.setObjectName("lineEdit")
        self.chat_in.setPlaceholderText("Send A Question, Enter To Send..."); self.chat_in.returnPressed.connect(self._send_chat)
        sb = QPushButton("Send"); sb.setObjectName("primaryBtn"); sb.clicked.connect(self._send_chat)
        row.addWidget(self.chat_in); row.addWidget(sb)
        self.chat_stat = QLabel("Ready Now"); self.chat_stat.setObjectName("status")
        l.addWidget(self.chat_hist); l.addLayout(row); l.addWidget(self.chat_stat)
        return page

    def _pg_task(self):
        page = QWidget(); l = QVBoxLayout(page); l.setContentsMargins(20,20,20,20)
        l.addWidget(QLabel("📊 Manager")); l.addWidget(TaskWidget())
        return page

    def _pg_effects(self):
        page = QWidget(); l = QVBoxLayout(page); l.setContentsMargins(20,20,20,20); l.setSpacing(12)
        l.addWidget(QLabel("❄ The Effect In Window ( 15 FPS ☠️) ( Repair In The Future☠️☠️☠️☠️☠️☠️☠️☠️☠️ )"))
        row = QHBoxLayout(); row.setSpacing(10)
        self._eff_btns = {}
        for key,label in [("none","Off"),("snow","❄ Falling Snow"),("rain","🌧 Raining"),("leaves","🍃 Falling Leave")]:
            b = QPushButton(label); b.setObjectName("effBtn"); b.setCheckable(True)
            b.clicked.connect(lambda _,k=key: self._set_eff(k))
            row.addWidget(b); self._eff_btns[key]=b
        cur = self.cfg.get("effect","none")
        if cur in self._eff_btns: self._eff_btns[cur].setChecked(True)
        l.addLayout(row); l.addStretch()
        return page

    def _pg_settings(self):
        page = QWidget(); l = QVBoxLayout(page); l.setContentsMargins(20,20,20,20); l.setSpacing(8)
        l.addWidget(QLabel("🔑 Settings — API Keys"))

        l.addWidget(QLabel("Claude API Key:"))
        self.key_in = QLineEdit(self.cfg.get("claude_key",""))
        self.key_in.setObjectName("lineEdit"); self.key_in.setEchoMode(QLineEdit.EchoMode.Password)
        self.key_in.setPlaceholderText("sk-ant-...  (console.anthropic.com)")
        l.addWidget(self.key_in)

        l.addWidget(QLabel("DeepSeek API Key:"))
        self.deepseek_key_in = QLineEdit(self.cfg.get("deepseek_key",""))
        self.deepseek_key_in.setObjectName("lineEdit"); self.deepseek_key_in.setEchoMode(QLineEdit.EchoMode.Password)
        self.deepseek_key_in.setPlaceholderText("sk-...  (platform.deepseek.com)")
        l.addWidget(self.deepseek_key_in)

        l.addWidget(QLabel("OpenAI API Key (ChatGPT):"))
        self.openai_key_in = QLineEdit(self.cfg.get("openai_key",""))
        self.openai_key_in.setObjectName("lineEdit"); self.openai_key_in.setEchoMode(QLineEdit.EchoMode.Password)
        self.openai_key_in.setPlaceholderText("sk-...  (platform.openai.com)")
        l.addWidget(self.openai_key_in)

        sv = QPushButton("💾 Save All"); sv.setObjectName("primaryBtn"); sv.clicked.connect(self._save_settings)
        self.cfg_stat = QLabel(""); self.cfg_stat.setObjectName("status"); self.cfg_stat.setWordWrap(True)
        note = QLabel(
            "Copyright By Night Kanz Nary\n"
            "My Youtube Channel Night Kanz Nary\n"
            "My Profile : https://guns.lol/nightkanznary21\n"
            "Bypass And Active By Night Kanz Nary\n"
            "Thanks For Use My New Tool "
        )
        note.setObjectName("note"); note.setWordWrap(True)
        l.addWidget(sv); l.addWidget(self.cfg_stat); l.addWidget(note); l.addStretch()
        return page

    # ── Actions ─────────────────────────────────
    def _send_code(self):
        prompt = self.code_in.text().strip()
        if not prompt: return
        self.code_stat.setText("Sending to AI...")
        self.code_out.setPlainText("Loading Data In Server.........🔃")
        t,w = run_ai(prompt,self._lang(),self._model(),self.cfg,"code",
                     self._on_code_done, lambda e: (self.code_out.setPlainText(f"// Error: {e}"),
                                                     self.code_stat.setText("Error")))
        self._ai_refs = [t,w]

    def _on_code_done(self, text):
        # Strip code fences
        if "```" in text:
            parts = text.split("```")
            for p in parts:
                s = p.strip()
                if s and not s.split("\n")[0].lower() in ("python","cpp","c++","lua","luau",""):
                    text = s; break
                elif s:
                    lines = s.split("\n",1)
                    if len(lines)==2: text=lines[1]; break
        self.code_out.setPlainText(text.strip())
        self.code_stat.setText("Success ✓")

    def _do_check(self):
        code = self.code_out.toPlainText().strip()
        if not code: self.check_out.setPlainText("Error, Not Found Code....."); return
        self.check_stat.setText("Loading Data In Server /100%"); self.check_out.setPlainText("Loading")
        t,w = run_ai(code,self._lang(),self._model(),self.cfg,"check",
                     lambda r: (self.check_out.setPlainText(r), self.check_stat.setText("Success ✓")),
                     lambda e: (self.check_out.setPlainText(f"// Lỗi: {e}"), self.check_stat.setText("Error")))
        self._ai_refs = [t,w]

    def _send_chat(self):
        msg = self.chat_in.text().strip()
        if not msg: return
        self.chat_hist.appendPlainText(f"You: {msg}"); self.chat_in.clear()
        self.chat_stat.setText("Sending To AI...")
        t,w = run_ai(msg,self._lang(),self._model(),self.cfg,"chat",
                     lambda r: (self.chat_hist.appendPlainText(f"AI: {r}\n"), self.chat_stat.setText("Ready")),
                     lambda e: (self.chat_hist.appendPlainText(f"[Error]: {e}\n"), self.chat_stat.setText("Error")))
        self._ai_refs = [t,w]

    def _set_eff(self, key):
        for k,b in self._eff_btns.items(): b.setChecked(k==key)
        self.cfg["effect"] = key; save_cfg(self.cfg)
        self.canvas.set_effect(key)

    def _save_settings(self):
        self.cfg["claude_key"]   = self.key_in.text().strip()
        self.cfg["deepseek_key"] = self.deepseek_key_in.text().strip()
        self.cfg["openai_key"]   = self.openai_key_in.text().strip()
        ok = save_cfg(self.cfg)
        self.cfg_stat.setText(f"Save Success {CFG_FILE}" if ok else "Save Error!")

# ─────────────────────────────────────────────
#  STYLESHEET
# ─────────────────────────────────────────────
CSS = """
QWidget          { background:#0f0f18; color:#d8e0ff; font-size:13px; font-family:'Segoe UI',sans-serif; }
#sidebar         { background:#13131e; border-right:1px solid rgba(120,130,255,35); }
#brand           { color:#b9c4ff; font-size:16px; font-weight:700; padding:2px 4px; }
#navBtn          { background:transparent; color:#c4cbf0; border:none; border-radius:8px;
                   padding:9px 8px; text-align:left; font-size:13px; }
#navBtn:hover    { background:rgba(90,108,255,35); }
#navBtn:checked  { background:#5a6cff; color:#fff; font-weight:600; }
#panel           { background:rgba(10,10,20,230); color:#d8e0ff;
                   border:1px solid rgba(120,130,255,40); border-radius:10px; padding:8px; }
#lineEdit        { background:rgba(35,35,58,220); color:#fff;
                   border:1px solid rgba(120,130,255,55); border-radius:8px; padding:6px 10px; }
#primaryBtn      { background:#5a6cff; color:#fff; border:none; border-radius:8px; padding:6px 16px; font-weight:600; }
#primaryBtn:hover{ background:#7886ff; }
#secondaryBtn    { background:rgba(90,108,255,55); color:#d8e0ff; border:none; border-radius:8px; padding:6px 14px; font-weight:600; }
#secondaryBtn:hover{ background:rgba(90,108,255,100); }
#dangerBtn       { background:rgba(255,85,85,180); color:#fff; border:none; border-radius:8px; padding:6px 14px; font-weight:600; }
#dangerBtn:hover { background:rgba(255,110,110,220); }
#effBtn          { background:rgba(60,60,90,180); color:#d8e0ff; border:1px solid rgba(120,130,255,40);
                   border-radius:10px; padding:10px 18px; font-size:14px; }
#effBtn:checked  { background:#5a6cff; color:#fff; border-color:#5a6cff; }
#effBtn:hover    { background:rgba(90,108,255,80); }
#combo           { background:rgba(35,35,58,220); color:#d8e0ff;
                   border:1px solid rgba(120,130,255,55); border-radius:6px; padding:3px 8px; }
#status          { color:#6a85c0; font-size:11px; }
#note            { color:#6a75aa; font-size:11px; background:rgba(30,30,55,180);
                   border-radius:8px; padding:8px; margin-top:6px; }
#bar             { background:rgba(35,35,58,220); border-radius:6px; }
#bar::chunk      { background:#5a6cff; border-radius:6px; }
QScrollBar:vertical   { background:rgba(20,20,35,200); width:8px; border-radius:4px; }
QScrollBar::handle:vertical{ background:rgba(90,108,255,120); border-radius:4px; }
"""


# ─────────────────────────────────────────────
def main():
    app = QApplication(sys.argv)
    w = MainWindow()
    w.show()
    sys.exit(app.exec())

if __name__ == "__main__":
    main()
