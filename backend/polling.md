# Polling a Backend Task Status Using Web Workers

This guide demonstrates how to implement a clean and efficient polling mechanism using Web Workers to track the status of a background task on a backend API.

## 🧠 Why Use a Web Worker?

By offloading the polling logic to a Web Worker, you avoid blocking the main browser thread, resulting in smoother UI performance and better scalability.

---

## 🏗️ Architecture Overview

- The frontend sends a request to the backend to start a task.
- The backend responds with a `task_id`.
- A Web Worker is initialized and starts polling the backend for task status using that `task_id`.
- The worker sends updates back to the main thread.
- Once the task is marked as `done` or `failed`, polling stops.

---

## 🚀 Setup

### `index.html`

```html
<button onclick="startTask()">Start Task</button>
<p id="status">Waiting...</p>

<script>
let worker = null;

function startTask() {
    fetch('http://localhost:9002/start-task', { method: 'POST' })
        .then(response => response.json())
        .then(data => {
            if (worker) {
                worker.terminate();
            }

            worker = new Worker('worker.js');
            worker.postMessage({ taskId: data.task_id });

            worker.onmessage = (event) => {
                const { status } = event.data;
                document.getElementById('status').innerText = `Status: ${status}`;

                if (status === 'done' || status === 'failed') {
                    worker.terminate();
                    worker = null;
                }
            };

            worker.onerror = (error) => {
                console.error("Worker Error:", error.message);
                document.getElementById('status').innerText = "Error in background task.";
                worker.terminate();
                worker = null;
            };
        })
        .catch(err => {
            console.error("Failed to start task:", err);
            document.getElementById('status').innerText = "Failed to start task.";
        });
}
</script>
```

### `worker.js`

``` javascript
let controller = null;

self.onmessage = function (event) {
    const { taskId } = event.data;
    poll(taskId);
};

function poll(taskId) {
    if (controller) {
        controller.abort();
    }

    controller = new AbortController();

    fetch(`http://localhost:9002/task-status/${taskId}`, {
        signal: controller.signal
    })
        .then(response => response.json())
        .then(data => {
            self.postMessage({ status: data.status });

            if (data.status !== 'done' && data.status !== 'failed') {
                setTimeout(() => poll(taskId), 2000);
            }
        })
        .catch(error => {
            if (error.name === 'AbortError') {
                console.log('Previous request aborted.');
            } else {
                console.error('Worker polling error:', error);
                setTimeout(() => poll(taskId), 5000);
            }
        });
}
```

---

## 🔄 Backend API Specification (example)

### Endpoints

| Endpoint                     | Method | Description                                                       |
|-----------------------------|--------|-------------------------------------------------------------------|
| `/start-task`               | POST   | Starts a new background task and returns a `task_id`.             |
| `/task-status/<task_id>`    | GET    | Returns the status of a task (`pending`, `in_progress`, `done`, `failed`). |

### 📥 Response for `/start-task`
```json
{
  "task_id": "abc123"
}
```

### 📥 Response for `/task-status/abc123`
```json
{
  "status": "in_progress"
}
```

---

## 🏃 Running the Project

> **Important:** Web Workers cannot be loaded from `file://` URLs due to browser security restrictions.  
> You **must** run your project through a local web server for the worker to load correctly.

### ✅ Recommended: Use Python's built-in HTTP server

If you have Python 3 installed, run the following in your project directory:

```bash
python3 -m http.server
```

This will start a local server at `http://localhost:8000`
Now open your browser and navigate to:

```bash
http://localhost:8000/index.html
```

This ensures your worker.js and other static assets are loaded correctly by the browser.

---

### ✅ Advantages
- Main thread stays responsive.

- Easy to scale without affecting UI performance.

- Clean separation of concerns between UI logic and polling mechanism.

---

### 📌 Notes
- 

- Ensure CORS is properly configured on the backend.

- You can extend this to show progress bars, cancel polling, or even retry on network failures.

- For real-time updates, consider using WebSockets or Server-Sent Events (SSE) if supported by your backend.



