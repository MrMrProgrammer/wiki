# 📡 WebSocket + Callback API Integration

This project demonstrates a simple integration between a **web page** and a **backend processing service** using **FastAPI**, **WebSocket**, and a **callback URL**.

## 📁 System Overview

- The **web page** collects user input.
- A WebSocket connection is established with the FastAPI server.
- A processing request is sent to the API.
- The API forwards the request to an external service (e.g., a microservice) along with a callback URL.
- When the result is ready, the external service sends it back to the callback URL.
- The FastAPI server then pushes the result to the web client via WebSocket.

---

## 📄 Core Files

### 1. `main.py` (FastAPI)

```python
connections = []

@app.websocket("/ws")
async def websocket_endpoint(websocket: WebSocket):
    await websocket.accept()
    connections.append(websocket)
    try:
        while True:
            await websocket.receive_text()  # keep the connection alive
    except WebSocketDisconnect:
        connections.remove(websocket)

@app.post("/callback")
async def callback(data: dict):
    for connection in connections:
        await connection.send_json(data)
    return {"status": "received"}

@app.post("/start")
async def start(request: dict):
    first_api_url = "http://localhost:9001/do-something"

    async with httpx.AsyncClient(timeout=20) as client:
        await client.post(first_api_url, params={
            "callback_url": "http://localhost:9002/callback",
            "user_input": request["user_input"]
        })

    return {"status": "started"}
```


### 2. `index.html` (Web Page)

- Accepts user input.

- Establishes WebSocket connection.

- Sends a request to /start.

- Displays the final result using SweetAlert.

```javascript
<script src="https://cdn.jsdelivr.net/npm/sweetalert2@11"></script>
<script>
    let socket;
    let socketReady = false;

    async function startProcess() {
        const input = document.getElementById('inputField').value.trim();
        if (!input) {
            Swal.fire({
                title: 'Error!',
                text: 'Please enter some text.',
                icon: 'error'
            });
            return;
        }

        // Connect to WebSocket
        await connectWebSocket();

        if (!socketReady) {
            Swal.fire({
                title: 'Error!',
                text: 'Failed to connect to server.',
                icon: 'error'
            });
            return;
        }

        // Show loading message
        Swal.fire({
            title: 'Request sent!',
            text: 'Waiting for the result...',
            icon: 'info',
            timer: 3000,
            showConfirmButton: false
        });

        // Send processing request to the backend
        await fetch('http://localhost:9002/start', {
            method: 'POST',
            headers: {'Content-Type': 'application/json'},
            body: JSON.stringify({ user_input: input })
        });
    }

    function connectWebSocket() {
        return new Promise((resolve, reject) => {
            if (socket && socket.readyState === WebSocket.OPEN) {
                socketReady = true;
                resolve();
                return;
            }

            socket = new WebSocket('ws://localhost:9002/ws');

            socket.onopen = function() {
                console.log('✅ WebSocket connected');
                socketReady = true;
                resolve();
            };

            socket.onmessage = function(event) {
                const data = JSON.parse(event.data);
                Swal.fire({
                    title: 'Result Ready!',
                    text: data.message,
                    icon: 'success'
                });
            };

            socket.onclose = function() {
                console.log('🔄 WebSocket closed');
                socketReady = false;
            };

            socket.onerror = function(event) {
                console.error('❌ WebSocket error:', event);
                socketReady = false;
                reject();
            };
        });
    }
</script>
```

## ⚙️ Dependencies

- Python >= 3.8  
- [FastAPI](https://fastapi.tiangolo.com/)  
- [httpx](https://www.python-httpx.org/)  
- [uvicorn](https://www.uvicorn.org/)  

### 📦 Install

```bash
pip install fastapi httpx uvicorn
```

## 🚀 Run the server

```bash
uvicorn main:app --reload --port 9002
```

### 🔄 How to Test

1. Start the FastAPI server on port 9002:

```bash
uvicorn main:app --reload --port 9002
```

2. Make sure the external processing service is running on port `9001`, and accepts a `callback_url`.

3. Open the `index.html` page in your browser.

4. Enter some text in the input field and click "Start".

5. Wait for the result to be received via WebSocket and shown in a SweetAlert popup.
