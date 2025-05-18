## 🎯&nbsp;Backend:

```python
from flask import Flask, jsonify, request, Response, render_template_string
import threading
import time
import uuid
import json
import datetime
from typing import Dict, Optional, Any, Union, Tuple

app = Flask(__name__)

# ========== CORE LOGIC ==========
tasks: Dict[str, 'Task'] = {}
lock = threading.Lock()
app_logs = []
log_lock = threading.Lock()

class Task:
    def __init__(self) -> None:
        self.id: str = str(uuid.uuid4())
        self.progress: int = 0
        self.status: str = 'running'
        self._pause_event = threading.Event()
        self._cancel_event = threading.Event()
        self._pause_event.set()
        self.lock = threading.Lock()
        self.thread = threading.Thread(target=self.run)
        self.thread.start()

    def run(self) -> None:
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
                        log_task_event(self.id, 'info', f"Progress: {self.progress}%")
                    if self.progress >= 100:
                        self.status = 'completed'
                        log_task_event(self.id, 'success', "Completed successfully")
                        break
                
                time.sleep(0.1)
        except Exception as e:
            with self.lock:
                self.status = 'failed'
            log_task_event(self.id, 'error', f"Failed: {str(e)}")

    def pause(self):
        with self.lock:
            if self.status == 'running':
                self._pause_event.clear()
                self.status = 'paused'  # This must be set
                log_task_event(self.id, 'pause', "Paused by user")

    def resume(self) -> None:
        with self.lock:
            if self.status == 'paused':
                self._pause_event.set()
                self.status = 'running'

    def cancel(self) -> None:
        self._cancel_event.set()
        self._pause_event.set()
        with self.lock:
            self.status = 'failed'

    def reset(self) -> 'Task':
        self.cancel()
        return Task()

# ========== ROUTES ==========
@app.route('/')
def index() -> str:
    """Serve the main interface"""
    return render_template_string(open("templates/index.html").read())

@app.route('/start-task', methods=['POST'])
def start_task() -> Union[Response, Tuple[Response, int]]:
    """Endpoint to start new tasks"""
    json_data: Optional[dict] = request.get_json(silent=True) or {}
    task_name = json_data.get('name', 'Unnamed') if isinstance(json_data, dict) else 'Unnamed'
    
    task = Task()
    with lock:
        tasks[task.id] = task
    
    log_task_event(task.id, 'launch', f"Launched - '{task_name}'")
    return jsonify({'task_id': task.id})

@app.route('/pause-task/<task_id>', methods=['POST'])
def pause_task(task_id: str) -> Union[Response, Tuple[Response, int]]:
    with lock:
        task = tasks.get(task_id)
    
    if task:
        with task.lock:
            if task.status == 'running':
                task.pause()
                return jsonify({'status': 'paused'})
            return jsonify({'error': 'Task not running'}), 400
    return jsonify({'error': 'Task not found'}), 404

@app.route('/resume-task/<task_id>', methods=['POST'])
def resume_task(task_id: str) -> Union[Response, Tuple[Response, int]]:
    with lock:
        task = tasks.get(task_id)
    
    if task:
        with task.lock:
            if task.status == 'paused':
                task.resume()
                return jsonify({'status': 'running'})
            return jsonify({'error': 'Task not paused'}), 400
    return jsonify({'error': 'Task not found'}), 404

@app.route('/cancel-task/<task_id>', methods=['POST'])
def cancel_task(task_id: str) -> Union[Response, Tuple[Response, int]]:
    with lock:
        task = tasks.get(task_id)
    
    if task:
        task.cancel()
        return jsonify({'status': 'canceled'})
    return jsonify({'error': 'Task not found'}), 404

@app.route('/reset-task/<task_id>', methods=['POST'])
def reset_task(task_id: str) -> Union[Response, Tuple[Response, int]]:
    with lock:
        task = tasks.get(task_id)
    
    if not task:
        return jsonify({'error': 'Task not found'}), 404
    
    try:
        new_task = task.reset()
        with lock:
            del tasks[task_id]
            tasks[new_task.id] = new_task
        return jsonify({'new_task_id': new_task.id})
    except Exception as e:
        return jsonify({'error': f'Reset failed: {str(e)}'}), 500

@app.route('/remove-task/<task_id>', methods=['POST'])
def remove_task(task_id: str) -> Union[Response, Tuple[Response, int]]:
    with lock:
        if task_id not in tasks:
            return jsonify({'error': 'Task not found'}), 404
        
        try:
            task = tasks[task_id]
            task.cancel()
            del tasks[task_id]
            return jsonify({'status': 'removed'})
        except Exception as e:
            return jsonify({'error': f'Removal failed: {str(e)}'}), 500

@app.route('/task-progress/<task_id>')
def task_progress(task_id):
    def generate():
        while True:
            with lock:
                task = tasks.get(task_id)
                if not task:
                    yield json.dumps({'error': 'Task not found'})
                    break
                
                with task.lock:
                    data = {
                        'status': task.status,
                        'progress': task.progress
                    }
                
                yield f"data: {json.dumps(data)}\n\n"
                
                if data['status'] in ['completed', 'failed']:
                    break
            time.sleep(0.5)
    return Response(generate(), mimetype='text/event-stream')

@app.route('/app-logs')
def app_logs_stream() -> Response:
    def generate() -> Any:
        with log_lock:
            for log in app_logs[-100:]:
                yield f"data: {json.dumps(log)}\n\n"
        
        while True:
            with log_lock:
                if len(app_logs) > 0:
                    log = app_logs[-1]
                    yield f"data: {json.dumps(log)}\n\n"
            time.sleep(0.5)
    
    return Response(generate(), mimetype='text/event-stream')

# ========== LOGGING ==========
def log_task_event(task_id: str, event_type: str, message: str) -> None:
    emoji_map = {
        'launch': '🚀',
        'success': '✅',
        'warning': '⚠️',
        'error': '❌',
        'info': 'ℹ️',
        'pause': '⏸️',
        'resume': '▶️',
        'cancel': '⏹️'
    }
    emoji = emoji_map.get(event_type, '🔵')
    timestamp = datetime.datetime.now().strftime("%H:%M:%S")
    log_entry = {
        'timestamp': datetime.datetime.now().isoformat(),
        'message': f"{emoji} [{timestamp}] Task {task_id[:6]} {message}"
    }
    with log_lock:
        app_logs.append(log_entry)

if __name__ == '__main__':
    app.run(debug=True)

if __name__ == '__main__':
    app.run(debug=True)
```

## 🎯&nbsp;Frontend:

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Task Monitor</title>
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
        }

        .task-container {
            margin: 2rem 0;
        }

        .task-card {
            background: white;
            padding: 1.5rem;
            border-radius: 1rem;
            box-shadow: 0 4px 6px -1px rgba(0,0,0,0.1);
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

        .status {
            padding: 0.25rem 0.75rem;
            border-radius: 2rem;
            font-size: 0.875rem;
            font-weight: 500;
        }

        .status.running { background: color-mix(in srgb, var(--running) 15%, transparent); color: var(--running); }
        .status.paused { background: color-mix(in srgb, var(--paused) 15%, transparent); color: var(--paused); }
        .status.completed { background: color-mix(in srgb, var(--completed) 15%, transparent); color: var(--completed); }
        .status.failed { background: color-mix(in srgb, var(--failed) 15%, transparent); color: var(--failed); }

        .progress-bar {
            height: 8px;
            background: #f3f4f6;
            border-radius: 4px;
            overflow: hidden;
        }

        .progress-fill {
            height: 100%;
            width: 0%;
            background: var(--running);
            transition: width 0.3s ease;
        }

        .controls {
            margin-top: 1rem;
            display: flex;
            gap: 0.5rem;
        }

        button {
            padding: 0.5rem 1rem;
            border: none;
            border-radius: 0.5rem;
            cursor: pointer;
            background: #f3f4f6;
            transition: all 0.2s ease;
        }

        button:hover {
            filter: brightness(0.95);
        }

        #newTaskBtn {
            background: var(--running);
            color: white;
        }
    </style>
</head>
<body>
    <div>
        <button id="newTaskBtn">✨ Start New Task</button>
    </div>

    <div class="task-container" id="tasksContainer"></div>

    <script>
        const tasksContainer = document.getElementById('tasksContainer');
        const newTaskBtn = document.getElementById('newTaskBtn');

        newTaskBtn.addEventListener('click', async () => {
            try {
                const response = await fetch('/start-task', { method: 'POST' });
                const { task_id } = await response.json();
                createTaskUI(task_id);
            } catch (error) {
                console.error('Task creation failed:', error);
            }
        });

        function createTaskUI(taskId) {
            const taskHTML = `
                <div class="task-card" id="task-${taskId}">
                    <div class="task-header">
                        <div>Task #${taskId.slice(0, 6)}</div>
                        <div class="status running">Running</div>
                    </div>
                    <div class="progress-bar">
                        <div class="progress-fill"></div>
                    </div>
                    <div class="controls">
                        <button class="pause">⏸ Pause</button>
                        <button class="resume" style="display:none">▶ Resume</button>
                        <button class="cancel">⏹ Cancel</button>
                    </div>
                </div>
            `;

            // Insert and animate new task
            tasksContainer.insertAdjacentHTML('afterbegin', taskHTML);
            const taskCard = document.getElementById(`task-${taskId}`);
            gsap.to(taskCard, {
                opacity: 1,
                y: 0,
                duration: 0.3,
                ease: "power2.out"
            });

            setupTaskControls(taskId);
        }

        function setupTaskControls(taskId) {
            const taskCard = document.getElementById(`task-${taskId}`);
            const status = taskCard.querySelector('.status');
            const progressFill = taskCard.querySelector('.progress-fill');
            const pauseBtn = taskCard.querySelector('.pause');
            const resumeBtn = taskCard.querySelector('.resume');
            const cancelBtn = taskCard.querySelector('.cancel');

            const eventSource = new EventSource(`/task-progress/${taskId}`);

            eventSource.onmessage = (e) => {
                const data = JSON.parse(e.data);
                if (data.error) return;

                // Update progress
                progressFill.style.width = `${data.progress}%`;

                // Update status
                status.className = `status ${data.status}`;
                status.textContent = data.status.charAt(0).toUpperCase() + data.status.slice(1);

                // Update controls
                pauseBtn.style.display = data.status === 'running' ? 'block' : 'none';
                resumeBtn.style.display = data.status === 'paused' ? 'block' : 'none';
            };

            // Control handlers
            pauseBtn.addEventListener('click', () => fetch(`/pause-task/${taskId}`, { method: 'POST' }));
            resumeBtn.addEventListener('click', () => fetch(`/resume-task/${taskId}`, { method: 'POST' }));
            cancelBtn.addEventListener('click', () => {
                eventSource.close();
                taskCard.remove();
                fetch(`/cancel-task/${taskId}`, { method: 'POST' });
            });
        }
    </script>
</body>
</html>
```
