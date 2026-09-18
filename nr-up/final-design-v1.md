## Problem with current MR shape

PDCP split into `nrup_{direct,f1ap,ue,handler}`. F1 vs direct is an **nr-up** concern, not PDCP.

---

## Target in a nutshell

```mermaid
flowchart TB
  subgraph PDCP["PDCP"]
    GNB["deliver_pdu_drb_gnb"]
    UE["deliver_pdu_drb_ue"]
  end
  subgraph NRUP["nr-up fn-ptr table"]
    IF["nr_up_if_t"]
    D["direct -> RLC<br/>(nr_up_monolithic.c)"]
    F["f1ap -> GTP<br/>(nr_up_f1u.c)"]
    U["nr_up_ue (extra, if feasible)"]
  end
  GNB --> IF
  UE --> IF
  IF --> D
  IF --> F
  IF --> U
```

**Summary:**
* PDCP calls nr-up only
* F1/direct chosen at `nr_up_init_*`
* no `nrup_direct` / `nrup_f1ap` PDCP splits
* Extra: UE also adopts nr-up for symmetry

---

## PDCP split: gNB vs UE

**PDCP design:** two libs only, `nr_pdcp_gnb` and `nr_pdcp_ue`, linked by CMake. Both share `nr_pdcp_oai_api.c` plus thin role TUs (`nr_pdcp_oai_api_gnb.c` / `_ue.c`) that own `deliver_pdu_drb_gnb` / `_ue` and call `nr_up_init_*`. Mono and F1/CU use the same gNB PDCP and the same deliver; only the nr-up backend differs (`nr_up_direct` vs `nr_up_f1ap`). Drop `nr_pdcp_nrup_{direct,f1ap,handler,ue}.c`.

```mermaid
flowchart LR
  subgraph COL1["gNB mono"]
    direction TB
    M1["nr-softmodem"]
    P1["PDCP lib: nr_pdcp_gnb<br/>nr_pdcp_oai_api.c<br/>nr_pdcp_oai_api_gnb.c<br/>cucp_cuup_*.c + core"]
    D1["deliver_pdu_drb_gnb"]
    U1["nr_up_direct"]
    M1 --> P1 --> D1 --> U1
  end
  subgraph COL2["gNB F1 / CU"]
    direction TB
    M2["nr-softmodem or nr-cuup"]
    P2["PDCP lib: nr_pdcp_gnb<br/>nr_pdcp_oai_api.c<br/>nr_pdcp_oai_api_gnb.c<br/>cucp_cuup_*.c + core"]
    D2["deliver_pdu_drb_gnb"]
    U2["nr_up_f1ap"]
    M2 --> P2 --> D2 --> U2
  end
  subgraph COL3["UE"]
    direction TB
    M3["nr-uesoftmodem"]
    P3["PDCP lib: nr_pdcp_ue<br/>nr_pdcp_oai_api.c<br/>nr_pdcp_oai_api_ue.c<br/>+ core"]
    D3["deliver_pdu_drb_ue"]
    U3["nr_up_ue"]
    M3 --> P3 --> D3 --> U3
  end
```

Note: core = `nr_pdcp_entity.c`, `nr_pdcp_ue_manager.c`, `nr_pdcp_sdu.c`, `nr_pdcp_timer_thread.c`, security/integrity (`nea*` / `nia*`), `asn1_utils.c`

| | Mono | F1 / CU | UE |
|--|------|---------|-----|
| **PDCP lib** | `nr_pdcp_gnb` | `nr_pdcp_gnb` | `nr_pdcp_ue` |
| **Deliver** | `deliver_pdu_drb_gnb` | `deliver_pdu_drb_gnb` | `deliver_pdu_drb_ue` |
| **nr-up** | `nr_up_direct` | `nr_up_f1ap` | `nr_up_ue` |

---

## How selected

* Guideline: **no PDCP role macros**
* gNB vs UE PDCP: CMake links `nr_pdcp_gnb` or `nr_pdcp_ue`
* mono vs F1 on softmodem: same binary links both backends (`nr_up_f1ap`, `nr_up_direct`), runtime `NODE_IS_*` picks init
* `nr-cuup`: links `nr_up_f1ap` only


### Linkage

```mermaid
flowchart TB
  SM["nr-softmodem"] -->|"links"| PG["nr_pdcp_gnb"]
  SM -->|"links"| D["nr_up_direct"]
  SM -->|"links"| F["nr_up_f1ap"]
  CU["nr-cuup"] -->|"links"| PG
  CU -->|"links"| F
  UE["nr-uesoftmodem"] -->|"links"| PU["nr_pdcp_ue"]
  UE -->|"links"| UU["nr_up_ue"]
```

### Init call chains

**nr-softmodem** (mono or F1: same entry, different `nr_up_init_*`)

```mermaid
sequenceDiagram
  participant EXE as softmodem<br/>(create_gNB_tasks / nr-softmodem.c)
  participant PDCP as PDCP<br/>(nr_pdcp_layer_init / nr_pdcp_oai_api_gnb.c)
  participant UP as NR-UP<br/>(nr_up_init_*)

  EXE->>PDCP: nr_pdcp_layer_init()
  alt NODE_IS_MONOLITHIC
    PDCP->>UP: nr_up_init_monolithic()
    Note over PDCP,UP: bind deliver_pdu_drb_gnb<br/>iface->deliver_drb = nr_up_mono_deliver_drb (RLC)
  else NODE_IS_CU
    PDCP->>UP: nr_up_init_f1u()
    Note over PDCP,UP: bind deliver_pdu_drb_gnb<br/>iface->deliver_drb = nr_up_f1u_deliver_drb (GTP)
  end
```

**nr-cuup**

```mermaid
sequenceDiagram
  participant EXE as cuup<br/>(main / nr-cuup.c)
  participant PDCP as PDCP<br/>(nr_pdcp_layer_init / nr_pdcp_oai_api_gnb.c)
  participant UP as NR-UP<br/>(nr_up_init_f1u / nr_up_f1u.c)

  EXE->>PDCP: nr_pdcp_layer_init()
  PDCP->>UP: nr_up_init_f1u()
  Note over PDCP,UP: bind deliver_pdu_drb_gnb<br/>iface->deliver_drb = nr_up_f1u_deliver_drb (GTP)
```

**nr-uesoftmodem**

```mermaid
sequenceDiagram
  participant EXE as uesoftmodem<br/>(main / nr-uesoftmodem.c)
  participant PDCP as PDCP<br/>(nr_pdcp_layer_init / nr_pdcp_oai_api_ue.c)
  participant UP as NR-UP<br/>(nr_up_init_ue)

  EXE->>PDCP: nr_pdcp_layer_init()
  PDCP->>UP: nr_up_init_ue()
  Note over PDCP,UP: bind deliver_pdu_drb_ue<br/>iface->deliver_drb = nr_up_ue deliver_drb (RLC)
```

---

## Sequence diagram

**Same API, different `deliver_drb`:** `nr_up_dl_transfer()` always dispatches `iface->deliver_drb`.

| | Mono | F1 (CU) |
|--|------|---------|
| Init | `nr_up_mono_deliver_drb` + precheck | `nr_up_f1u_deliver_drb` (+ precheck) |
| Transfer | RLC queue + consume | GTP send (+ consume) |

```mermaid
sequenceDiagram
  participant SDAP as SDAP<br/>(nr_sdap_entity.c)
  participant PDCP as PDCP
  participant UP as nr_up
  participant BE as backend

  Note over PDCP,BE: Init once: nr_up_init_* fills fn-ptrs
  Note over SDAP,BE: Hot path
  SDAP->>PDCP: nr_pdcp_data_req_drb
  PDCP->>UP: nr_up_dl_congestion_precheck
  PDCP->>PDCP: process_sdu
  PDCP->>PDCP: deliver_pdu_drb_gnb
  PDCP->>UP: nr_up_dl_transfer
  UP->>BE: mono RLC / F1 GTP
```

| Concern | Owner | Detail |
|---------|--------|--------|
| Congestion precheck **call** | PDCP | `nr_pdcp_data_req_drb` calls `nr_up_dl_congestion_precheck` before SN. |
| Deliver bind | PDCP | `drb_deliver = deliver_pdu_drb_*` at init, calls `nr_up_dl_transfer` |
| Congestion **policy** | nr-up backend | `iface->dl_congestion_precheck`: compare PDU size to cached **`available_tx_bytes`** (mono: from RLC TX space, F1: from DDDS `desired_buffer_size`). Stale cache (ALLOW after ~3 s) |
| RLC or GTP send + consume | nr-up backend | `iface->deliver_drb` |
| TS 38.425 lib | `nrup_lib` | encode/decode only |
