# VisionGuide AI — Backend and Alert Engine

Hackathon prototype for AI-based navigation and obstacle alerts for visually impaired people.

## Features
- FastAPI health endpoint and WebSocket `/ws`
- YOLO object detection
- Left / ahead / right image-region classification
- Image-based proximity heuristic (not actual distance)
- Multi-frame box-size trend heuristic (not a persistent object tracker)
- Navigation-context priority boost
- Per-connection cooldown and alert suppression
- `/describe` endpoint with local detector fallback
- Latency metric and basic tests

## Setup (Windows)
```powershell
python -m venv .venv
.venv\Scripts\activate
pip install -r requirements.txt
uvicorn backend.main:app --host 0.0.0.0 --port 8000
```

Health check: http://127.0.0.1:8000/health

Run tests from project root:
```powershell
pytest -q
```

## WebSocket protocol

Connect to `ws://HOST:8000/ws` on a trusted local network.

Send camera frame:
```json
{"type":"frame","image":"BASE64_ENCODED_JPEG"}
```

Successful result:
```json
{
  "type":"result",
  "detections":[
    {
      "label":"person",
      "confidence":0.91,
      "box":[200,30,400,400],
      "frame_width":640,
      "frame_height":480
    }
  ],
  "alert":{
    "type":"alert",
    "text":"Caution. Possible nearby obstacle ahead: person.",
    "priority":6,
    "object":"person",
    "direction":"ahead",
    "proximity_estimate":"near",
    "trend_estimate":"unknown",
    "confidence":0.91,
    "navigation_context":"",
    "key":["person","ahead"]
  },
  "latency_ms":125.4,
  "latency_scope":"backend decode, inference and alert logic"
}
```
Values above are examples, not measured results.

Send navigation context:
```json
{
  "type":"navigation",
  "instruction":"Turn left at the next street",
  "direction":"left",
  "active":true
}
```

Allowed `direction`: `left`, `ahead`, `right`, or `null`.
The navigation direction is a route hint, not a declaration that the path is safe. The camera orientation and navigation direction must be calibrated together.

Describe latest received frame:
```http
POST /describe
Content-Type: application/json

{"question":"What is around me?"}
```

## Scene description service
Without environment credentials, `/describe` reuses the local YOLO detector to list detected objects. To use a remote vision-language service, configure `WATSONX_DESCRIBE_URL` and `WATSONX_BEARER_TOKEN` for a team-managed adapter that accepts the schema described in `backend/describe.py`. These variables do not by themselves make this a direct IBM watsonx.ai API integration; adapt the client to the actual IBM API and deployment.

## Important limitations
- Bounding-box size is not a physical distance sensor.
- Trend estimation groups by object class and image region; it is not a real object tracker.
- COCO pretrained models do not reliably detect every curb, stair, pothole, or accessibility hazard. Use a suitable custom model and test it.
- The latency reported is backend processing latency, not end-to-end phone-to-audio latency.
- One shared model is used. For multiple concurrent users, serialize or queue inference safely and monitor load.
- Use a secure deployment, authentication, and encrypted transport outside a trusted demo network.
- This prototype is not a certified mobility aid and must not be the sole means of avoiding traffic or obstacles.
