# System Architecture — Drone Security Analyst Agent

## 1. High-Level System Overview

```mermaid
graph TB
    subgraph INPUT["Input Sources"]
        V["MP4 / RTSP Video\n(demo_patrol.mp4)"]
        AU["AU-AIR Dataset\n(annotations.json + frames/)"]
    end

    subgraph LOADER["Data Preparation (au_air_loader.py)"]
        L1["Parse angle_shot_data\n(lat, lon, alt, yaw, speed)"]
        L2["Assign zones via GPS\nPolygon Geofence"]
        L3["Simulate timestamps\n(patrol shift schedule)"]
        L4["simulated_frames.json\nsimulated_telemetry.json"]
        L1 --> L2 --> L3 --> L4
    end

    subgraph LIVE["Live Pipeline (live_agent.py)"]
        VS["VideoFileSource / RTSP\n(rtsp_source.py)"]
        PL["Frame Pipeline\n(pipeline.py)"]
        VLM["VLM Analyzer\nGroq llama-3.2-11b-vision"]
        AE["Alert Engine\n(alert_engine.py)"]
        OT["Object Tracker\n(object_tracker.py)"]
        VS --> PL --> VLM --> AE --> OT
    end

    subgraph STORAGE["Persistence"]
        DB[("SQLite\nevents.db")]
        CH[("ChromaDB\nchroma_store/")]
        TR[("SQLite\ntracks table")]
    end

    subgraph API["API Layer (FastAPI)"]
        A1["GET /events"]
        A2["GET /alerts"]
        A3["GET /frames/search"]
        A4["GET /summary/today"]
        A5["POST /agent/ask"]
        A6["GET /alerts/stream (SSE)"]
        A7["GET /geofence"]
        A8["GET /tracks"]
    end

    subgraph UI["Dashboard (React + Vite)"]
        D1["Alert Feed\n+ severity filter"]
        D2["Frame Search\n(semantic)"]
        D3["Agent Q&A"]
        D4["Geofence View"]
    end

    AU --> LOADER
    LOADER --> LIVE
    V --> VS
    AE --> DB
    OT --> TR
    PL --> CH
    AE -.->|"SSE push"| A6
    DB --> API
    CH --> API
    TR --> API
    API --> UI
```

---

## 2. LangGraph State Machine (Batch Agent)

```mermaid
stateDiagram-v2
    [*] --> ingest_frame : start
    ingest_frame : Load next MergedEvent\nfrom merged_events[]
    analyze_frame : Send frame → Groq VLM\nReturn caption + objects + anomaly flag
    evaluate_alert : Check rules vs frame_history\nIf fired → Groq LLM alert message
    log_and_index : Write FrameEvent → SQLite\nWrite caption → ChromaDB\nIncrement index
    summarize : Query all events → Groq LLM\nGenerate 1-sentence daily summary

    ingest_frame --> analyze_frame
    analyze_frame --> evaluate_alert
    evaluate_alert --> log_and_index
    log_and_index --> ingest_frame : more frames
    log_and_index --> summarize : done
    summarize --> [*]
```

---

## 3. Live Video Pipeline

```mermaid
sequenceDiagram
    participant V as MP4 / RTSP
    participant VS as VideoFileSource
    participant GF as Polygon Geofence
    participant VLM as Groq VLM
    participant AE as Alert Engine
    participant OT as Object Tracker
    participant DB as SQLite
    participant CH as ChromaDB
    participant SSE as SSE Stream

    loop Every frame
        V->>VS: raw frame bytes
        VS->>VS: resize 640×480\nencode JPEG
        VS->>GF: (lat, lon) lookup
        GF-->>VS: zone name
        VS->>VLM: base64 frame + prompt
        VLM-->>AE: caption + objects + anomaly
        AE->>AE: evaluate rules\n(perimeter breach, loitering,\nunauth entry, repeat vehicle)
        AE->>DB: write FrameEvent row
        AE->>CH: index caption embedding
        AE->>OT: update vehicle track
        OT->>DB: write/update track row
        AE-->>SSE: publish alert payload
    end
```

---

## 4. Alert Rule Evaluation (Priority Order)

```mermaid
flowchart TD
    S([New FrameEvent]) --> R1

    R1{"zone = restricted_zone\nAND objects detected?"}
    R1 -->|Yes| A1[CRITICAL\nPerimeter Breach]
    R1 -->|No| R2

    R2{"human detected\nAND nighttime\nAND same zone ≥ 3 frames?"}
    R2 -->|Yes| A2[HIGH\nLoitering]
    R2 -->|No| R3

    R3{"nighttime\nAND gate zone\n(main_gate / side_gate)?"}
    R3 -->|Yes| A3[HIGH\nUnauthorized Entry]
    R3 -->|No| R4

    R4{"vehicle seen\n≥ 3× in frame history?"}
    R4 -->|Yes| A4[MEDIUM\nRepeated Vehicle]
    R4 -->|No| R5

    R5{"no objects\nAND daytime\nAND last 2 frames empty?"}
    R5 -->|Yes| A5[LOW\nIdle Detection]
    R5 -->|No| A6[NONE]

    A1 & A2 & A3 & A4 & A5 & A6 --> LLM{Alert level\n≠ NONE?}
    LLM -->|Yes| MSG["Groq LLM\ngenerates human-readable\nalert message"]
    LLM -->|No| STORE
    MSG --> STORE[(Write to SQLite\n+ SSE push)]
```

---

## 5. Polygon Geofence — Zone Assignment

```mermaid
graph LR
    subgraph GPS["Drone GPS (lat, lon)"]
        P["Point"]
    end

    subgraph ZONES["PropertyGeofence (Shapely polygons)"]
        Z1["garage\ntier: interior"]
        Z2["parking_lot\ntier: interior"]
        Z3["main_gate\ntier: perimeter"]
        Z4["side_gate\ntier: perimeter"]
        Z5["restricted_zone\ntier: restricted"]
    end

    P -->|"point-in-polygon\ntest (Shapely)"| Z1
    P --> Z2
    P --> Z3
    P --> Z4
    P --> Z5

    Z1 & Z2 & Z3 & Z4 & Z5 -->|"first match wins"| OUT["Zone Name\nassigned to frame"]
```

---

## 6. Dual Storage Model

```mermaid
graph TD
    FE["FrameEvent"]

    FE -->|"structured fields\nframe_id, zone, timestamp,\nalert_level, objects"| SQ[("SQLite — events.db\n\nGET /events?zone=\nGET /alerts?level=\nGET /summary/today")]

    FE -->|"VLM caption\n→ embedding vector"| CH[("ChromaDB — chroma_store/\n\nGET /frames/search?q=\nSemantic similarity search")]

    FE -->|"vehicle color\nhistogram + track_id"| TR[("SQLite — tracks table\n\nGET /tracks\nVehicle re-id across frames")]
```

---

## 7. Object Tracking (Vehicle Re-ID)

```mermaid
flowchart LR
    F["Frame with\nvehicle bbox"] --> H["Compute HSV\ncolor histogram"]
    H --> CMP{"Compare vs\nexisting tracks\n(cosine similarity)"}
    CMP -->|"similarity ≥ 0.75"| UPD["Update existing track\n+1 sighting_count\nupdate last_seen"]
    CMP -->|"similarity < 0.75"| NEW["Create new track\nnew UUID track_id"]
    UPD & NEW --> DB[("SQLite\ntracks table")]
```

---

## 8. API + Frontend Data Flow

```mermaid
graph LR
    subgraph FE["React Dashboard (Vite :5173)"]
        AF["AlertFeed\n(severity filter + SSE live)"]
        FS["FrameSearch\n(semantic query)"]
        QA["Agent Q&A\n(natural language)"]
        GV["Geofence View\n(zone polygons)"]
    end

    subgraph BE["FastAPI (:8000)"]
        R1["GET /alerts"]
        R2["GET /alerts/stream (SSE)"]
        R3["GET /frames/search"]
        R4["POST /agent/ask"]
        R5["GET /geofence"]
        R6["GET /tracks"]
        R7["GET /summary/today"]
    end

    AF <-->|"poll + SSE"| R1
    AF <-->|"live push"| R2
    FS <-->|"semantic query"| R3
    QA <-->|"LLM Q&A"| R4
    GV <-->| | R5

    R1 --> SQ[("SQLite")]
    R2 --> SSE["Alert Stream\n(asyncio queue)"]
    R3 --> CH[("ChromaDB")]
    R4 --> LLM["Groq LLM\nllama-3.3-70b-versatile"]
    R5 --> GJ["geofence.json"]
```

---

## Tech Stack Summary

| Layer | Technology | Purpose |
|---|---|---|
| Dataset | AU-AIR | Real aerial frames + per-frame telemetry |
| Video Input | OpenCV / RTSP | Read MP4 or live RTSP stream |
| Vision AI | Groq `llama-3.2-11b-vision-preview` | Frame captioning + object detection |
| LLM | Groq `llama-3.3-70b-versatile` | Alert messages + natural language Q&A |
| Agent Framework | LangGraph | Stateful looping graph for batch processing |
| Geofencing | Shapely | GPS point-in-polygon zone assignment |
| Object Tracking | OpenCV HSV histograms | Vehicle re-identification across frames |
| Alert Streaming | FastAPI SSE | Real-time push to dashboard |
| Relational DB | SQLite | Structured event + track storage |
| Vector DB | ChromaDB | Semantic search over VLM captions |
| API | FastAPI | REST + SSE backend |
| Dashboard | React (Vite) | Alert feed, search, Q&A, geofence UI |
| Config | python-dotenv | Environment variable management |
