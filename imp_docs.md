## 🎯&nbsp;Backend:

```python
from flask import Flask, jsonify, request, Response, render_template_string
import threading
import time
import uuid
import json
import datetime
from typing import Dict

app = Flask(__name__)

# ===== TASK MANAGEMENT =====
tasks: Dict[str, 'Task'] = {}
lock = threading.Lock()
app_logs = []
log_lock = threading.Lock()

class Task:
    def __init__(self, name):
        self.id = str(uuid.uuid4())
        self.name = name or f"Task {self.id[:6]}"
        self.progress = 0
        self.status = 'running'
        self._pause_event = threading.Event()
        self._cancel_event = threading.Event()
        self._pause_event.set()
        self.lock = threading.Lock()
        self.thread = threading.Thread(target=self.run)
        self.thread.start()
        self.log(f"🚀 Created: {self.name}")

    def run(self):
        try:
            while self.progress < 100:
                if self._cancel_event.is_set():
                    with self.lock:
                        self.status = 'failed'
                    break
                
                self._pause_event.wait()
                
                with self.lock:
                    self.progress = min(self.progress + 1, 100)
                    if self.progress % 10 == 0:
                        self.log(f"📊 {self.name}: {self.progress}%")
                    if self.progress >= 100:
                        self.status = 'completed'
                        break
                
                time.sleep(0.1)
        except Exception as e:
            with self.lock:
                self.status = 'failed'
            self.log(f"❌ {self.name} failed: {str(e)}", "error")
        finally:
            self.log(f"🔚 {self.name} ended: {self.status}")

    def pause(self):
        with self.lock:
            if self.status == 'running':
                self._pause_event.clear()
                self.status = 'paused'
                self.log(f"⏸ {self.name} paused")

    def resume(self):
        with self.lock:
            if self.status == 'paused':
                self._pause_event.set()
                self.status = 'running'
                self.log(f"▶️ {self.name} resumed")

    def cancel(self):
        self._cancel_event.set()
        self._pause_event.set()
        with self.lock:
            self.status = 'failed'
        self.log(f"⏹ {self.name} canceled")

    def reset(self):
        self.cancel()
        new_task = Task(self.name)
        self.log(f"🔄 {self.name} reset to new task")
        return new_task

    def log(self, message: str, log_type: str = "info"):
        timestamp = datetime.datetime.now().isoformat()
        entry = {
            "timestamp": timestamp,
            "message": message,
            "type": log_type
        }
        with log_lock:
            app_logs.append(entry)

# ===== ROUTES =====
@app.route('/')
def index():
    return render_template_string(open("templates/index.html").read())

@app.route('/start-task', methods=['POST'])
def start_task():
    data = request.get_json()
    task = Task(name=data.get('name'))
    with lock:
        tasks[task.id] = task
    return jsonify({'task_id': task.id, 'task_name': task.name})

@app.route('/pause-task/<task_id>', methods=['POST'])
def pause_task(task_id):
    with lock:
        task = tasks.get(task_id)
    if task and task.status == 'running':
        task.pause()
        return jsonify({'status': 'paused'})
    return jsonify({'error': 'Task not running or not found'}), 400

@app.route('/resume-task/<task_id>', methods=['POST'])
def resume_task(task_id):
    with lock:
        task = tasks.get(task_id)
    if task and task.status == 'paused':
        task.resume()
        return jsonify({'status': 'running'})
    return jsonify({'error': 'Task not paused or not found'}), 400

@app.route('/reset-task/<task_id>', methods=['POST'])
def reset_task(task_id):
    with lock:
        task = tasks.get(task_id)
        if not task:
            return jsonify({'error': 'Task not found'}), 404
        
        new_task = task.reset()
        del tasks[task_id]
        tasks[new_task.id] = new_task
        return jsonify({'new_task_id': new_task.id})

@app.route('/remove-task/<task_id>', methods=['POST'])
def remove_task(task_id):
    with lock:
        if task_id in tasks:
            del tasks[task_id]
            return jsonify({'status': 'removed'})
        return jsonify({'error': 'Task not found'}), 404

@app.route('/task-progress/<task_id>')
def task_progress(task_id):
    def generate():
        while True:
            with lock:
                task = tasks.get(task_id)
                if not task:
                    yield f"data: {json.dumps({'error': 'Task not found'})}\n\n"
                    break
                
                with task.lock:
                    data = {
                        'status': task.status,
                        'progress': task.progress,
                        'name': task.name,
                        'id': task.id
                    }
                
                yield f"data: {json.dumps(data)}\n\n"
                
                if data['status'] in ['completed', 'failed']:
                    break
            time.sleep(0.5)
    return Response(generate(), mimetype='text/event-stream')

@app.route('/app-logs')
def app_logs_stream():
    def generate():
        last_sent = 0
        while True:
            with log_lock:
                if len(app_logs) > last_sent:
                    for log in app_logs[last_sent:]:
                        yield f"data: {json.dumps(log)}\n\n"
                    last_sent = len(app_logs)
            time.sleep(0.5)
    return Response(generate(), mimetype='text/event-stream')

if __name__ == '__main__':
    app.run(debug=True)
```

## 🎯&nbsp;Frontend:

```html
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>Task Manager</title>
    <script src="https://cdnjs.cloudflare.com/ajax/libs/gsap/3.12.2/gsap.min.js"></script>
    <style>
      :root {
        --running: #3b82f6;
        --paused: #eab308;
        --completed: #22c55e;
        --failed: #ef4444;
      }

      body {
        font-family: -apple-system, sans-serif;
        padding: 2rem;
        max-width: 800px;
        margin: 0 auto;
        background: #f8fafc;
      }

      .task-card {
        background: white;
        padding: 1.5rem;
        border-radius: 1rem;
        box-shadow: 0 4px 6px -1px rgba(0, 0, 0, 0.1);
        margin-bottom: 1rem;
        opacity: 0;
        transform: translateY(20px);
      }

      .task-header {
        display: flex;
        justify-content: space-between;
        align-items: center;
        margin-bottom: 1rem;
      }

      .task-title {
        font-weight: 600;
        color: #1e293b;
      }

      .task-id {
        color: #64748b;
        font-family: monospace;
        font-size: 0.875rem;
      }

      .status {
        padding: 0.375rem 0.75rem;
        border-radius: 2rem;
        font-size: 0.875rem;
        display: inline-flex;
        align-items: center;
        gap: 0.5rem;
      }

      .status::before {
        content: "";
        width: 0.75rem;
        height: 0.75rem;
        border-radius: 50%;
        display: block;
      }

      .status.running {
        background: #dbeafe;
        color: var(--running);
      }
      .status.running::before {
        background: var(--running);
      }

      .status.paused {
        background: #fef3c7;
        color: var(--paused);
      }
      .status.paused::before {
        background: var(--paused);
      }

      .status.completed {
        background: #dcfce7;
        color: var(--completed);
      }
      .status.completed::before {
        background: var(--completed);
      }

      .status.failed {
        background: #fee2e2;
        color: var(--failed);
      }
      .status.failed::before {
        background: var(--failed);
      }

      .progress-container {
        position: relative;
        height: 8px;
        background: #e2e8f0;
        border-radius: 4px;
        overflow: hidden;
      }

      .progress-fill {
        height: 100%;
        width: 0%;
        background: var(--running);
        transition: width 0.3s ease;
      }

      .progress-text {
        position: absolute;
        right: 0.5rem;
        top: 50%;
        transform: translateY(-50%);
        color: white;
        font-weight: 500;
        text-shadow: 0 1px 2px rgba(0, 0, 0, 0.2);
      }

      .controls {
        margin-top: 1rem;
        display: flex;
        gap: 0.5rem;
        flex-wrap: wrap;
      }

      button {
        padding: 0.5rem 1rem;
        border: none;
        border-radius: 0.5rem;
        cursor: pointer;
        display: inline-flex;
        align-items: center;
        gap: 0.5rem;
        transition: all 0.2s ease;
      }

      .console-panel {
        background: #1e293b;
        color: white;
        padding: 1.5rem;
        border-radius: 1rem;
        margin-top: 2rem;
      }

      #consoleLogs {
        height: 200px;
        overflow-y: auto;
        font-family: monospace;
        padding: 1rem;
        background: #0f172a;
        border-radius: 0.5rem;
        margin: 1rem 0;
      }
    </style>
  </head>
  <body>
    <div>
      <input type="text" id="taskName" placeholder="Task name (optional)" />
      <button id="createTask">✨ Create Task</button>
    </div>

    <div id="tasksContainer"></div>

    <div class="console-panel">
      <h3>System Logs</h3>
      <div id="consoleLogs"></div>
      <button onclick="document.getElementById('consoleLogs').innerHTML = ''">
        🧹 Clear Logs
      </button>
    </div>

    <script>
      const tasksContainer = document.getElementById("tasksContainer");
      const consoleLogs = document.getElementById("consoleLogs");

      document
        .getElementById("createTask")
        .addEventListener("click", async () => {
          const taskName = document.getElementById("taskName").value.trim();

          try {
            const response = await fetch("/start-task", {
              method: "POST",
              headers: { "Content-Type": "application/json" },
              body: JSON.stringify({ name: taskName }),
            });

            const data = await response.json();
            createTaskUI(data.task_id, data.task_name);
            document.getElementById("taskName").value = "";
          } catch (error) {
            addLog(`❌ Failed to create task: ${error.message}`);
          }
        });

      function createTaskUI(taskId, taskName) {
        const taskHTML = `
                <div class="task-card" id="task-${taskId}">
                    <div class="task-header">
                        <div>
                            <div class="task-title">${taskName}</div>
                            <div class="task-id">#${taskId.slice(0, 6)}</div>
                        </div>
                        <div class="status running">Running</div>
                    </div>
                    <div class="progress-container">
                        <div class="progress-fill"></div>
                        <div class="progress-text">0%</div>
                    </div>
                    <div class="controls">
                        <button class="pause">⏸ Pause</button>
                        <button class="resume" style="display:none">▶ Resume</button>
                        <button class="reset" style="display:none">🔄 Reset</button>
                        <button class="cancel">⏹ Cancel</button>
                        <button class="remove" style="display:none">🗑 Remove</button>
                    </div>
                </div>
            `;

        tasksContainer.insertAdjacentHTML("afterbegin", taskHTML);
        const taskElement = document.getElementById(`task-${taskId}`);

        gsap.to(taskElement, {
          opacity: 1,
          y: 0,
          duration: 0.3,
          ease: "power2.out",
        });

        setupTaskControls(taskId, taskName);
      }

      function setupTaskControls(taskId, taskName) {
        const taskElement = document.getElementById(`task-${taskId}`);
        const status = taskElement.querySelector(".status");
        const progressFill = taskElement.querySelector(".progress-fill");
        const progressText = taskElement.querySelector(".progress-text");
        const controls = {
          pause: taskElement.querySelector(".pause"),
          resume: taskElement.querySelector(".resume"),
          reset: taskElement.querySelector(".reset"),
          cancel: taskElement.querySelector(".cancel"),
          remove: taskElement.querySelector(".remove"),
        };

        const eventSource = new EventSource(`/task-progress/${taskId}`);

        eventSource.onmessage = (e) => {
          const data = JSON.parse(e.data);

          // Update progress
          progressFill.style.width = `${data.progress}%`;
          progressText.textContent = `${data.progress}%`;

          // Update status
          status.className = `status ${data.status}`;
          status.textContent =
            data.status.charAt(0).toUpperCase() + data.status.slice(1);

          // Update controls
          controls.pause.style.display =
            data.status === "running" ? "flex" : "none";
          controls.resume.style.display =
            data.status === "paused" ? "flex" : "none";
          controls.reset.style.display = [
            "paused",
            "completed",
            "failed",
          ].includes(data.status)
            ? "flex"
            : "none";
          controls.remove.style.display = ["completed", "failed"].includes(
            data.status
          )
            ? "flex"
            : "none";
        };

        // Control handlers
        controls.pause.addEventListener("click", () =>
          fetch(`/pause-task/${taskId}`, { method: "POST" })
        );
        controls.resume.addEventListener("click", () =>
          fetch(`/resume-task/${taskId}`, { method: "POST" })
        );
        controls.reset.addEventListener("click", () => {
          fetch(`/reset-task/${taskId}`, { method: "POST" })
            .then((response) => response.json())
            .then((data) => {
              taskElement.remove();
              createTaskUI(data.new_task_id, taskName);
            });
        });
        controls.cancel.addEventListener("click", () => {
          fetch(`/cancel-task/${taskId}`, { method: "POST" });
          taskElement.remove();
        });
        controls.remove.addEventListener("click", () => {
          fetch(`/remove-task/${taskId}`, { method: "POST" });
          taskElement.remove();
        });
      }

      // Console logging
      const consoleEvents = new EventSource("/app-logs");
      consoleEvents.onmessage = (e) => {
        const log = JSON.parse(e.data);
        const entry = document.createElement("div");
        entry.innerHTML = `
                <span style="color: #94a3b8">[${new Date(
                  log.timestamp
                ).toLocaleTimeString()}]</span>
                ${log.message}
            `;
        consoleLogs.appendChild(entry);
        consoleLogs.scrollTop = consoleLogs.scrollHeight;
      };
    </script>
  </body>
</html>
```
