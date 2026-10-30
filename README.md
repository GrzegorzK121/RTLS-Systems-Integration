# Integration of Ubisense and Pozyx for UWB-Based Indoor Localization

This project presents a software integration layer for two independent UWB-based Real-Time Location Systems: **Ubisense Dimension4** and **Pozyx**.

The application receives localization data from both systems, decodes them into a common internal structure, performs inter-system data fusion and provides a single position for a **virtual tracked object** represented by a pair of physical tags.

The project was developed as part of a master's thesis at **Gdańsk University of Technology**.

---

## Project goal

Pozyx and Ubisense Dimension4 operate as independent and hardware-incompatible localization systems. Each platform uses its own infrastructure, physical tags, localization engine and data format.

The purpose of this project was to integrate information produced by both systems at the software level and maintain localization continuity while the tracked object moves between their coverage areas.

The developed application is responsible for:

- receiving localization data from Pozyx and Dimension4,
- decoding different message formats,
- transforming coordinates into a common reference system,
- pairing one Pozyx tag and one Ubisense tag into one logical object,
- selecting or combining localization results depending on the current localization zone,
- using quality metrics provided by both systems,
- displaying the resulting position in real time,
- recording reference measurements for later analysis.

---

## Experimental RTLS infrastructure

The experimental environment contains two independent RTLS infrastructures.

The **Dimension4** system consists of reference nodes connected to the system server and synchronized within its own infrastructure.

The **Pozyx** system consists of four reference nodes connected to the Pozyx server through Ethernet and PoE.

Both systems provide localization results to a computer running the integration application.

![RTLS system topology](images/system_topology.png)

*Figure 1. Topology of the experimental Ubisense Dimension4 and Pozyx infrastructures.*

---

## Physical installation

### Pozyx

Four Pozyx reference nodes were physically installed in the test corridor.

![Pozyx reference node](images/pozyx_anchor.jpg)

*Figure 2. Pozyx reference node installed in the experimental environment.*

### Ubisense Dimension4

The Dimension4 infrastructure was installed in the second part of the corridor.

![Ubisense reference nodes](images/ubisense_anchors.jpg)

*Figure 3. Ubisense reference nodes installed in the test corridor.*

---

## Application architecture

The developed software layer receives data from both localization systems and converts them into a common representation used by the fusion engine.

Pozyx and Dimension4 use separate data acquisition paths:

- **Pozyx → MQTT / JSON**
- **Dimension4 → Ubisense OTW Protocol**

The decoded information is stored in the shared `current_state` structure. The fusion engine then calculates the final position of the logical tracked object. Results are delivered to browser interfaces through WebSocket communication and can also be stored in CSV files.

![Application architecture](images/application_architecture.png)

*Figure 4. Architecture of the developed inter-system RTLS data integration application.*

---

## Data acquisition

### Pozyx

Localization information from the Pozyx server is received using **MQTT**.

The application connects to the MQTT broker and subscribes to the `tags` topic. Messages are transmitted in **JSON** format.

The decoder extracts information including:

- tag identifier,
- X coordinate,
- Y coordinate,
- Z coordinate,
- `position_score`,
- timestamp-related information.

The coordinates are converted from millimetres to metres and transformed into the common coordinate system used by the integration layer.

---

### Ubisense Dimension4

Localization data from Dimension4 are obtained using the **Ubisense OTW (Over The Wire) Protocol**.

The application receives binary OTW frames with a fixed length of **40 bytes**.

The frame contains:

- system timestamp,
- tag identifier,
- measurement status,
- X coordinate,
- Y coordinate,
- Z coordinate,
- position variance,
- hardware timestamp.

Individual fields use different data types and byte orders. The application decodes the binary frame before converting the result to the common internal representation.

---

## Data normalization

After decoding, results from both systems are converted into a common structure.

Typical fields include:

```text
id
system
x
y
z
quality
raw_quality
timestamp
hw_timestamp
```

The latest measurements are stored in:

```python
current_state
```

This shared structure is then used by the fusion engine.

---

## Virtual tracked object

Pozyx and Dimension4 cannot use the same physical tag.

For this reason, the tracked object is represented by a pair consisting of:

- one Pozyx tag,
- one Ubisense tag.

Both physical tags are mounted next to each other and are assigned to one logical object in the application.

The mapping is stored in `config.py`.

Example:

```python
VIRTUAL_TAGS = {
    "Tester1": {
        "tag1_id": "<POZYX_TAG_ID>",
        "tag2_id": "<UBISENSE_TAG_ID>"
    }
}
```

The fusion engine treats this pair as one virtual tracked object.

---

## Fusion algorithm

The monitored corridor is divided into three localization zones:

1. **Pozyx zone**
2. **Common fusion zone**
3. **Dimension4 zone**

In the Pozyx zone, the final position is based on the Pozyx result.

In the common zone, both systems contribute to the final position.

In the Dimension4 zone, the final position is based on the Dimension4 result.

The common zone used during the experiments covers approximately:

```text
X = 20 m ... 24 m
```

The fusion process is illustrated below.

![Fusion algorithm](images/fusion_flowchart.png)

*Figure 5. Flowchart of the algorithm used to determine the final position of the virtual tracked object.*

---

## Quality-based weighted fusion

In the common zone, the position is calculated using quality-dependent weights.

For Pozyx, the quality value is obtained from:

```text
position_score
```

For Dimension4, the quality value is calculated from the variance contained in the OTW frame.

The weights are calculated as:

```text
Wp = Qp / (Qp + Qu)

Wu = Qu / (Qp + Qu)
```

where:

- `Qp` – Pozyx quality,
- `Qu` – Dimension4 quality,
- `Wp` – Pozyx weight,
- `Wu` – Dimension4 weight.

The resulting coordinates are calculated as:

```text
Xf = Xp * Wp + Xu * Wu

Yf = Yp * Wp + Yu * Wu
```

---

## Stale data detection

The application also verifies when each measurement was last updated.

If the time since the most recent update exceeds:

```text
1.5 s
```

the quality of that measurement is set to zero.

This prevents outdated data from participating in the fusion process.

---

## Main application modules

The backend is implemented in Python using **FastAPI**.

### `main.py`

Responsible for:

- starting the FastAPI application,
- serving browser interfaces,
- initializing background data receivers,
- handling WebSocket communication.

### `receivers.py`

Responsible for receiving localization information from both systems.

It contains:

```python
mqtt_listener()
```

for Pozyx and:

```python
UbisenseUDPProtocol
```

for Dimension4.

### `fusion_engine.py`

Responsible for:

- decoding Pozyx JSON messages,
- decoding Ubisense OTW frames,
- coordinate normalization,
- quality calculation,
- maintaining `current_state`,
- executing the data fusion algorithm.

### `recorder.py`

Responsible for:

- continuous data recording,
- reference point measurements,
- CSV export,
- storing Pozyx, Dimension4 and fused positions.

### `config.py`

Contains:

- network parameters,
- coordinate offsets,
- localization settings,
- mappings between Pozyx and Ubisense tags.

---

## Browser interfaces

The FastAPI backend provides several browser-based views.

### Main interface

The main browser interface displays the localization map and live readings from the connected systems.

![Main application interface](images/main_interface.png)

*Figure 6. Main browser interface of the RTLS integration application.*

---

### Mobile measurement interface

A dedicated mobile interface was created for measurements performed directly in the corridor.

It displays the latest readings from:

- Pozyx,
- Ubisense,
- the fusion algorithm.

The operator can select the known reference position and save the current measurement directly from a smartphone.

![Mobile measurement interface](images/mobile_interface.png)

*Figure 7. Mobile interface used during reference measurements.*

---

## Data recording

The application contains a dedicated measurement recording mechanism.

For each reference point, it can store:

- measurement time,
- logical object identifier,
- reference X coordinate,
- reference Y coordinate,
- Pozyx X and Y coordinates,
- Pozyx quality,
- Ubisense X and Y coordinates,
- Ubisense quality,
- fused X coordinate,
- fused Y coordinate,
- identifiers of both physical tags.

The results are stored in CSV files and can later be used for localization error analysis.

---

## Example processing flow

A typical application cycle is:

```text
Pozyx Server
    ↓
MQTT / JSON
    ↓
mqtt_listener()
    ↓
Pozyx decoder
    ↓
common structure
                 \
                  → current_state → FusionEngine → virtual object
                 /
Dimension4 Server
    ↓
Ubisense OTW Protocol
    ↓
UbisenseUDPProtocol
    ↓
OTW decoder
    ↓
common structure
```

The resulting position is then distributed through:

```text
FusionEngine
    ↓
WebSocket
    ↓
Browser interfaces
```

and:

```text
FusionEngine
    ↓
DataRecorder
    ↓
CSV files
```

---

## Technologies

### Backend

- Python
- FastAPI
- Uvicorn
- asyncio
- WebSocket

### Pozyx integration

- MQTT
- JSON

### Ubisense Dimension4 integration

- Ubisense OTW Protocol
- binary data decoding
- UDP datagram reception

### Frontend

- HTML
- CSS
- JavaScript
- Canvas

### Data processing

- CSV
- custom coordinate normalization
- weighted data fusion

---

## Repository structure

```text
.
├── README.md
├── main.py
├── fusion_engine.py
├── receivers.py
├── recorder.py
├── config.example.py
├── requirements.txt
│
├── static/
│   ├── index.html
│   ├── mobile.html
│   ├── mobile_map.html
│   ├── terminals.html
│   ├── zapis.html
│   └── style.css
│
└── images/
    ├── system_topology.png
    ├── application_architecture.png
    ├── fusion_flowchart.png
    ├── pozyx_anchor.jpg
    ├── ubisense_anchors.jpg
    ├── mobile_interface.png
    └── main_interface.png
```

---

## Installation

Install the required Python dependencies:

```bash
pip install fastapi uvicorn aiomqtt
```

Configure the application parameters in `config.py`.

Example:

```python
MQTT_BROKER = "<POZYX_SERVER_IP>"

UDP_IP = "0.0.0.0"
UDP_PORT = <UBISENSE_OTW_PORT>

VIRTUAL_TAGS = {
    "Tester1": {
        "tag1_id": "<POZYX_TAG_ID>",
        "tag2_id": "<UBISENSE_TAG_ID>"
    }
}
```

Start the application:

```bash
python main.py
```

The browser interface is served by the FastAPI/Uvicorn backend.

---

## Experimental evaluation

The application was tested in a corridor environment using known reference points.

Measurements were performed for multiple positions along the X axis and for different Y and Z configurations.

The collected data were used to analyse:

- X-axis localization error,
- Y-axis localization error,
- total two-dimensional position error,
- differences between localization zones,
- systematic coordinate offsets,
- the effect of coordinate correction.

---

## Project result

The developed application demonstrates the possibility of integrating localization results from two hardware-incompatible commercial UWB RTLS platforms at the software level.

The final system provides:

- simultaneous reception of Pozyx and Dimension4 data,
- decoding of MQTT/JSON and Ubisense OTW data,
- a common internal localization structure,
- pairing of two physical tags into one logical tracked object,
- quality-based localization fusion,
- continuous transition between localization zones,
- real-time browser visualization,
- mobile measurement support,
- CSV data recording for experimental analysis.

---

## Author

**Grzegorz Kaczorek**

Electronics and Telecommunications  
Gdańsk University of Technology

Master's thesis project focused on the integration of heterogeneous UWB RTLS systems.
