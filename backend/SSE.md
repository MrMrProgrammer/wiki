# 📡 Server-Sent Events (SSE) with FastAPI

This guide explains how to implement Server-Sent Events (SSE) using **FastAPI** on the backend and a simple **HTML/JavaScript** client to receive real-time messages from the server.

---

## 🧠 What is SSE?

**Server-Sent Events (SSE)** is a standard allowing a server to **push real-time updates** to clients over a single HTTP connection. Unlike WebSockets, SSE is **unidirectional** (server → client), simple to implement, and supported natively in modern browsers.

---

## 📁 Project Structure Overview

```
project/
├── app/
│   ├── routes.py          # Contains FastAPI routes for SSE
│   ├── schemas.py         # Pydantic schema for messages
│   └── templates/
│       └── sse_index.html # Frontend page for viewing messages
```

---

## 🛠 Backend (FastAPI)

### ✅ 1. Subscribing to Events (`/events`)

```python
@router.get("/events")
async def sse():
    queue = asyncio.Queue()
    subscribers.add(queue)

    async def event_generator():
        try:
            while True:
                message = await queue.get()
                yield f"data: {message}\n\n"
        except asyncio.CancelledError:
            pass
        finally:
            subscribers.remove(queue)

    return StreamingResponse(event_generator(), media_type="text/event-stream")
```

- Each connected client gets its own `asyncio.Queue`.
- When a message arrives, it's streamed to the client using the `text/event-stream` MIME type.
- Disconnected clients are automatically unsubscribed.

---

### ✅ 2. Sending a Message (`/send`)

```python
@router.post("/send")
async def send_message(payload: MessageSchema):
    for queue in subscribers:
        await queue.put(payload.message)
    return {"status": "message sent"}
```

- Accepts a JSON payload.
- Pushes the message to **all currently connected subscribers**.

---

### ✅ 3. Serving the HTML Frontend (`/`)

```python
@router.get("/", response_class=HTMLResponse)
async def get_index():
    file_path = Path(__file__).parent / "templates/sse_index.html"
    if not file_path.exists():
        return HTMLResponse(content="File not found", status_code=404)
    html_content = file_path.read_text(encoding="utf-8")
    return HTMLResponse(content=html_content)
```

- Serves the frontend interface using FastAPI.
- Reads the HTML file from the `templates/` directory.

---

## 🌐 Frontend (HTML + JavaScript)

### `sse_index.html`

```html
<script>
    const eventSource = new EventSource("/sse/events");

    eventSource.onmessage = function(event) {
        const message = event.data;

        Swal.fire({
            title: '📨 New Message Received',
            text: message,
            icon: 'info',
            confirmButtonText: 'OK',
            timer: 3000,
            timerProgressBar: true
        });

        const messages = document.getElementById("messages");
        const newMessage = document.createElement("p");
        newMessage.className = "mb-2 text-sm text-gray-800 bg-gray-100 p-2 rounded-lg";
        newMessage.textContent = message;
        messages.appendChild(newMessage);
        messages.scrollTop = messages.scrollHeight;
    };
</script>
```

- Connects to the `/sse/events` endpoint using `EventSource`.
- Listens for real-time messages from the server.
- Displays each message in a SweetAlert modal and appends it to the message container.

---

## ⚙️ How It Works

1. Clients open an SSE connection to `/sse/events`.
2. Each connection is stored via an `asyncio.Queue`.
3. Messages sent to `/send` are distributed to all subscribers.
4. Each connected browser displays the new message in real-time.

---

## 🚀 Benefits of SSE

- **Simple & native** in browsers (no extra libraries needed)
- Ideal for **notifications**, **live logs**, **system status**, etc.
- Lightweight compared to WebSockets for one-way communication

---

## ⚠ Notes & Best Practices

- SSE uses a long-lived HTTP connection — ensure your **server and Docker setup** allow enough file descriptors (`ulimit -n`).
- Consider using a **Pub/Sub system (e.g., Redis)** in distributed systems.
- SSE is not ideal for bidirectional communication — use **WebSockets** for that.

---

## 📦 Example cURL to Send a Message

```bash
curl -X POST http://localhost:8000/sse/send      -H "Content-Type: application/json"      -d '{"message": "Hello from server!"}'
```

---

## ✅ Conclusion

This setup provides a clean, real-time experience using Server-Sent Events in FastAPI. It is a great fit for many real-time use-cases where simplicity and performance matter.

Feel free to customize the broadcasting logic, message filtering, or enhance it with Redis Pub/Sub for larger deployments.

---
