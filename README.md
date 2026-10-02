# SMK_v2 — Kịch bản và lưu đồ thuật toán toàn dự án

Viết theo **code hiện tại** (2026-10-02) của chìa (`key/`), ECU (`ecu/`) và lõi bảo mật (`vkey_core/`). Phần nào mới là thiết kế, chưa code, gom ở **Phần E**.

Thay cho `so-do-mermaid.md` và `so-do-de-hieu.md`, hai file đó vẽ theo kiến trúc cũ.

**Ký hiệu lưu đồ:**

| Ký hiệu | Cú pháp Mermaid | Ý nghĩa |
|---|---|---|
| Elip | `id(["..."])` | Bắt đầu / Kết thúc |
| Hình bình hành phải | `id[/"..."/]` | Nhập, chờ dữ liệu |
| Hình bình hành trái | `id[\"..."\]` | Xuất: LED, còi, gửi khung |
| Chữ nhật | `id["..."]` | Bước xử lý |
| Thoi | `id{"..."}` | Rẽ nhánh |
| Chữ nhật viền kép | `id[["..."]]` | Gọi lưu đồ khác |

**Mục lục**

- [0. Toàn cảnh](#0-toàn-cảnh)
- Phần A — Chìa: [A1 Vòng lặp chính](#a1-vòng-lặp-chính-của-chìa) · [A2 Thao tác nút](#a2-nhận-diện-thao-tác-nút) · [A3 Nút → lệnh](#a3-quyết-định-hành-động-theo-nút) · [A4 Gửi lệnh](#a4-gửi-lệnh-tới-xe) · [A5 Ghép chìa](#a5-ghép-chìa-phía-chìa) · [A6 Hiện diện](#a6-hiện-diện-phía-chìa-nghe-xe--gọi-xe--trả-lời-đo) · [A7 LED](#a7-đèn-led-của-chìa)
- Phần B — ECU: [B1 Khởi động](#b1-khởi-động-ecu) · [B2 Khung VKEY](#b2-xử-lý-một-khung-vkey-từ-chìa--app) · [B3 Ghép thiết bị](#b3-ghép-thiết-bị-phía-ecu-và-chia-slot) · [B4 Phiên đo](#b4-phiên-đo-khoảng-cách-channel-sounding) · [B5 GẦN/XA](#b5-kết-luận-gần--xa--trong-xe) · [B6 Trạng thái xe](#b6-máy-trạng-thái-xe) · [B7 Lệnh](#b7-luật-cho-lệnh-từ-chìa--app) · [B8 Hiện diện & timer](#b8-luật-hiện-diện-và-timer-nền) · [B9 Phanh](#b9-phanh-nhấp-giữ-đề) · [B10 Nút cốp](#b10-nút-cốp-trên-xe) · [B11 Chấp hành](#b11-cơ-cấu-chấp-hành) · [B12 CAN](#b12-đọc-can-dòng-minio) · [B13 Chống trộm](#b13-chống-trộm)
- [Phần C — App SmartKey](#phần-c--app-smartkey)
- [Phần D — Kịch bản từ đầu tới cuối](#phần-d--kịch-bản-sử-dụng-từ-đầu-tới-cuối)
- [Phần E — Chưa code](#phần-e--thiết-kế-chưa-code)

---

## 0. Toàn cảnh

```mermaid
flowchart LR
    subgraph KEY["CHÌA · nRF54L05 · CR2032"]
        KB["4 nút: LOCK · UNLOCK · TRUNK · WARNING"]
        KL["1 LED"]
        KR["BLE: nghe xe 45 ms / 5 s<br/>gọi xe bằng beacon ngắn<br/>reflector Channel Sounding"]
    end
    subgraph APP["APP SMARTKEY · điện thoại"]
        AP["Nút mở / khoá / cốp / tìm xe<br/>ghép, mời, cho mượn, thu hồi"]
    end
    subgraph ECU["ECU · nRF54L15"]
        EB["BLE: quảng bá 30 ms, +8 dBm<br/>GATT VKEY · initiator Channel Sounding"]
        EV["vkey_server: 8 slot, nonce, counter, HMAC"]
        ES["Máy trạng thái xe + luật"]
        EA["Cơ cấu chấp hành"]
        EI["Đầu vào: CAN · DOOR STATE · BT1 · BT-TRUNK"]
    end
    subgraph CAR["XE · MINIO"]
        CO["ACC · IGN · START · khoá trung tâm<br/>cốp · kính · hazard · còi"]
        CI["CAN 500 kbps · dây trạng thái khoá 2 cửa<br/>nút cốp"]
    end
    KR <-->|"lệnh VKEY ký HMAC<br/>đo khoảng cách mỗi 1 s"| EB
    AP <-->|"lệnh VKEY ký HMAC"| EB
    EB --> EV --> ES --> EA --> CO
    CI --> EI --> ES
```

| Tính năng | Ở đâu | Lưu đồ |
|---|---|---|
| Nút chìa → lệnh | chìa | A2, A3, A4 |
| Ghép chìa / app | chìa + ECU | A5, B3, C |
| Tự mở / tự khoá theo khoảng cách | chìa + ECU | A6, B4, B5, B8 |
| Xác thực lệnh | ECU | B2 |
| Khoá, mở, cốp, tìm xe, đề từ xa | ECU | B7, B11 |
| Đề bằng phanh | ECU | B9 |
| Timer nền (hết giờ, tự khoá) | ECU | B8 |
| Chống trộm | ECU | B13 |

---

# PHẦN A — CHÌA

## A1. Vòng lặp chính của chìa

Một luồng duy nhất làm mọi việc dùng radio (lệnh nút, ghép, link đo), nên các việc này không bao giờ chồng nhau.

```mermaid
flowchart TD
    S(["BẬT NGUỒN CHÌA"]) --> INIT["Khởi tạo: HAL VKEY, đọc khoá / counter / mã xe đã lưu<br/>LED, 4 nút, BLE"]
    INIT --> WAIT[/"Ngủ tới khi có sự kiện<br/>hoặc tới mốc giờ gần nhất: mốc nút 2 s / 5 s, mốc nghe xe"/]
    WAIT --> K{"Loại sự kiện?"}
    K -->|"Nút đổi trạng thái"| G[["Nhận diện thao tác nút · A2"]]
    K -->|"Tới mốc giờ"| G
    K -->|"Xe mở / mất link đo"| PL[["Xử lý link đo · A6"]]
    G --> D[["Quyết định hành động · A3"]]
    PL --> PM
    D --> ACT{"Hành động?"}
    ACT -->|"Gửi lệnh"| TX[["Gửi lệnh · A4"]]
    ACT -->|"Ghép chìa"| PR[["Ghép chìa · A5"]]
    ACT -->|"Từ chối: chưa ghép"| ER[\"LED lỗi 80 ms × 5"\]
    ACT -->|"Không làm gì"| PM
    TX --> DR["Bỏ các lần bấm xảy ra lúc đang bận"]
    PR --> DR
    DR --> PM
    ER --> PM
    PM[["Bước hiện diện: nghe xe / gọi xe · A6"]] --> WAIT
```

## A2. Nhận diện thao tác nút

Nút được chống dội 20 ms trong driver. Thao tác "giữ" phát **ngay lúc đủ thời gian**, không đợi nhả.

```mermaid
flowchart TD
    S(["Nút đổi trạng thái<br/>hoặc tới mốc giờ"]) --> ST{"Trạng thái"}
    ST -->|"RẢNH"| ONE{"Nhấn đúng 1 nút?"}
    ONE -->|"Đúng"| P["ĐANG GIỮ: nhớ nút, t0 = bây giờ"]
    ONE -->|"Sai: nhấn từ 2 nút"| W["CHỜ NHẢ: bỏ qua lần này"]
    ST -->|"ĐANG GIỮ"| WH{"Chuyện gì xảy ra?"}
    WH -->|"Giữ đủ 2 s"| H[\"Phát GIỮ 2 S"\]
    WH -->|"Đã phát GIỮ 2 S, giữ đủ 5 s"| L[\"Phát GIỮ 5 S"\]
    L --> W
    WH -->|"Nhả trước 2 s"| SH[\"Phát NHẤN"\]
    SH --> I["RẢNH"]
    WH -->|"Nhả trong 2–5 s"| N0["Không phát thêm"]
    N0 --> I
    WH -->|"Nhấn thêm nút khác"| W
    ST -->|"CHỜ NHẢ"| REL{"Đã nhả hết nút?"}
    REL -->|"Đúng"| I
    REL -->|"Sai"| E(["RA"])
    P --> E
    W --> E
    H --> E
    I --> E
```

## A3. Quyết định hành động theo nút

| Nút | NHẤN (dưới 2 s) | GIỮ 2 S | GIỮ 5 S |
|---|---|---|---|
| LOCK | Khoá (`LOCK_ALL`) | Khoá | — |
| UNLOCK | Mở (`UNLOCK_ALL`) | **Đề từ xa** (`REMOTE_START`) | — |
| TRUNK | Mở cốp (`TRUNK_RELEASE`) | Mở cốp | — |
| WARNING | **Tìm xe** (`FIND_CAR`) | không làm gì (đang đếm tới 5 s) | **Ghép chìa** |

```mermaid
flowchart TD
    S(["Có thao tác nút"]) --> P{"GIỮ 5 S của WARNING?"}
    P -->|"Đúng"| PAIR[["Ghép chìa · A5"]]
    P -->|"Sai"| C{"Thao tác ứng với một lệnh?<br/>theo bảng trên"}
    C -->|"Sai"| N(["Không làm gì"])
    C -->|"Đúng"| PD{"Chìa đã ghép với xe?"}
    PD -->|"Sai"| REJ[\"LED lỗi 80 ms × 5<br/>không phát gì"\]
    PD -->|"Đúng"| SEND[["Gửi lệnh · A4"]]
```

## A4. Gửi lệnh tới xe

```mermaid
flowchart TD
    S(["Gửi lệnh X"]) --> LED[\"LED nháy 100 / 100 ms"\]
    LED --> LNK{"Đang có link đo<br/>và kênh VKEY đang mở?"}
    LNK -->|"Đúng"| N1["Xin nonce mới từ xe"]
    N1 --> EX1[["Trao đổi lệnh · A4b"]]
    LNK -->|"Sai"| HOLD["Tạm dừng hiện diện, ngắt link cũ"]
    HOLD --> FIND{"Quét thấy quảng bá<br/>của xe trong 5 s?"}
    FIND -->|"Sai"| FAIL["Lỗi"]
    FIND -->|"Đúng"| CON{"Nối xe trong 3 s<br/>và mở kênh VKEY trong 3 s?"}
    CON -->|"Sai"| FAIL
    CON -->|"Đúng"| FR{"Nonce trong quảng bá<br/>nghe được chưa quá 3,5 s?"}
    FR -->|"Đúng"| EX2[["Trao đổi lệnh dùng nonce đó · A4b"]]
    FR -->|"Sai"| N2["Xin nonce mới"]
    N2 --> EX2
    EX2 --> DIS["Ngắt kết nối, bật lại hiện diện"]
    FAIL --> DIS
    EX1 --> RES{"Thành công?"}
    DIS --> RES
    RES -->|"Đúng"| OK[\"LED sáng 150 ms"\]
    RES -->|"Sai"| NG[\"LED lỗi 80 ms × 5"\]
    OK --> E(["RA"])
    NG --> E
```

### A4b. Trao đổi một lệnh (chính sách trả lời, giống app)

```mermaid
flowchart TD
    S(["Đã có nonce"]) --> CTR["counter = counter + 1<br/>GHI XUỐNG BỘ NHỚ TRƯỚC khi gửi"]
    CTR --> TAG["Ký: HMAC-SHA256 khoá slot, lệnh, counter, nonce<br/>lấy 12 byte làm tag"]
    TAG --> TX[\"Gửi khung CMD 20 byte"\]
    TX --> RX{"Xe trả lời?"}
    RX -->|"ACK OK hoặc STATUS"| OK(["Thành công"])
    RX -->|"ACK BUSY, lần đầu"| B["Chờ 350 ms"]
    B --> CTR
    RX -->|"ERR BAD_NONCE, lần đầu"| NN["Xin nonce mới"]
    NN --> CTR
    RX -->|"ERR REPLAY, lần đầu"| RP["counter = counter + 33, ghi xuống bộ nhớ"]
    RP --> TAG
    RX -->|"Không trả lời trong 3 s, lần đầu"| CTR
    RX -->|"BAD_TAG, LOCKED_OUT,<br/>hoặc cùng lỗi lần thứ hai"| NG(["Thất bại"])
```

> ECU từ chối lệnh (ví dụ xe chưa đỗ) cũng trả **ACK BUSY**, vì `vkey_core` không có mã lỗi riêng. Chìa thử lại một lần sau 350 ms rồi báo lỗi.

## A5. Ghép chìa (phía chìa)

```mermaid
flowchart TD
    S(["Giữ WARNING 5 s"]) --> LED[\"LED nháy 200 / 200 ms"\]
    LED --> H["Tạm dừng hiện diện, ngắt link"]
    H --> SC{"Quét thấy xe tên<br/>VKEY-SETUP-… hoặc VKEY-ENROLL-…<br/>trong 15 s?"}
    SC -->|"Sai"| NG
    SC -->|"Đúng"| C{"Nối và mở kênh VKEY<br/>trong 5 s?"}
    C -->|"Sai"| NG
    C -->|"Đúng"| K["Sinh cặp khoá X25519"]
    K --> R0["Gửi nửa khoá công khai thứ 1,<br/>nhận nửa thứ 1 của xe"]
    R0 --> R1["Gửi nửa thứ 2, nhận nửa thứ 2"]
    R1 --> KD["Tính khoá chung<br/>K_dev = HKDF khoá chung, mã xe"]
    KD --> CF["Gửi xác nhận confirm_p kèm yêu cầu slot:<br/>đã ghép đúng xe này → xin lại slot cũ<br/>còn lại → xin slot chìa trống bất kỳ"]
    CF --> AN{"Xe trả lời?"}
    AN -->|"ERR: đã đủ 3 chìa"| NG
    AN -->|"Im lặng: cửa sổ ghép chưa mở"| NG
    AN -->|"confirm_d + slot được cấp"| V{"confirm_d khớp?"}
    V -->|"Sai: có thể bị chen giữa"| NG
    V -->|"Đúng"| SV["Lưu khoá, counter = 32, mã xe"]
    SV --> OK[\"LED 400 - 200 - 400 ms"\]
    NG[\"LED lỗi 80 ms × 5"\] --> E(["Ngắt kết nối · RA"])
    OK --> E
```

## A6. Hiện diện phía chìa: nghe xe – gọi xe – trả lời đo

Chìa là **bên thu**: không phát gì cho tới khi nghe thấy xe. Không dùng RSSI; khoảng cách chỉ do xe đo bằng Channel Sounding.

```mermaid
flowchart TD
    S(["Bước hiện diện"]) --> C0{"Đã ghép, nút / ghép không giữ radio,<br/>chưa có link?"}
    C0 -->|"Sai"| E(["RA"])
    C0 -->|"Đúng"| B{"Đang phát beacon gọi xe?"}
    B -->|"Đúng, chưa hết 3 s"| E
    B -->|"Đúng, hết 3 s mà xe không nối"| STOP["Tắt beacon<br/>chờ 30 s mới được gọi lại"]
    STOP --> E
    B -->|"Sai"| DUE{"Tới giờ nghe?<br/>mỗi 5 s"}
    DUE -->|"Sai"| E
    DUE -->|"Đúng"| L[/"Nghe quảng bá của xe trong 45 ms"/]
    L --> HEARD{"Nghe thấy xe mình?"}
    HEARD -->|"Sai"| MISS["Đếm lần mất dấu<br/>3 lần liền → lần sau được gọi ngay"]
    MISS --> E
    HEARD -->|"Đúng"| OKC{"Được phép gọi?<br/>hết 30 s chờ, hoặc vừa mất dấu xe,<br/>hoặc vừa bấm nút"}
    OKC -->|"Sai"| E
    OKC -->|"Đúng"| CALL[\"Phát beacon nối được 100 ms một gói,<br/>tối đa 3 s"\]
    CALL --> E
```

Khi xe nối vào (lưu đồ B4 ở phía xe):

```mermaid
flowchart TD
    U(["Xe nối vào chìa"]) --> RF["Bật vai reflector Channel Sounding"]
    RF --> G["Mở kênh VKEY trên xe"]
    G --> P["Gửi GET_STATUS ký HMAC<br/>để chứng minh khoá"]
    P --> OK{"Xe chấp nhận?"}
    OK -->|"Sai"| D["Ngắt link"]
    OK -->|"Đúng"| R["Xe đo mỗi 1 s, mỗi lần 9 mẫu<br/>chìa chỉ phản hồi tone và gửi dữ liệu đo"]
    R --> DOWN(["Link kết thúc: rớt sóng,<br/>xe thả, xe chuyển sang chìa khác"])
    D --> DOWN
    DOWN --> W["Chờ 30 s mới gọi xe lại<br/>trừ khi mất dấu xe 3 lần hoặc bấm nút"]
```

## A7. Đèn LED của chìa

| Khi nào | Mẫu nháy |
|---|---|
| Đang gửi lệnh | 100 ms sáng / 100 ms tắt, lặp |
| Lệnh thành công | sáng 150 ms |
| Lỗi (chưa ghép, không thấy xe, xe từ chối…) | 80 ms × 5 |
| Đang ghép | 200 ms / 200 ms, lặp |
| Ghép xong | 400 – 200 – 400 ms |

---

# PHẦN B — ECU

## B1. Khởi động ECU

```mermaid
flowchart TD
    S(["BẬT NGUỒN ECU"]) --> O["Mọi ngõ ra về 0: ACC, IGN, START, còi, nguồn SIM,<br/>khoá, hazard, 4 dây kính, cầu H cốp"]
    O --> OK1{"Thành công?"}
    OK1 -->|"Sai"| ERR[\"Log lỗi, dừng"\]
    OK1 -->|"Đúng"| H["HAL VKEY, PSA crypto<br/>đọc khoá slot, counter, mã thiết bị"]
    H --> V["vkey_server + nhịp 100 ms:<br/>xoay nonce, hết khoá tạm, đóng cửa sổ ghép"]
    V --> B["Bật BLE, nạp bond"]
    B --> ADV["Quảng bá mỗi 30 ms, +8 dBm:<br/>tên VKEY-…, mã xe, nonce"]
    ADV --> CS[["Luồng đo khoảng cách · B4"]]
    CS --> PB[["Nút ghép BT1 + LED · B3"]]
    PB --> CAN{"MCP2515 sẵn sàng?"}
    CAN -->|"Sai"| W[\"Log: không có CAN,<br/>xe coi như đang tắt"\]
    CAN -->|"Đúng"| CL["CAN chỉ nghe, 500 kbps · B12"]
    W --> VS
    CL --> VS["Máy trạng thái xe: SLEEP, chống trộm BẬT<br/>luồng quét 50 ms · B6"]
    VS --> RUN(["CHẠY"])
```

## B2. Xử lý một khung VKEY từ chìa / app

```mermaid
flowchart TD
    S(["Khung tới kênh RX"]) --> LEN{"Dài 1–20 byte?"}
    LEN -->|"Sai"| DROP(["Bỏ"])
    LEN -->|"Đúng"| POL[["Kế hoạch slot cho khung ghép · B3"]]
    POL --> TY{"Loại khung?"}
    TY -->|"NONCE_REQ"| NR[\"Trả nonce hiện tại"\]
    TY -->|"ENROLL / ADMIN"| EN[["Ghép / quản trị · B3"]]
    TY -->|"CMD"| L{"Đang khoá tạm<br/>do sai tag?"}
    L -->|"Đúng"| ELO[\"ERR LOCKED_OUT + số giây còn lại"\]
    L -->|"Sai"| SL{"Slot đã ghép?"}
    SL -->|"Sai"| EK[\"ERR BAD_KEY_ID / NOT_ENROLLED"\]
    SL -->|"Đúng"| CM{"Lệnh hợp lệ?"}
    CM -->|"Sai"| EC[\"ERR BAD_CMD"\]
    CM -->|"Đúng"| CT{"counter lớn hơn lần trước?"}
    CT -->|"Sai"| ER[\"ERR REPLAY"\]
    CT -->|"Đúng"| TG{"Tag đúng với nonce hiện tại,<br/>hoặc nonce trước còn trong 2 s ân hạn?"}
    TG -->|"Sai: nonce hết hạn"| EN2[\"ERR BAD_NONCE"\]
    TG -->|"Sai: tag sai"| FC["Cộng 1 lần sai, đủ 5 lần thì<br/>khoá tạm 30 s, gấp đôi tới 15 phút"]
    FC --> ET[\"ERR BAD_TAG"\]
    TG -->|"Đúng"| BURN["Đốt nonce, lưu counter, xoá đếm sai"]
    BURN --> GS{"GET_STATUS?"}
    GS -->|"Đúng"| STA[\"STATUS: cửa, khoá, pin = 0"\]
    GS -->|"Sai"| ACT[["Luật xe · B7"]]
    ACT --> AR{"Luật nhận lệnh?"}
    AR -->|"Đúng"| AOK[\"ACK OK, đã thực hiện"\]
    AR -->|"Sai: từ chối / hàng đợi đầy"| ABZ[\"ACK BUSY"\]
    STA --> AUTH
    AOK --> AUTH
    ABZ --> AUTH
    AUTH{"Khung đến từ link đo<br/>và slot là chìa 5–7?"}
    AUTH -->|"Đúng"| GATE["Mở cổng đo khoảng cách · B4"]
    AUTH -->|"Sai"| ROT
    GATE --> ROT["Xoay nonce mới, cập nhật quảng bá"]
    ROT --> E(["RA"])
```

## B3. Ghép thiết bị (phía ECU) và chia slot

| Slot | Dành cho |
|---|---|
| 0 | App chủ xe (duy nhất ký được lệnh quản trị) |
| 1–4 | App máy phụ / người mượn |
| 5–7 | Chìa (tối đa 3) |

```mermaid
flowchart TD
    S(["Giữ BT1 trên xe 5 s"]) --> W["Mở cửa sổ ghép 60 s, LED nháy 250 / 250 ms<br/>tên quảng bá: chưa có app chủ → VKEY-SETUP-…<br/>đã có → VKEY-ENROLL-…"]
    W --> REQ[/"Thiết bị gửi 2 nửa khoá công khai X25519"/]
    REQ --> K["Xe sinh cặp khoá, trả 2 nửa của mình,<br/>tính khoá chung và K_dev"]
    K --> CFM[/"Thiết bị gửi confirm_p + slot muốn xin"/]
    CFM --> P{"Slot xin là gì?"}
    P -->|"Chìa: slot chìa trống bất kỳ"| FREE{"Còn slot 5–7 trống?"}
    FREE -->|"Sai"| RF[\"Từ chối: đã đủ 3 chìa"\]
    FREE -->|"Đúng"| PICK["Chọn slot trống nhỏ nhất trong 5–7"]
    P -->|"Chìa: xin lại slot cũ"| OLD["Dùng đúng slot cũ"]
    P -->|"App xin 5–7"| RF2[\"Từ chối: slot chìa chỉ cho chìa"\]
    P -->|"App xin 0–4"| CH{"Xe chưa có app chủ?"}
    PICK --> MODE["Chìa không bao giờ thành chủ:<br/>xe chưa có chủ vẫn ghép chìa như thiết bị thêm"]
    OLD --> MODE
    MODE --> VER
    CH -->|"Đúng"| OWN["Cấp slot 0 = chủ xe"]
    CH -->|"Sai"| ANY["Cấp đúng slot app xin"]
    OWN --> VER
    ANY --> VER
    VER{"confirm_p đúng?"}
    VER -->|"Sai"| BAD[\"ERR BAD_TAG, cộng 1 lần sai"\]
    VER -->|"Đúng"| SAVE["Lưu khoá slot, counter = 0,<br/>đóng cửa sổ ghép: mỗi lần giữ nút chỉ ghép 1 thiết bị"]
    SAVE --> RSP[\"Trả confirm_d + slot đã cấp"\]
    RSP --> E(["RA"])
```

- Không giữ nút thì xe **im lặng** với mọi yêu cầu ghép, kể cả xe mới chưa có chủ.
- App chủ còn có thể **mời** máy khác bằng QR (không cần ra xe), cho **mượn có hạn**, và **thu hồi** một slot (lệnh ADMIN chỉ slot 0 ký được). Lời mời xin slot 5–7 bị từ chối.

## B4. Phiên đo khoảng cách (Channel Sounding)

```mermaid
flowchart TD
    S(["Luồng đo"]) --> F{"Xe có chìa đã ghép?"}
    F -->|"Sai"| SL["Chờ 1 s"]
    SL --> F
    F -->|"Đúng"| SC{"Quét thấy beacon chìa của xe trong 10 s?<br/>bỏ qua chìa vừa chuyển đi"}
    SC -->|"Sai"| F
    SC -->|"Đúng"| CN["Nối vào chìa, khoảng kết nối 37,5 ms"]
    CN --> ST["Mã hoá link → đổi MTU → tìm dịch vụ RAS"]
    ST --> AU{"Chìa chứng minh khoá,<br/>slot 5–7, trong 10 s?"}
    AU -->|"Sai"| BF["Ngắt, nghỉ 10 s"]
    BF --> F
    AU -->|"Đúng"| CF["Vai initiator, đăng ký dữ liệu đo,<br/>đọc khả năng chìa, tạo cấu hình, bật bảo mật đo,<br/>lịch: 1 thủ tục mỗi 112,5 ms"]
    CF --> OK{"Thiết lập xong?"}
    OK -->|"Sai"| BF
    OK -->|"Đúng"| M[/"Chờ 1 lần đo = 9 mẫu, khoảng 1 s"/]
    M --> AVG["Trung bình các mẫu đo được"]
    AVG --> PR[["Kết luận GẦN / XA / TRONG XE · B5"]]
    PR --> C1{"Không GẦN và 15 s<br/>không lại gần thêm 1 m?"}
    C1 -->|"Đúng"| DROP["Thả chìa: ngắt link<br/>lúc đó đo thấy xa → tính là chìa đã đi"]
    C1 -->|"Sai"| C2{"Không GẦN, xe có từ 2 chìa,<br/>đã 10 s từ lần tìm trước?"}
    C2 -->|"Đúng"| PK{"Quét 1,5 s thấy<br/>chìa khác đang gọi xe?"}
    PK -->|"Đúng"| SW["Chuyển chìa: ngắt link này"]
    PK -->|"Sai"| LK
    C2 -->|"Sai"| LK{"Link còn?"}
    LK -->|"Đúng"| M
    LK -->|"Sai: 4 s không nghe thấy nhau"| END
    DROP --> END["Kết thúc phiên, báo XA"]
    SW --> END
    END --> F
```

## B5. Kết luận GẦN / XA / TRONG XE

```mermaid
flowchart TD
    S(["Khoảng cách d<br/>trung bình 9 mẫu"]) --> V{"d đo được?"}
    V -->|"Sai: cả 9 mẫu hỏng"| E(["Không kết luận"])
    V -->|"Đúng"| N{"Đang XA và d ≤ 1,5 m?"}
    N -->|"Đúng"| NEAR["→ GẦN"]
    NEAR --> EV1[\"Báo luật xe: chìa tới GẦN · B8"\]
    N -->|"Sai"| F{"Đang GẦN và d ≥ 4 m?"}
    F -->|"Đúng"| AWAY["→ XA"]
    AWAY --> EV2[\"Báo luật xe: chìa đi XA · B8"\]
    F -->|"Sai"| K["Giữ nguyên trạng thái:<br/>vùng 1,5–4 m chống khoá / mở liên tục"]
```

- **TRONG XE** (dùng khi giữ phanh để đề): lần đo gần nhất d < 0,5 m và không cũ quá 2 s.
- **Mất link** thì XA ngay.
- **Bằng chứng chìa ở xa** (cho luật tự khoá 17): đang đo thấy d ≥ 4 m, hoặc link rớt trong lúc xe đang mở khoá.

## B6. Máy trạng thái xe

| Trạng thái | ACC | IGN | Ý nghĩa |
|---|---|---|---|
| SLEEP | 0 | 0 | xe tắt, đã khoá |
| STANDBY | 0 | 0 | xe tắt, đã mở khoá |
| ON | 1 | 1 | bật điện, chưa sẵn sàng chạy |
| READY | 1 | 1 | đã đề, sẵn sàng chạy |
| REMOTE_ON | 1 | 1 | đề từ xa, cửa khoá, tối đa 10 phút |

```mermaid
stateDiagram-v2
    [*] --> SLEEP
    SLEEP --> STANDBY : UNLOCK hoặc chìa tới GẦN
    SLEEP --> REMOTE_ON : REMOTE_START, xe đỗ
    STANDBY --> SLEEP : LOCK, hoặc tự khoá luật 17
    STANDBY --> ON : nhấp phanh
    STANDBY --> READY : giữ phanh 2 s, chìa trong xe
    STANDBY --> REMOTE_ON : REMOTE_START, xe đỗ
    ON --> READY : giữ phanh 2 s, chìa trong xe
    ON --> SLEEP : LOCK, số P
    ON --> STANDBY : 30 phút, đứng yên, số P
    READY --> SLEEP : LOCK, số P
    READY --> READY : giữ phanh 2 s, CAN báo chết máy
    REMOTE_ON --> READY : giữ phanh 2 s, chìa trong xe
    REMOTE_ON --> SLEEP : LOCK số P, hoặc hết 10 phút
```

Hai điều kiện dùng chung:

- **Xe đỗ:** CAN còn tín hiệu thì số P hoặc phanh tay kéo; CAN im thì xe phải đang tắt (bus ngủ).
- **Được tắt xe:** CAN còn tín hiệu **và** số P. Riêng hết giờ đề từ xa mà mất CAN thì vẫn tắt.

## B7. Luật cho lệnh từ chìa / app

```mermaid
flowchart TD
    S(["Lệnh đã xác thực"]) --> C{"Lệnh?"}
    C -->|"LOCK"| L1{"Xe đỗ?"}
    L1 -->|"Sai"| REJ
    L1 -->|"Đúng"| L2{"Xe đang bật?"}
    L2 -->|"Sai"| LA["Khoá, lên kính 5 s<br/>→ SLEEP, chống trộm BẬT"]
    L2 -->|"Đúng"| L3{"Được tắt xe?"}
    L3 -->|"Sai"| REJ
    L3 -->|"Đúng"| LB["Tắt ACC + IGN, chờ 1 s, khoá, lên kính 5 s<br/>→ SLEEP, chống trộm BẬT"]
    C -->|"UNLOCK"| U1{"Xe đỗ?"}
    U1 -->|"Sai"| REJ
    U1 -->|"Đúng"| UA["Chống trộm TẮT, im còi nếu đang kêu, mở khoá<br/>xe đang tắt → STANDBY, đếm lại 10 s ân hạn"]
    C -->|"TRUNK"| T1{"Xe đỗ?"}
    T1 -->|"Sai"| REJ
    T1 -->|"Đúng"| TA["Chạy cầu H cốp"]
    C -->|"FIND_CAR"| F1{"Xe đang tắt?"}
    F1 -->|"Sai"| REJ
    F1 -->|"Đúng"| FA["Bật hazard, còi 3 tiếng,<br/>5 s sau tắt hazard"]
    C -->|"REMOTE_START"| R1{"Xe đang tắt và đỗ?"}
    R1 -->|"Sai"| REJ
    R1 -->|"Đúng"| RA["Khoá, bật ACC, 500 ms, bật IGN<br/>→ REMOTE_ON 10 phút, chống trộm BẬT"]
    REJ[\"Từ chối: chìa / app nhận BUSY"\]
```

## B8. Luật hiện diện và timer nền

```mermaid
flowchart TD
    S(["Chìa đổi GẦN / XA"]) --> D{"Sự kiện"}
    D -->|"tới GẦN"| N{"Trạng thái xe"}
    N -->|"SLEEP"| N1["Mở khoá, chống trộm TẮT → STANDBY"]
    N -->|"REMOTE_ON"| N2["Mở khoá, chống trộm TẮT,<br/>đếm lại 10 phút"]
    N -->|"Khác"| N3["Không làm gì"]
    D -->|"đi XA"| A{"Trạng thái xe"}
    A -->|"SLEEP / STANDBY"| A1["Lên kính 5 s"]
    A -->|"REMOTE_ON"| A2["Khoá lại, xe vẫn chạy"]
    A -->|"ON / READY"| A3{"CAN còn, số P, xe đứng yên?"}
    A3 -->|"Đúng"| A4["Chỉ khoá cửa, không tắt xe"]
    A3 -->|"Sai"| N3
```

```mermaid
flowchart TD
    S(["Mỗi 50 ms"]) --> T{"Trạng thái xe"}
    T -->|"STANDBY"| T1{"Đã 10 s từ lúc mở khoá,<br/>chìa không GẦN,<br/>có bằng chứng chìa ở xa?"}
    T1 -->|"Đúng"| T1A["Khoá, kể cả khi cửa đang mở<br/>→ SLEEP, chống trộm BẬT"]
    T1 -->|"Sai"| NO["Không làm gì"]
    T -->|"REMOTE_ON"| T2{"Đủ 10 phút?"}
    T2 -->|"Đúng"| T2B{"Được tắt xe, hoặc mất CAN?"}
    T2B -->|"Đúng"| T2A["Tắt ACC + IGN → SLEEP"]
    T2B -->|"Sai: chưa về số P"| NO
    T2 -->|"Sai"| NO
    T -->|"ON"| T3{"Đủ 30 phút, đứng yên,<br/>được tắt xe?"}
    T3 -->|"Đúng"| T3A["Tắt ACC + IGN → STANDBY"]
    T3 -->|"Sai"| NO
    T -->|"SLEEP / READY"| NO
```

## B9. Phanh: nhấp, giữ, đề

Tín hiệu phanh lấy từ CAN, quét mỗi 50 ms.

```mermaid
flowchart TD
    S(["Phanh đổi trạng thái"]) --> G{"Cử chỉ"}
    G -->|"Nhấp: nhả trước 2 s"| TP{"Đang STANDBY?"}
    TP -->|"Đúng"| ON["Bật ACC, 500 ms, bật IGN → ON"]
    TP -->|"Sai"| NO["Bỏ qua, không báo"]
    G -->|"Giữ đủ 2 s, phát 1 lần"| H0{"Xe đỗ?"}
    H0 -->|"Sai"| NO
    H0 -->|"Đúng"| H1{"Trạng thái xe"}
    H1 -->|"READY"| H2{"CAN báo chết máy:<br/>READY_state = 1?"}
    H2 -->|"Đúng"| CR2["Đề lại: ACC, IGN, xung START 1,5 s"]
    H2 -->|"Sai"| NO
    H1 -->|"STANDBY / ON / REMOTE_ON"| H3{"Chìa TRONG XE, dưới 0,5 m?"}
    H3 -->|"Sai"| NO
    H3 -->|"Đúng"| CR["ACC, 500 ms, IGN, 500 ms,<br/>xung START 1,5 s → READY"]
    H1 -->|"SLEEP"| NO
```

## B10. Nút cốp trên xe

```mermaid
flowchart TD
    S(["Nút BT-TRUNK nhấn ổn định 100 ms"]) --> P{"Xe đỗ?"}
    P -->|"Sai"| NO(["Bỏ qua"])
    P -->|"Đúng"| U{"Xe đang mở khoá,<br/>hoặc chìa GẦN?"}
    U -->|"Sai"| NO
    U -->|"Đúng"| T["Chạy cầu H cốp"]
```

## B11. Cơ cấu chấp hành

Một luồng duy nhất lấy từng việc trong hàng đợi và chạy theo **thứ tự cố định**:

```mermaid
flowchart TD
    S(["Lấy 1 việc từ hàng đợi"]) --> AO{"Tắt báo động?"}
    AO -->|"Đúng"| AO1["Dừng còi, phát xung tắt hazard"]
    AO -->|"Sai"| PO
    AO1 --> PO{"Tắt điện?"}
    PO -->|"Đúng"| PO1["START, IGN, ACC = 0<br/>có kèm khoá thì chờ 1 s"]
    PO -->|"Sai"| LK
    PO1 --> LK{"Khoá?"}
    LK -->|"Đúng"| LK1[["Xung khoá"]]
    LK -->|"Sai"| UL
    LK1 --> UL{"Mở khoá?"}
    UL -->|"Đúng"| UL1[["Xung mở khoá"]]
    UL -->|"Sai"| PW
    UL1 --> PW{"Bật điện hoặc đề?"}
    PW -->|"Đúng"| PW1["ACC = 1, 500 ms, IGN = 1<br/>đề: thêm 500 ms, START 1,5 s"]
    PW -->|"Sai"| TR
    PW1 --> TR{"Cốp?"}
    TR -->|"Đúng"| TR1["Cầu H: chiều B 300 ms → cả hai 80 ms<br/>→ chiều F 300 ms → nghỉ"]
    TR -->|"Sai"| WN
    TR1 --> WN{"Lên kính?"}
    WN -->|"Đúng"| WN1["UP phải + UP trái, tắt DOWN cùng bên trước<br/>hẹn 5 s thì nhả"]
    WN -->|"Sai"| FD
    WN1 --> FD{"Tìm xe?"}
    FD -->|"Đúng"| FD1["Bật hazard nếu chưa, còi 3 × 300 ms,<br/>hẹn 5 s tắt hazard"]
    FD -->|"Sai"| AL
    FD1 --> AL{"Báo động?"}
    AL -->|"Đúng"| AL1["Bật hazard nếu chưa,<br/>còi 500 / 500 ms liên tục"]
    AL -->|"Sai"| E(["Xong, lấy việc tiếp"])
    AL1 --> E
```

Xung khoá / mở (dây khoá trung tâm là kiểu bấm-nhả):

```mermaid
flowchart TD
    S(["Muốn khoá hoặc mở"]) --> K{"Trạng thái khoá hiện tại đã đúng?<br/>theo CAN cửa lái, CAN im thì ECU tự nhớ"}
    K -->|"Đúng"| E(["Không phát xung"])
    K -->|"Sai"| P["Kéo dây khoá trung tâm xuống GND 150 ms<br/>mỗi xung đảo trạng thái khoá ↔ mở"]
    P --> N["Nhớ trạng thái mới,<br/>bỏ qua giá trị khoá từ CAN trong 1,5 s"]
    N --> E
```

## B12. Đọc CAN (dòng MINIO)

```mermaid
flowchart TD
    S(["Khung CAN tới, chế độ chỉ nghe"]) --> T["Ghi thời điểm: CAN còn = có khung trong 2 s"]
    T --> ID{"ID khung"}
    ID -->|"0x2BF"| A["Tốc độ: dưới 1 km/h = đứng yên"]
    ID -->|"0x155"| B["Số: 2 = P"]
    ID -->|"0x201"| C["Phanh chân"]
    ID -->|"0x4D4"| D["Cửa lái mở / đóng<br/>khoá cửa lái: 5 = khoá, 0 = mở<br/>bỏ qua 1,5 s sau xung của ECU"]
    ID -->|"0x4C5"| F["Phanh tay, READY_state"]
    ID -->|"Khác"| G["Bỏ qua"]
```

ID và bit lấy từ dự án `main`, cần log khung thô trên xe thật để xác nhận.

## B13. Chống trộm

- **Bật:** mỗi lần ECU khoá xe (lệnh LOCK, tự khoá, đề từ xa…), và ngay khi ECU khởi động.
- **Tắt:** mở khoá hợp lệ, tức lệnh UNLOCK từ chìa/app, hoặc xe tự mở khi chìa tới GẦN.
- **Đầu vào:** dây DOOR STATE (P1.07) báo trạng thái **khoá** của cả 2 cửa: 12 V = cả 2 đã khoá, 0 V = có cửa đang mở khoá.

```mermaid
flowchart TD
    S(["DOOR STATE đổi mức, ổn định 100 ms"]) --> E1{"12 V → 0 V:<br/>có cửa vừa bị mở khoá?"}
    E1 -->|"Sai"| X(["Không làm gì"])
    E1 -->|"Đúng"| A{"Chống trộm đang BẬT?"}
    A -->|"Sai: vừa mở bằng chìa / app"| X
    A -->|"Đúng"| K{"Đang kêu rồi?"}
    K -->|"Đúng"| X
    K -->|"Sai"| F{"Chìa GẦN hoặc TRONG XE?"}
    F -->|"Đúng: chủ xe đang ở đó"| X
    F -->|"Sai"| AL[\"KÊU: hazard bật,<br/>còi 500 / 500 ms liên tục"\]
    AL --> W[/"Chờ mở khoá hợp lệ:<br/>UNLOCK từ chìa / app, hoặc chìa tới GẦN"/]
    W --> OFF["Tắt còi, tắt hazard, chống trộm TẮT"]
```

Lệnh mở khoá hợp lệ tắt chống trộm **ngay lúc ECU nhận lệnh**, trước khi dây DOOR STATE kịp xuống 0 V, nên mở bằng chìa/app không bao giờ làm kêu.

---

# PHẦN C — APP SMARTKEY

App dùng đúng giao thức và lõi bảo mật như chìa (`vkey_core` giống từng byte). App **không** tự mở cửa theo khoảng cách; muốn mở phải bấm.

```mermaid
flowchart TD
    S(["Mở màn hình xe"]) --> SC[/"Quét quảng bá: company 0xFFFF, phiên bản 1,<br/>đúng mã xe"/]
    SC --> CN["Nối, không bond, không đổi MTU<br/>bật notify kênh TX"]
    CN --> GS["Gửi GET_STATUS, hiện cửa / khoá"]
    GS --> BT[/"Người dùng bấm Mở / Khoá / Cốp / Tìm xe"/]
    BT --> EX[["Trao đổi lệnh · A4b"]]
    EX --> ECU[["ECU xử lý · B2 → B7"]]
    ECU --> RES[\"Hiện kết quả<br/>BUSY lặp lại = xe từ chối hoặc bận"\]
    RES --> BT
```

| Việc | Cách làm |
|---|---|
| Ghép app làm chủ | Giữ BT1 trên xe 5 s → app thấy `VKEY-SETUP-…` → ghép → slot 0 |
| Thêm điện thoại | Giữ BT1 5 s → app thấy `VKEY-ENROLL-…` → ghép vào slot 1–4; hoặc app chủ tạo QR mời |
| Cho mượn có hạn | App chủ tạo QR có hạn; hết hạn thì app chủ tự thu hồi khi lại gần xe |
| Thu hồi một thiết bị | App chủ gửi lệnh ADMIN `REVOKE_SLOT` (kể cả chìa) |
| Đề từ xa | App chưa có nút; lệnh này hiện chỉ gửi từ chìa |

---

# PHẦN D — KỊCH BẢN SỬ DỤNG TỪ ĐẦU TỚI CUỐI

## D1. Đi tới xe, tự mở, lên xe đề máy

```mermaid
sequenceDiagram
    participant N as Người cầm chìa
    participant K as Chìa
    participant E as ECU
    participant X as Xe
    loop mỗi 5 s
        K->>K: nghe quảng bá 45 ms
    end
    K->>E: nghe thấy xe, phát beacon gọi xe tối đa 3 s
    E->>K: nối vào, mã hoá link
    K->>E: GET_STATUS ký HMAC, chứng minh khoá
    E->>K: bắt đầu đo, mỗi 1 s lấy 9 mẫu
    N->>N: đi tới xe
    E->>E: trung bình ≤ 1,5 m, kết luận GẦN
    E->>X: xung mở khoá 150 ms, chống trộm TẮT, STANDBY
    N->>X: mở cửa, lên xe, giữ phanh 2 s
    X->>E: CAN báo phanh
    E->>E: chìa trong xe dưới 0,5 m
    E->>X: ACC, 500 ms, IGN, 500 ms, START 1,5 s
    E->>E: READY
```

## D2. Đỗ xe rồi đi khỏi

```mermaid
sequenceDiagram
    participant N as Người cầm chìa
    participant E as ECU
    participant X as Xe
    alt Bấm LOCK trên chìa
        N->>E: LOCK
        E->>X: số P: tắt ACC + IGN, 1 s, khoá, lên kính 5 s
        E->>E: SLEEP, chống trộm BẬT
    else Quên tắt, xe vẫn READY, đi xa
        E->>E: trung bình ≥ 4 m, kết luận XA
        E->>X: số P và đứng yên: chỉ khoá cửa, xe vẫn chạy
    else Xe đang tắt, đã mở khoá, đi xa
        E->>X: XA: lên kính 5 s
        E->>X: đủ 10 s từ lúc mở và có bằng chứng xa: khoá
        E->>E: SLEEP, chống trộm BẬT
    end
    E->>E: chìa không gần, đứng yên 15 s: thả link
```

## D3. Đề từ xa rồi ra xe

```mermaid
sequenceDiagram
    participant N as Người cầm chìa
    participant K as Chìa
    participant E as ECU
    participant X as Xe
    N->>K: giữ UNLOCK 2 s
    K->>E: REMOTE_START
    E->>X: khoá, ACC, 500 ms, IGN
    E->>E: REMOTE_ON, đếm 10 phút, chống trộm BẬT
    N->>N: đi ra xe
    K->>E: nghe thấy xe, gọi xe, xe nối và đo
    E->>X: tới GẦN: mở khoá, đếm lại 10 phút
    N->>X: lên xe, giữ phanh 2 s, chìa trong xe
    E->>X: xung START, READY
```

## D4. Bấm nút từ xa khi chưa có link đo

```mermaid
sequenceDiagram
    participant K as Chìa
    participant E as ECU
    K->>K: tạm dừng hiện diện
    K->>E: quét quảng bá, lấy nonce, tối đa 5 s
    K->>E: nối, mở kênh VKEY
    K->>E: CMD ký HMAC, nonce từ quảng bá nếu còn mới
    E->>E: kiểm tra, chạy luật, xếp việc cho actuator
    E->>K: ACK OK, hoặc BUSY nếu từ chối
    K->>K: ngắt, LED OK hoặc lỗi, bật lại hiện diện
```

## D5. Ghép xe mới: app chủ và chìa, thứ tự nào cũng được

```mermaid
sequenceDiagram
    participant O as Chủ xe
    participant A as App
    participant K as Chìa
    participant E as ECU
    O->>E: giữ BT1 5 s, cửa sổ ghép 60 s
    A->>E: ghép, xin slot 0
    E->>A: cấp slot 0, xe có chủ, đóng cửa sổ
    O->>E: giữ BT1 5 s lần nữa
    O->>K: giữ WARNING 5 s
    K->>E: ghép, xin slot chìa trống
    E->>K: cấp slot 5, đóng cửa sổ
    Note over E: ghép chìa trước app cũng được:<br/>chìa vào slot 5–7, slot 0 vẫn để cho app
```

## D6. Kẻ gian mở khoá bằng chìa cơ

```mermaid
sequenceDiagram
    participant T as Kẻ gian
    participant X as Xe
    participant E as ECU
    participant O as Chủ xe
    Note over E: xe đã khoá, chống trộm BẬT
    T->>X: mở khoá bằng chìa cơ
    X->>E: DOOR STATE 12 V xuống 0 V
    E->>E: không có lệnh mở khoá hợp lệ, chìa không ở gần
    E->>X: hazard bật, còi 500 / 500 ms liên tục
    T->>X: vào xe, giữ phanh
    E->>E: chìa không trong xe, không đề
    O->>E: bấm UNLOCK trên chìa hoặc app
    E->>X: tắt còi, tắt hazard, chống trộm TẮT
```

---

# PHẦN E — THIẾT KẾ CHƯA CODE

| Nội dung | Tài liệu | Ghi chú |
|---|---|---|
| Lệnh từ cả app và chìa: tự khoá phân biệt ai mở (17 ✏️), khoá lại sau 30 s không mở cửa (39), thu hồi chìa thì ngắt link đo (38) | [ecu-tinh-nang.md](ecu-tinh-nang.md) mục 6, câu G–J | Chia slot 5–7 cho chìa và chỉ chìa được đo (37) **đã code** |
| Kịch bản dùng chung app + chìa: tự khoá khi người khác dùng app, khoá từ app khi chìa trong xe, khoá khi hết giờ đề từ xa, factory reset | [kich-ban-app-chia.md](kich-ban-app-chia.md), câu L–P | Câu K đã chốt: muốn đề phải có chìa trong xe |
| Chống trộm phần còn lại: kêu khi cửa lái mở khoá qua CAN, khi đạp phanh hoặc bật điện trái phép, bấm LOCK để im còi, gửi SMS | [ecu-tinh-nang.md](ecu-tinh-nang.md) mục 7, câu Q–V | Bản tối giản (B13) đã code |
| Lấy trạng thái khoá từ DOOR STATE thay vì CAN | — | Đề xuất; tránh xung khoá sai khi CAN ngủ |
| Module SIM A7670C | — | Đang tắt nguồn |
