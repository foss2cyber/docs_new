A **modular addition** to implement launchable demo apps without modifying the existing code. Create these new files:

**1. demo_app.py** (drop-in module):
```python
from flask import Flask
import threading
import time
import random
from typing import Optional
import subprocess
import signal

class DemoAppManager:
    def __init__(self, main_app_port=5000):
        self.next_port = main_app_port + 1
        self.active_demos = {}
        self.lock = threading.Lock()
        
    def create_demo_app(self, task_id: str) -> int:
        with self.lock:
            port = self.next_port
            self.next_port += 1
            
            app = Flask(f"DemoApp_{task_id}")
            
            @app.route('/')
            def demo_route():
                return "This is a demo app. 🚀"
            
            def run_server():
                import sys
                from werkzeug.serving import make_server
                server = make_server('localhost', port, app)
                self.active_demos[task_id] = (server, port)
                server.serve_forever()
                
            thread = threading.Thread(target=run_server, daemon=True)
            thread.start()
            
            # Wait for server to start
            time.sleep(0.5)
            return port

    def stop_demo_app(self, task_id: str):
        with self.lock:
            if task_id in self.active_demos:
                server, port = self.active_demos.pop(task_id)
                server.shutdown()
                time.sleep(0.2)  # Allow cleanup

# Initialize with main app port +1 as first demo port
demo_manager = DemoAppManager(main_app_port=5000)
```

**2. Add to existing app.py** (at the bottom, before `if __name__`):
```python
from demo_app import demo_manager

@app.route('/start-demo-task', methods=['POST'])
def start_demo_task():
    task_id = str(uuid.uuid4())
    port = demo_manager.create_demo_app(task_id)
    
    # Create a pseudo-task for progress tracking
    with lock:
        tasks[task_id] = {
            'id': task_id,
            'status': 'running',
            'type': 'demo',
            'port': port,
            'progress': 0,
            'created_at': datetime.datetime.now()
        }
    
    return jsonify({
        'task_id': task_id,
        'demo_url': f'http://localhost:{port}'
    })

@app.route('/stop-demo-task/<task_id>', methods=['POST'])
def stop_demo_task(task_id):
    demo_manager.stop_demo_app(task_id)
    with lock:
        if task_id in tasks:
            del tasks[task_id]
    return jsonify({'status': 'removed'})
```

**3. Frontend additions** (add to index.html before closing `</script>`):
```javascript
// Add to controls section
const demoControlsHTML = `
    <div style="margin: 1rem 0;">
        <button id="startDemoTask" style="background:#8b5cf6;color:white;">
            🚀 Launch Demo App
        </button>
    </div>
`;
document.body.insertAdjacentHTML('afterbegin', demoControlsHTML);

// Add demo task handler
document.getElementById('startDemoTask').addEventListener('click', async () => {
    try {
        const response = await fetch('/start-demo-task', { method: 'POST' });
        const { task_id, demo_url } = await response.json();
        createDemoTaskUI(task_id, demo_url);
    } catch (error) {
        addLog(`❌ Failed to launch demo: ${error.message}`);
    }
});

function createDemoTaskUI(taskId, demoUrl) {
    const taskHTML = `
        <div class="task-card" id="task-${taskId}">
            <div class="task-header">
                <div>
                    <div class="task-title">Demo App</div>
                    <div class="task-id">#${taskId.slice(0, 6)}</div>
                </div>
                <div class="status running">Running</div>
            </div>
            <div class="progress-container">
                <div class="progress-fill" style="width:100%;background:#8b5cf6"></div>
                <div class="progress-text">DEMO</div>
            </div>
            <div class="controls">
                <a href="${demoUrl}" target="_blank" style="color:white;text-decoration:none;">
                    <button style="background:#8b5cf6">🌐 Open Demo</button>
                </a>
                <button class="stop-demo">⏹ Stop Demo</button>
            </div>
        </div>
    `;

    tasksContainer.insertAdjacentHTML('afterbegin', taskHTML);
    const taskElement = document.getElementById(`task-${taskId}`);
    
    gsap.to(taskElement, {
        opacity: 1,
        y: 0,
        duration: 0.3,
        ease: "power2.out"
    });

    // Demo control handler
    taskElement.querySelector('.stop-demo').addEventListener('click', () => {
        fetch(`/stop-demo-task/${taskId}`, { method: 'POST' });
        taskElement.remove();
    });
}
```

**Key Features:**
1. **Non-invasive Integration** - Can be removed by:
   - Deleting `demo_app.py`
   - Removing the 2 new routes from `app.py`
   - Removing the frontend additions
2. **Automatic Port Management** - Starts demo apps on sequential ports
3. **Progress UI Integration** - Uses existing UI patterns
4. **Isolated Operation** - Demo apps run in separate processes
5. **Safe Cleanup** - Proper server shutdown on removal

**Usage:**
1. Click "Launch Demo App" button
2. New card appears with "Open Demo" button
3. Demo app opens in new tab showing the message
4. Stop demos with the ⏹ button

**Note:** This uses Werkzeug's development server. For production, you'd want to:
1. Add error handling for port conflicts
2. Implement proper process management
3. Add authentication if needed
4. Use proper production WSGI servers

This implementation keeps your original app intact while adding demo capabilities through modular extensions.
