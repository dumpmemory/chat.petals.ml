# Petals Chat

A chat [web app](http://chat.petals.ml) + HTTP and WebSocket endpoints for BLOOM inference with the [Petals](https://petals.ml) client

## Interactive Chat

<div align="center">
<img src="https://i.imgur.com/p2nwiho.png" width="400px">
</div>

You can try it out [here](http://chat.petals.ml) or run the backend on your server using these commands:

```bash
git clone https://github.com/borzunov/chat.petals.ml.git
cd chat.petals.ml
pip install -r requirements.txt
flask run --host=0.0.0.0 --port=5000
```

In production, we recommend using gunicorn instead of the Flask dev server:

```bash
gunicorn app:app --bind 0.0.0.0:5000 --threads 100 --timeout 1000
```

> **Note:** Python 3.7+ required, and it is important to use `--threads` (not `--workers`) to ensure the HTTP API reuses inference sessions correctly.
Each session may use a separate thread, so you need to have as many threads as many concurrent users you'd like to support.

The chat uses the WebSocket API under the hood.

## APIs

The backend provides two APIs endpoints:

- [WebSocket API](#websocket-api-apiv2generate) (`/api/v2/generate`, recommended)
- [HTTP API](#http-api-apiv1) (`/api/v1/...`)

Please use the WebSocket API when possible - it is much faster, more powerful, and consumes less resources.

If you develop your own web app, you can use our endpoint at `http://chat.petals.ml/api/...` for research and development, then set up your own backend for production using the commands above.

> **Note:** We do not recommend using the endpoint at `http://chat.petals.ml/api/...` in production. It has a limited throughput, and we may pause or stop it any time.

### Backend's system requirements

- These requirements are for serving either [bigscience/bloom-petals](https://huggingface.co/bigscience/bloom-petals) **or**
  [bigscience/bloomz-petals](https://huggingface.co/bigscience/bloom-petals).
  Set the model you'd like to serve by updating [config.py](config.py) and [static/chat.js](static/chat.js).
  You need 2x of these requirements to serve both.

- For the generation speed of 1-2 sec/token, you need one of the following:
    - A GPU server with 10+ GB GPU VRAM
    - A CPU-only server with 20+ GB RAM (in this case, set `TORCH_DTYPE=torch.float32` in [config.py](config.py))
    - A CPU-only server with 10+ GB RAM and AVX512 support
        - Present on late Intel Xeon CPUs, e.g., on [DigitalOcean](https://digitalocean.com) droplets with a dedicated CPU
    - In future, we may implement using [faiss](https://github.com/facebookresearch/faiss) for generation.
        This would allow to use any CPU-only server with 8+ GB RAM for fast **approximate** greedy and top-k generation.

- For the generation speed of 3-4 sec/token, you need:
    - A CPU-only server with 10+ GB RAM

## WebSocket API (`/api/v2/generate`)

This API implies that you open a WebSocket connection and exchange JSON-encoded requests and responses.
This may be done from any programming language, see the example on Javascript:

```javascript
const ws = new WebSocket(`ws://${location.host}/api/v2/generate`);
ws.onopen = () => {
    ws.send(JSON.stringify({type: "open_inference_session", max_length: 1024}));
    ws.onmessage = event => {
        const response = JSON.parse(event.data);
        // TODO: Your code here
    };
};
```

The requests must follow this protocol:

### open_inference_session

The first request must be of type **open_inference_session** and include the `max_length` parameter (int, required)
and, optionally, the `model` (str) parameter (default: `config.DEFAULT_MODEL_NAME`).

The inference session created by this request is unique to this WebSocket connection and cannot be reused in other connections.
It is closed automatically when the connection is closed.

Request:

```javascript
{type: "open_inference_session", max_length: 1024}
```

Response:

```javascript
{ok: true}  // If successful
{ok: false, traceback: "..."}  // If failed
```

### generate

The next requests must be of type **generate** and include the same parameters as in the [/api/v1/generate HTTP API](#post-apiv1generate), except for `session_id` (it's identified by the connection you use in the WebSocket API).

A new feature of the WebSocket API is the `stop_sequence` parameter (str, optional). If you set it, the server will continue generation with the same parameters unless it generates the `stop_sequence`, so you may get multiple responses without having to send the request again and wait for the round trip's latency.

Intermediate responses contain the field `stop: false`, and the last response contains `stop: true`. For example, you can set `max_new_tokens: 1` and receive tokens one by one, as soon as they are generated. Check out the chat's [frontend code](static/chat.js) for a detailed example of how to do that.

Request:

```javascript
{type: "generate", "inputs": "A cat in French is \"", "max_new_tokens": 3}
```

Response (one or multiple):

```javascript
{ok: true, outputs: "chat\".", stop: true}  // If successful
{ok: false, traceback: "..."}  // If failed
```

## HTTP API (`/api/v1/...`)

### POST /api/v1/generate

Parameters:

- **model** (str, optional) - Model name. Default: `config.DEFAULT_MODEL_NAME`.
- **inputs** (str, optional) - New user inputs. May be omitted if you continue generation in an inference session (see below).
- **do_sample** (bool, optional) - If `0` (default), runs greedy generation. If `1`, performs sampling with parameters below.
- **temperature** (float, optional)
- **top_k** (int, optional)
- **top_p** (float, optional)
- **max_length** (int) - Max length of generated text (including prefix) in tokens.
- **max_new_tokens** (int) - Max number of newly generated tokens (excluding prefix).
- **session_id** (str, optional) - UUID of an inference session opened earlier (see methods below). This allows you to continue generation later without processing prefix from scratch.

Notes:

- You need to specify either `max_length` or `max_new_tokens`.
- If you'd like to solve downstream tasks in the zero-shot mode, start with `do_sample=0` (default).
- If you'd like to make a chat bot or write a long text, start with `do_sample=1, temperature=0.75, top_p=0.9` (tuning these params may help).

Returns (JSON):

- **ok** (bool)
- **outputs** (str)
- **traceback** (str) - the Python traceback if `ok == False`

### GET /api/v1/open_inference_session

Parameters:

- **max_length** (int, required)
- **model** (str, optional) - Model name. Default: `config.DEFAULT_MODEL_NAME`.

Notes:

- If you pass the `model` parameter, you must pass it to all calls of `/api/v1/generate` with this session too.

Returns (JSON):

- **ok** (bool)
- **session_id** (str) - UUID of the opened session
- **traceback** (str) - the Python traceback if `ok == False`

### GET /api/v1/close_inference_session

Parameters:

- **session_id** (str, required)

Returns (JSON):

- **ok** (bool)
- **traceback** (str) - the Python traceback if `ok == False`
