---
client: Maybank Securities
clientShort: MSVN
project: Vero Monitor Service (VMS)
projectCode: VMS-MSVN-2026
kicker: Đề xuất triển khai
date: 19/05/2026
validUntil: 22/06/2026
prepared: Tony
cover: img/proposal-cover.png
coverEyebrow: Proposal · 2026 · Confidential
coverTitle: Vero Monitor Service
coverSubtitle: Chuẩn hoá khả năng theo dõi hệ thống end-to-end cho Maybank Securities
---

# I. Tóm tắt hiện trạng

Maybank Securities (MSVN) đang vận hành **3 khối hệ thống** (Core / WEB / Mobile) do 3 nhà cung cấp (Freewill / Innotech / Goline) — và các hệ thống vệ tinh như Gateway Sở giao dịch HSX, HNX qua FixGW, MDDS, SFTP và Bank Connection.

## Hiện trạng kiến trúc

Để thống nhất term và scope dự án, Vero ghi nhận hiện trạng MSVN theo 3 khối kiến trúc như sau (nguồn: tài liệu khảo sát Core/WEB/Mobile, file Survey System Maybank và sơ đồ PROD Network Diagram 2025-05-30).

**Bảng 1.** Hiện trạng kiến trúc 3 khối Maybank Securities

| Khối | Sản phẩm | Vendor | Kiến trúc | DB / MQ | Stack theo dõi |
|---|---|---|---|---|---|
| **Core** | iFIS Equity + iFIS Derivative + SBA | Freewill | Monolithic, Active-Standby thủ công | DB2 + Informix; ActiveMQ | Không có log center / dashboard |
| **WEB** | KeTrade Web Trading | Innotech | Monolithic, F5 Least Connection | MS SQL Always On; ActiveMQ | Chưa có End-to-End Monitoring |
| **Mobile** | Maybank Trade Mobile | Goline | 50+ microservices trên Docker Swarm | Oracle RAC; Kafka + Redis | Prometheus, Grafana, Jaeger, ELK |

:::callout{kind="info" title="Hệ thống vệ tinh"}
HSX và HNX qua **FixGW** (FIX TCP/IP) + **MDDS** (FIX multicast) + **SFTP**; VSD lưu ký; EIB, BIDV, TCB, VTB cấp tiền/đối soát; Vietstock, TradingView, Fidessa, ECD, SMS/Email vendor, Smart OTP.
:::

## Pain Point xác định qua khảo sát

Qua khảo sát, MSVN đang gặp các pain point chính: **downtime hệ thống giao dịch, database bottleneck, thiếu monitoring real-time, log phân tán, deploy thủ công Core/WEB, incident response phụ thuộc vendor**.

**Bảng 2.** Pain Point xác định qua khảo sát

| # | Pain Point | Mức | Mô tả ngắn |
|:---:|---|---|---|
| **P1** | Downtime hệ thống giao dịch | **Cao** | Thiếu regression/automation test, phụ thuộc vendor khi xử lý sự cố |
| **P2** | Database bottleneck | **Cao** | Query chậm, deadlock; chưa có planning & optimization; Oracle RAC chia sẻ nhiều schema |
| **P3** | Thiếu monitoring real-time | **Cao** | Chỉ Mobile có Prometheus/Grafana; Core và WEB chưa có |
| **P4** | Incident response chậm | **Cao** | Phụ thuộc vendor; đang áp dụng Jira |
| **P5** | Log phân tán | Trung bình | Chưa có log center thống nhất giữa Core/WEB/Mobile |
| **P6** | Deploy thủ công Core + WEB | Trung bình | Đang triển khai GitLab |
| **P7** | Thiếu dashboard tổng quan | Trung bình | Không có cross-system view |

# II. Đề xuất triển khai


Để xử lý các pain point trên, Vero đề xuất dự án **Vero Monitor Service (VMS)** — triển khai theo dõi end-to-end 4 tầng (Infrastructure / Service / User / Business Flow), bổ sung Synthetic Monitoring cho các flow trading critical. Ở Basic Scope, output tập trung vào visibility vận hành: service/host có đang up, dependency có còn kết nối, synthetic API có pass và flow nào đang có điểm nghi vấn. Ở Advanced Scope, hệ thống bổ sung dữ liệu chuyên sâu như DBM/MQM/APM/RUM/Tracing để phân tích root cause và hỗ trợ vận hành.

Gồm **2 phương án triển khai** như sau:

:::compare
- title: Phương án 1 — Basic Scope
  badge: Triển khai trong 3 tháng
  items:
    - Infrastructure monitoring (host-level) cho 3 khối
    - Connection monitoring (service-to-service, open port)
    - Exchange connection monitoring (FIX session, MDDS, SFTP, bank/exchange)
    - Service health check (up/down, basic latency)
    - Dashboard monitoring & Alert (11 dashboard Basic)
    - Synthetic test API cho flow trading critical (tối đa 100 API)
    - Pre-market readiness check 8:30 hằng ngày
    - Knowledge handover cho đội vận hành MSVN
- title: Phương án 2 — Advanced Scope
  badge: Triển khai trong 9 tháng
  items:
    - Toàn bộ Basic Scope
    - Dashboard monitoring & Alert mở rộng (20 dashboard tổng: 11 Basic + 9 Advanced)
    - Database Monitoring (DBM) — DB2/Informix/MSSQL/Oracle RAC
    - Message Queue Monitoring (MQM) — ActiveMQ + Kafka
    - APM cho service trọng yếu (trade-api / authenservice / market-*)
    - Distributed Tracing end-to-end
    - Real User Monitoring (RUM) web & mobile
    - Synthetic test web (browser automation) cho flow trading critical
    - Runbook (Request → Approve → Bot Execute) + escalation matrix trên action/script được phê duyệt
    - Log center thống nhất (ELK / opensource log)
:::

:::phase{label="Phương án" n="1" title="Basic Scope — Monitor 4 tầng ở mức Basic"}
**Mục tiêu:** triển khai monitoring cho khối Core / WEB / Mobile/ Hệ thống vệ tinh theo 4 tầng, với mức cover của Basic như sau:
- **Infrastructure:** host-level monitoring (CPU, RAM, disk, network, host status, container restart).
- **Service:** connection monitoring + service health check (TCP/HTTP/process/basic latency), chưa đi vào DBM/MQM/APM/tracing chuyên sâu.
- **User:** Synthetic API + Pre-market readiness check lúc 8:30, chưa bao gồm RUM/Synthetic Web.
- **Business Flow:** dashboard flow/topology thể hiện luồng service hiện tại, giúp phát hiện nhanh điểm nghi vấn khi một flow nghiệp vụ phát sinh issue.

**Phạm vi theo tầng**

*Tầng 1 — Infrastructure (host-level only)*
- CPU, RAM, disk, network, host status, container restart cho 3 khối Core / WEB / Mobile.
- Triển khai **Vero Stack Monitor** (do Vero phát triển) + **bộ metric collector custom của Vero** cho cả 3 khối Core / WEB / Mobile.
- Chỉ theo dõi ở mức host-level; DBM/MQM chi tiết nằm trong Advanced Scope ở tầng Service.

*Tầng 2 — Service (Connection + Health, KHÔNG DBM/MQM/APM/Tracing)*
- **Connection monitoring (nội bộ):** service-to-service connection, service open port — theo dõi tình trạng kết nối giữa các service và trạng thái port lắng nghe.
- **Exchange connection monitoring:** FIX session (FixGW) — logon/logout, heartbeat, sequence gap, FIX MKT Event; MDDS — message receive/heartbeat/gap detection; SFTP connection; VSD/EIB/BIDV/TCB/VTB connection health.
- **Service health check:** up/down qua TCP port + process, basic latency qua standard probe (HTTP/TCP/heartbeat).
- KHÔNG bao gồm: DBM/MQM chi tiết, APM endpoint latency p50/p95/p99 và distributed tracing (các metric này nằm trong Advanced Scope).

*Tầng 3 — User Monitoring (chỉ Synthetic API, KHÔNG RUM)*
- **Synthetic test API** cho flow trading critical — **tối đa 100 API** (đặt lệnh, market data, account, OTP…). Chạy 24/7, alert khi fail.
- **Yêu cầu tiền đề:** MSVN cung cấp test account/test mode, dữ liệu test, whitelist/network access và cơ chế OTP handling phù hợp (test OTP / static OTP / mock OTP / bypass OTP trên môi trường test).
- **Pre-market readiness check 8:30** hằng ngày — verify FIX session, MDDS, status các Service đang monitor. Gửi report cho management lúc 8:30.
- KHÔNG bao gồm Real User Monitoring (RUM) trên web/mobile (metric này trong Advanced Scope).

*Tầng 4 — Business Flow (flow dashboard theo dữ liệu hiện có)*
- Business Flow trong Basic là các dashboard flow/topology thể hiện luồng service hiện tại để đội vận hành xác định nhanh điểm nghi vấn khi có issue. Ví dụ: đặt lệnh fail do service không kết nối được DB, dependency timeout, FIX session down hoặc network latency tăng.
- Ở mức Basic, Business Flow dựa trên service-to-service connection, port/health check và metric hạ tầng (CPU/RAM/disk/network). Vì vậy Basic phát hiện tốt các vấn đề như đứt kết nối, latency network tăng, host/service down hoặc tài nguyên tăng đột biến; chưa đủ để root cause sâu ở cấp transaction/code/query.
- Để Business Flow đạt hiệu quả cao nhất cần bổ sung APM, Distributed Tracing, DBM và MQM trong Advanced Scope, cùng instrumentation được MSVN phê duyệt nếu cần.

**Deliverable**
- Bộ **11 dashboard Basic** — chi tiết tại mục III & IV.
- Synthetic script API (tối đa 100 API) chạy 24/7 + pre-market readiness automation gửi báo cáo lúc 8:30.
- Tài liệu vận hành + workshop training cho đội vận hành MSVN để tự đọc dashboard, xử lý alert.

:::

:::phase{label="Phương án" n="2" title="Advanced Scope — Bổ sung DBM/MQM + Service APM/Tracing + RUM + Synthetic Web + Runbook/Log Center"}
**Mục tiêu:** ngoài toàn bộ output Basic Scope, phương án 2 bổ sung: tầng Service thêm DBM + MQM + APM + Distributed Tracing; tầng User thêm RUM + Synthetic Web; bổ sung Runbook + Log Center.

**Phạm vi (ngoài Basic)**

*Tầng 2 — Service (mở rộng: DBM + MQM + APM + Tracing)*
- **Database Monitoring (DBM)** cho toàn bộ DB của 3 khối:
  - **Core** (DB2 / Informix): connection pool, IO, lock, slow query, deadlock, replication lag (Active-Standby).
  - **WEB** (MS SQL Always On): connection pool, IO, slow query, deadlock, AG sync lag.
  - **Mobile** (Oracle RAC): connection pool, IO, slow query, deadlock, RAC instance status, ASM disk, schema-level usage.
  - Top-N slow query / schema, query plan regression alert.
- **Message Queue Monitoring (MQM)** cho ActiveMQ (Core/WEB) và Kafka (Mobile):
  - Broker status, cluster health, partition leader, ISR.
  - Queue / topic depth, consumer lag, throughput in/out, message rate.
  - Dead-letter queue size, retry, redelivery, failed message.
- **APM** cho service trọng yếu (trade-api / authenservice / market-*) — đo latency p50/p95/p99 theo endpoint, hot path, dependency, slow query trên các service MSVN cho phép instrument.
- **Distributed Tracing** end-to-end: client → trade-api → iFIS E/D → FixGW → HSX/HNX, áp dụng cho các đoạn flow đã có hoặc đã được vendor bổ sung correlation/trace ID.

*Tầng 3 — User (mở rộng: RUM + Synthetic Web)*
- **Real User Monitoring (RUM)** cho KeTrade web và Mobile — đo trải nghiệm user thật theo device, app version, network khi MSVN cho phép tích hợp SDK vào web/mobile.
- **Synthetic test web** 24/7 cho flow trading critical: login KeTrade, load bảng giá, đặt lệnh test, OTP, lookup tài khoản (bổ sung lớp web UI cho phần API đã cover ở Basic).

*Runbook & Log Center*
- **Runbook** thành workflow Request → Approve → Bot Execute (deploy / restart / rollback / run job) + escalation matrix nhiều cấp. Vero thiết kế workflow và tích hợp với API/script đã được MSVN phê duyệt; quyền production, script deploy/rollback/restart và quyết định thực thi thuộc MSVN.
- **Log center thống nhất** (ELK hoặc opensource log stack tương đương) để gom log, index, search và dashboard theo log format hiện có. Correlation ID / Trace ID chỉ được khai thác khi app/service đã output sẵn các field này.

**Deliverable (ngoài Basic)**
- Tổng cộng **20 dashboard**: 11 dashboard Basic + 9 dashboard Advanced.
- DBM dashboard + alert cho Core (DB2/Informix), WEB (MSSQL Always On), Mobile (Oracle RAC) + top-N slow query report.
- MQM dashboard + alert cho ActiveMQ và Kafka (broker, queue/topic depth, consumer lag, DLQ).
- Synthetic script web 24/7 cho flow trading critical.
- APM + RUM + Distributed Tracing hoạt động trên service đã được instrument hoặc được MSVN cho phép tích hợp agent/SDK.
- Runbook (Request → Approve → Bot Execute) + escalation matrix nhiều cấp, tích hợp trên **tối đa 20 action/script** đã được MSVN phê duyệt.
- Log center thống nhất + ingest/parsing/search dashboard theo log format hiện có của 3 khối (Core / WEB / Mobile).

:::

# III. Chi tiết theo dõi hệ thống Core/WEB/Mobile

Vero đề xuất mô hình monitoring 4 tầng để MSVN nhìn hệ thống theo luồng vận hành: hạ tầng có ổn không, service/dependency có kết nối không, synthetic/user-side check có pass không, và flow nghiệp vụ nào đang có điểm nghi vấn. Mức metric cụ thể phụ thuộc Basic hoặc Advanced Scope.

:::flow
- step: Infrastructure
  desc: Host / Network / Resource
- step: Service
  desc: Connection / Health / Dependency
- step: User
  desc: Synthetic / Pre-market / RUM
- step: Business Flow
  desc: Flow / Topology / Issue Point
:::

**Bảng 3.** Mô hình monitoring 4 tầng

| Tầng | Câu hỏi cần trả lời | Dữ liệu thu thập | Alert chính | Người dùng |
|---|---|---|---|---|
| **Infrastructure** | Tài nguyên có đang quá tải/suy giảm? | CPU, memory, disk, network, host status, container restart | Host down, disk full, network error, tài nguyên tăng bất thường | Infrastructure / Platform |
| **Service** | Service/dependency nào mất kết nối hoặc không phản hồi? | Service-to-service connection, open port, process status, basic HTTP/TCP latency | Service down, port closed, dependency timeout, latency network tăng | Application / Technical |
| **User** | Synthetic API/pre-market check có fail không? | Synthetic API status, login/API step status, pre-market readiness result | Synthetic API fail, pre-market check fail, traffic/API drop bất thường | Operation |
| **Business Flow** | Luồng service nào đang có điểm nghi vấn? | Flow/topology status, connection path, service health, infra signal theo flow | Flow path broken, dependency unreachable, service/host bất thường trong flow | Management / Business owner |

## Preview dashboard

Mock-up dashboard Vero Stack Monitor cho Maybank Securities — gồm **dashboard tổng quan** (Overall System), **3 dashboard chi tiết** theo từng khối, và **mô hình flow** topology giữa các thành phần được monitor. Bố cục và metric sẽ được chốt theo MSVN trong giai đoạn Design.

:::callout{kind="info" title="Metric limitation trong Basic Scope"}
Các dashboard preview có thể hiển thị một số metric chuyên sâu để minh hoạ khả năng mở rộng. Với Basic Scope, chỉ support metric vận hành cơ bản trong phạm vi đã chốt như host, service health, connection, synthetic API, pre-market readiness và flow/topology. Các metric DBM/MQM/APM/RUM/Tracing chỉ được triển khai khi MSVN chọn Advanced Scope hoặc khi hệ thống đã có instrumentation tương ứng.
:::

:::gallery{cols="1" title="Dashboard tổng quan"}
- src: img/01-overall-system.png
  alt: Overall System dashboard
  caption: "**Overall System** — view cross-system cho management & operation. Tổng hợp trạng thái 3 khối Core / WEB / Mobile, connection ra Sở/đối tác, synthetic API, pre-market readiness, incident active và flow/topology trọng yếu."
:::

:::callout{kind="note" title="Scope metric — Dashboard tổng quan"}
- **Metric trong mockup thuộc Advanced:** Error Rate, Req/s / Request Rate, high latency theo service, Queue drift / queue depth, Replica lag, transaction/order rate, bank Req/s và latency.
:::

:::gallery{cols="2" title="Dashboard chi tiết theo khối"}
- src: img/02-core-detail.png
  alt: Core systems dashboard
  caption: "**Core (Freewill)** — iFIS Equity/Derivative, SBA, DB2/Informix, ActiveMQ. Host/process status, connection health, FIX/bank dependency status, basic latency và business flow theo dữ liệu hiện có."
- src: img/03-web-detail.png
  alt: WEB systems dashboard
  caption: "**WEB (Innotech)** — KeTrade Trading. Availability, service health, basic HTTP/TCP latency, connection status tới dependency, MS SQL Always On/F5 ở mức connection/health."
- src: img/04-mobile-detail.png
  alt: Mobile systems dashboard
  caption: "**Mobile (Goline)** — Maybank Trade Mobile. Host/container status, service health, connection giữa microservice, Oracle/Kafka ở mức connection/health và synthetic API liên quan."
:::

:::callout{kind="note" title="Scope metric — Dashboard chi tiết theo khối"}
- **Core — metric trong mockup thuộc Advanced:** Transactions/min, DB2 bufferpool / lock / sessions / log space, Informix shared memory / checkpoint / Top SQL, ActiveMQ queue depth / enqueue / dequeue, replication lag chi tiết.
- **WEB — metric trong mockup thuộc Advanced:** Req/s, p95, error rate, subscription count, push msg/s, pricing pipeline throughput, ActiveMQ flow, MSSQL p95 / deadlock / buffer cache / top query.
- **Mobile — metric trong mockup thuộc Advanced:** Error rate 5xx, top error services, Kafka lag, Oracle RAC sessions / wait / IOPS, Kafka/Redis detail, WSS p99 / msg/sec, Keycloak token/fail metrics, WAF security rule metrics.
:::

:::gallery{cols="1" title="Mô hình flow — Overall"}
- src: img/00-flow-overall-system.png
  alt: Overall flow diagram
  caption: "**Flow Overall** — sơ đồ topology end-to-end: client web/mobile → API gateway → service core/WEB/Mobile → DB/MQ → connection ra Sở (HSX/HNX) + ngân hàng. Mỗi node hiển thị status/basic latency; mỗi edge hiển thị connection health."
:::

:::callout{kind="note" title="Scope metric — Flow Overall"}
- **Metric trong mockup thuộc Advanced:** sessions, Req/s, p95, error rate, queue depth, Kafka lag, Redis hit rate, DB sessions / IOPS, Msg/s, feed/s, order latency và business transaction count.
:::

:::gallery{cols="2" title="Mô hình flow — chi tiết theo khối"}
- src: img/00-flow-core.png
  alt: Core flow diagram
  caption: "**Flow Core** — iFIS Equity/Derivative, SBA, order routing, market data, integration hub; DB2 + Informix + ActiveMQ."
- src: img/00-flow-web.png
  alt: WEB flow diagram
  caption: "**Flow WEB** — KeTrade Trading: InnoBasTrade, InnoTradeAPI, PriceBoard, OrderEngine; F5 + MS SQL Always On + ActiveMQ."
- src: img/00-flow-mobile.png
  alt: Mobile flow diagram
  caption: "**Flow Mobile** — 50+ microservice trên Docker Swarm sau F5 + Traefik; Oracle RAC + Kafka + Redis Sentinel."
:::

:::callout{kind="note" title="Scope metric — Flow chi tiết theo khối"}
- **Flow Core — metric trong mockup thuộc Advanced:** feed msg/s, ord/s, sessions, req/s, ActiveMQ queue depth / enqueue / dequeue, DB IOPS / cache hit / shared memory, error rate.
- **Flow WEB — metric trong mockup thuộc Advanced:** sessions, req/s, p95, WSS clients, query/s, MSSQL lag / IOPS, ActiveMQ queue depth / msg/s và throughput theo pricing pipeline.
- **Flow Mobile — metric trong mockup thuộc Advanced:** sessions, req/s, p95, error rate, Kafka lag / msg/s, Redis hit rate, Oracle sessions/schema detail, Prometheus/Grafana/Jaeger/ELK metric chi tiết.
:::

:::gallery{cols="1" title="Dashboard KPI & SLA Scorecard"}
- src: img/14-kpi-sla.png
  alt: KPI and SLA Scorecard dashboard
  caption: "**KPI & SLA Scorecard** — view tổng hợp các chỉ số đo trực tiếp: Synthetic API success rate, Pre-market readiness compliance, Service coverage %, Open incidents, SLA breach theo ngày, availability theo service và Incident count theo khối Core/WEB/Mobile/Satellite."
:::

:::callout{kind="note" title="Scope metric — KPI & SLA Scorecard"}
- **Metric trong mockup thuộc Advanced:** runbook coverage, runbook execution coverage, APM p99 latency nếu đưa vào SLA per service, distributed trace coverage, RUM error rate và Synthetic Web success rate.
:::
# IV. Chi tiết theo dõi hệ thống vệ tinh

:::gallery{cols="1" title="Dashboard Exchange Gateway (HSX & HNX)"}
- src: img/05-gateway-hsx-hnx.png
  alt: Exchange Gateway HSX HNX dashboard
  caption: "**Exchange Gateway — HSX & HNX** — monitor 2 cột HSX/HNX với process status (**CPU / Memory / Disk** / last restart), FIX Session (FixGW), FIX MKT Event, MDDS total message receive/heartbeat/gap detection, SFTP connection và connection status."
:::

:::box{title="Connection / Session monitoring tới HSX & HNX"}
- **FIX Session (FixGW)** — theo dõi logon/logout, heartbeat (35=0), sequence gap, resend request, disconnect bất thường. Alert khi session down trong giờ giao dịch.
- **FIX MKT Event** — theo dõi market/trading phase event từ kênh FIX, last event time và event status. Alert khi không nhận event hoặc event bất thường.
- **MDDS (FIX multicast)** — total message receive, heartbeat, gap detection theo MsgSeqNum. Alert khi mất heartbeat hoặc gap bất thường.
- **SFTP connection** — port reachable, connectivity check và last activity nếu có. Alert khi mất kết nối hoặc không truy cập được.
:::

:::gallery{cols="1" title="Dashboard hệ thống vệ tinh"}
- src: img/06-satellite-systems.png
  alt: Satellite systems dashboard
  caption: "**Satellite Systems Monitoring** — grid card cho VSD (lưu ký), EIB/BIDV/TCB/VTB (bank connection), SFTP, Vietstock, TradingView, Fidessa, ECD, Smart OTP, SMS/Email gateway. Mỗi card hiển thị process status, **CPU / Memory / Disk** mini-bar, port reachable, **network in/out throughput** và last activity."
:::

# V. Chi tiết Advanced Scope

Phần này diễn giải chi tiết 8 nhóm hạng mục thuộc **Phương án 2 — Advanced Scope** (ngoài Basic), kèm preview tham khảo.

## 1. Database Monitoring (DBM)

**Đối tượng:** DB2 + Informix (Core / Freewill), MS SQL Always On (WEB / Innotech), Oracle RAC (Mobile / Goline).

**Metric thu thập:**
- Connection pool usage, IO throughput, lock wait time, deadlock count
- Slow query (top-N theo schema), query plan regression
- Replication / AG sync lag (DB2 Active-Standby, MSSQL Always On, Oracle RAC instance status)
- Storage trend, ASM disk usage (Oracle), buffer cache hit ratio

**Mục đích:** cung cấp dữ liệu để xác định bottleneck DB; là input cho việc tuning query/index và hỗ trợ xử lý sự cố liên quan database.

:::gallery{cols="1" title="Preview — DBM Dashboard"}
- src: img/07-dbm.png
  alt: Database Monitoring dashboard
  caption: "Dashboard DBM tổng hợp 4 hệ quản trị (DB2/Informix/MSSQL/Oracle RAC) — connection pool, IO, slow query, deadlock, replication lag."
:::

## 2. Message Queue Monitoring (MQM)

**Đối tượng:** ActiveMQ (Core / WEB), Kafka (Mobile).

**Metric thu thập:**
- Broker status, cluster health, partition leader, ISR (Kafka)
- Queue / topic depth, consumer lag, throughput in/out, message rate
- Dead-letter queue size, retry count, redelivery, failed message
- Producer/consumer latency

**Mục đích:** theo dõi queue backlog và consumer lag; cung cấp dữ liệu để đo SLA cho luồng đặt lệnh, OTP, notification.

:::gallery{cols="1" title="Preview — MQM Dashboard"}
- src: img/08-mqm.png
  alt: Message Queue Monitoring dashboard
  caption: "Dashboard MQM cho ActiveMQ + Kafka — broker status, queue/topic depth, consumer lag, DLQ, throughput in/out."
:::

## 3. APM cho service trọng yếu

**Đối tượng:** trade-api, authenservice, market-* (Mobile); có thể mở rộng sang iFIS Equity/Derivative khi vendor instrument.

**Metric thu thập:**
- Latency p50 / p95 / p99 theo endpoint, hot path
- Dependency map service-to-service, slow downstream call
- Error trace, exception stack trace, request log link
- Throughput, error rate, Apdex score

**Mục đích:** xác định endpoint có latency hoặc error rate vượt ngưỡng; cung cấp call graph để xác định nguyên nhân; là input cho MSVN và các bên kỹ thuật liên quan tuning code, xử lý bottleneck application.

:::gallery{cols="1" title="Preview — APM Dashboard"}
- src: img/09-apm.png
  alt: APM dashboard
  caption: "APM cho service trọng yếu — latency p50/p95/p99 theo endpoint, hot path, dependency, slow query, error rate."
:::

## 4. Distributed Tracing end-to-end

**Phạm vi:** trace 1 request đi xuyên các service: client → API gateway → trade-api → authenservice → iFIS Equity/Derivative → FixGW → HSX/HNX → callback, trong phạm vi các service đã có hoặc được vendor bổ sung correlation/trace ID.

**Cách triển khai:**
- Sử dụng **Vero Tracing** (do Vero phát triển) — hỗ trợ Correlation ID + Trace ID xuyên service theo chuẩn W3C TraceContext.
- Vero cung cấp SDK/hướng dẫn integration và cấu hình tracing backend. Việc chỉnh sửa application code để phát sinh/propagate correlation ID, trace ID do MSVN thực hiện hoặc điều phối vendor hệ thống thực hiện.

**Mục đích:** truy vết latency và lỗi xuyên suốt các service không cần đọc log thủ công; cung cấp span breakdown theo từng service để xác định bottleneck trong chuỗi gọi.

:::gallery{cols="1" title="Preview — Distributed Tracing"}
- src: img/10-tracing.png
  alt: Distributed Tracing visualization
  caption: "Waterfall trace cho 1 giao dịch đặt lệnh — span breakdown qua từng service, hiển thị bottleneck span và slow downstream call."
:::

## 5. Real User Monitoring (RUM) web & mobile

**Phạm vi:** đo trải nghiệm user thật trên KeTrade Web và Maybank Trade Mobile.

**Metric thu thập:**
- Page load time, Largest Contentful Paint (LCP), First Input Delay (FID)
- JavaScript error, crash rate (mobile), AJAX/API call latency
- Phân tích theo device, OS, app version, network type, vùng địa lý
- User journey (session replay nếu cần)

**Mục đích:** xác định device/version/vùng có latency hoặc error rate cao; cung cấp dữ liệu để so sánh before/after release; phát hiện regression sau khi rollout.

:::gallery{cols="1" title="Preview — RUM Dashboard"}
- src: img/12-rum.png
  alt: Real User Monitoring dashboard
  caption: "RUM dashboard — page load p95 theo device/version/OS/network, crash rate Mobile, error JS theo trang, breakdown theo địa lý."
:::

:::gallery{cols="1" title="Preview — RUM Detail"}
- src: img/17-rum-detail.png
  alt: Real User Monitoring user session detail dashboard
  caption: "RUM Detail — drill-down 1 user/session để support kiểm tra lỗi từ góc nhìn user: user context, current issue, session replay timeline, Web Vitals, frontend error, Browser API timing, resource timing, console/device context, support actions và RUM identifiers."
:::

## 6. Synthetic Web Monitoring

**Phạm vi:** Synthetic test 24/7 cho flow trading critical trên KeTrade Web — login, load bảng giá, đặt lệnh test, OTP, lookup tài khoản.

**Cách triển khai:**
- Sử dụng **browser automation** (Playwright / Selenium) để simulate user thật trên UI thực tế — click, fill form, navigate, verify cả response HTTP lẫn DOM state (button enabled, table có data, modal đóng).
- Script chạy 24/7 theo lịch định kỳ; có thể chạy multi-location để kiểm tra geo latency.
- Tích hợp vào pre-market readiness 8:30 và post-deploy verification.
- **Yêu cầu tiền đề:** MSVN cung cấp test account/test mode, dữ liệu test, whitelist/network access và cơ chế OTP handling phù hợp cho flow web.

**Pain point cover được (so với Synthetic API):**
- **API change / breaking change** — vendor đổi API contract mà UI chưa update, synthetic API có thể vẫn pass nhưng synthetic web sẽ fail ở DOM verify.
- **Sequence bug / logic flow** — bắt được lỗi luồng đa bước (login → vào bảng giá → đặt lệnh → confirm), trong khi synthetic API chỉ test từng endpoint độc lập.
- **Frontend regression** — JS error, render lỗi, state UI sai sau release web.

**Mục đích:** lớp test cuối cùng từ góc nhìn user thực tế; bắt được class lỗi mà synthetic API + RUM bỏ sót; là gate cho pre-market readiness và post-deploy verification.

## 7. Runbook (Request → Approve → Bot Execute)

**Cách tiếp cận:** runbook đóng gói thành workflow với 3 bước: **Request → Approve → Bot Execute**. Người vận hành submit yêu cầu, người có thẩm quyền review và approve, bot thực thi action đã được MSVN cho phép sau khi được approve.

**3 bước:**
- 1. **Request** — submit yêu cầu (deploy version mới, restart, rollback, run job…). Mỗi request có context: service, scope, lý do, link APM/log.
- 2. **Approve** — approver (system owner / manager / on-call lead) review request, đối chiếu policy, approve hoặc reject. Quyết định được ghi vào audit trail.
- 3. **Bot Execute** — runbook bot tích hợp với tool/API/script đã được MSVN phê duyệt để thực thi action trong scope. Bot ghi log realtime; nếu health check fail, bot kích hoạt bước rollback/restore khi MSVN đã cung cấp script tương ứng.

**Phạm vi áp dụng:**
- **Deploy** — hỗ trợ workflow approve/execute cho pipeline deploy đã được MSVN cung cấp hoặc phê duyệt.
- **Quản trị** — restart service, clear cache, rotate certificate, run batch, rollback version nếu MSVN cung cấp quyền và script/API vận hành tương ứng.
- **Giới hạn scope:** tối đa **20 runbook actions** trong Advanced Scope. Action ngoài danh sách đã chốt sẽ đưa vào backlog hoặc change request.

**Mục đích:** chuẩn hoá các action production qua Request → Approve → Bot Execute thay vì SSH thủ công khi có điều kiện tích hợp; mọi action có audit trail (ai request, ai approve, bot làm gì, kết quả); hỗ trợ rollback khi health check fail nếu script/API rollback đã được MSVN cung cấp.

:::gallery{cols="1" title="Preview — Runbook Workflow"}
- src: img/13-runbook.png
  alt: Runbook workflow Request Approve Bot Execute
  caption: "Workflow tự động 3 bước: **Request → Approve → Bot Execute** cho deploy/quản trị trên action/script được phê duyệt. Mọi action có audit trail; rollback tự động chỉ áp dụng khi MSVN cung cấp script/API tương ứng."
:::

# VI. Tool & technology stack

Vero triển khai **Vero Stack Monitor** + **Vero metric collector** làm lớp theo dõi chung cho 3 khối Core / WEB / Mobile. Trường hợp MSVN chọn phương án thương mại, Vero thiết kế và cấu hình theo yêu cầu.

**Bảng 4.** Tool stack đề xuất theo từng tầng

| Hạng mục | Tool đề xuất (do Vero phát triển / tích hợp) | Phương án thương mại |
|---|---|---|
| Stack monitor platform | **Vero Stack Monitor** — dashboard, alert, synthetic, business flow tích hợp cho hệ thống chứng khoán | — |
| Metric collection | **Vero metric collector** (do Vero phát triển) | Datadog, New Relic, Dynatrace |
| Visualization | **Vero Dashboard** (do Vero phát triển) cho 3 khối Core / WEB / Mobile | Datadog Dashboard, Grafana Cloud |
| Alerting | **Vero Alert** (do Vero phát triển) — severity, routing, escalation theo owner | Datadog Monitor, PagerDuty |
| Synthetic monitoring | **Vero Synthetic Engine** (do Vero phát triển) | Datadog Synthetics, Checkly |
| Log center | ELK (Filebeat → Logstash/Elastic → Kibana) | Splunk, Datadog Logs |
| Distributed tracing | **Vero Tracing** (do Vero phát triển) | Datadog APM, Honeycomb |
| APM (Advanced) | **Vero APM** (do Vero phát triển) | Datadog APM, New Relic, Dynatrace |
| RUM (Advanced) | **Vero RUM** (do Vero phát triển) | Datadog RUM, New Relic Browser |
| Infrastructure | **Kubernetes (K8s)** — orchestrate Vero Stack Monitor & các component liên quan | OpenShift, EKS, GKE |
| CI/CD | **GitLab CI + ArgoCD** — CI/CD cho các component thuộc Vero Monitor System (Vero Stack Monitor, collector, dashboard, alert/synthetic engine và các component Advanced theo scope) | GitLab Premium, GitHub Actions, Spinnaker |
| Runbook (Advanced) | **Vero Runbook Engine** — runbook engine do Vero phát triển, đóng gói workflow Request → Approve → Bot Execute và tích hợp với script/API đã được MSVN phê duyệt | Rundeck, StackStorm, PagerDuty Automation |

:::callout{kind="note"}
**Lưu ý license:** phần lớn stack đề xuất là open-source nên Vero implementation fee không bao gồm license. Nếu MSVN chọn phương án thương mại (Datadog, New Relic, Dynatrace…), chi phí license do MSVN chi trả trực tiếp với vendor; Vero sẽ hỗ trợ thiết kế và cấu hình.
:::

# VII. Kết quả kỳ vọng

Phần này mô tả kết quả kỳ vọng Vero deliver để xử lý các pain point hiện tại của MSVN và giảm rủi ro vận hành có thể phát sinh trong tương lai.

## Pain Point → Solution → Cách giải quyết

Mỗi pain point được map với solution Vero deliver và cách solution đó giúp MSVN phát hiện, khoanh vùng hoặc xử lý issue nhanh hơn.

**Bảng 5.** Mapping Pain Point sang solution Vero deliver

| Pain Point | Solution Vero deliver | Cách giải quyết pain point | Scope |
|---|---|---|---|
| **Downtime giao dịch** (Cao) | Monitoring 4 tầng, alert owner/routing, Synthetic API, Pre-market readiness check; Advanced bổ sung Runbook. | Phát hiện sớm host/service/dependency bất thường, kiểm tra readiness trước giờ giao dịch, route alert về đúng owner và chuẩn hoá thao tác xử lý qua Runbook khi đã được MSVN phê duyệt. | Basic + Advanced |
| **Database bottleneck** (Cao) | DBM dashboard + alert cho DB2/Informix/MSSQL/Oracle RAC. | Làm rõ dấu hiệu bottleneck DB như slow query, lock/deadlock, connection pool, IO và replication/AG sync lag để MSVN, DB team và các bên kỹ thuật liên quan có dữ liệu xử lý đúng điểm nghẽn. | Advanced |
| **Thiếu monitoring real-time** (Cao) | Dashboard 4 tầng, health/connection check, Synthetic API; Advanced bổ sung MQM/APM/RUM/Tracing. | Thay việc kiểm tra thủ công bằng dashboard và alert real-time, giúp đội vận hành nhìn được trạng thái host, service, dependency, user-side check và flow nghiệp vụ trên cùng một hệ thống. | Basic + Advanced |
| **Log phân tán** (Trung bình) | Log Center + ingest/parsing/search dashboard theo log format hiện có. | Gom log về một điểm tra cứu tập trung để giảm thời gian tìm log giữa Core/WEB/Mobile; correlation theo trace ID chỉ mở rộng khi app/service đã output field tương ứng. | Advanced |
| **Deploy thủ công** (Trung bình) | Pre-market check, post-change health check; Advanced bổ sung Runbook cho action/script được phê duyệt. | Basic giúp kiểm tra trạng thái sau thay đổi; Advanced chuẩn hoá các thao tác deploy/restart/rollback/run job qua Request → Approve → Bot Execute, có audit trail thay vì phụ thuộc SSH thủ công. | Basic + Advanced |
| **Incident response chậm** (Cao) | Alert catalog, severity/routing, tài liệu xử lý alert cơ bản; Advanced bổ sung Runbook + escalation matrix + RACI. | Khi có incident, đội vận hành biết alert thuộc layer/flow nào, owner nào xử lý, bước đầu cần làm gì và khi nào escalate, giảm thời gian triage ban đầu. | Basic + Advanced |
| **Thiếu dashboard tổng quan** (Trung bình) | 11 dashboard Basic; Advanced mở rộng thành 20 dashboard tổng. | Cung cấp view cross-system cho management/operation/technical, kèm flow topology để nhanh chóng xác định service/dependency nào đang ảnh hưởng một flow nghiệp vụ. | Basic + Advanced |

## KPI dự án

Để đánh giá mức độ hoàn thành và hiệu quả sau triển khai, dưới đây là danh sách KPI dùng làm cơ sở nghiệm thu và theo dõi kết quả dự án.

:::stats
- label: Synthetic API
  value: ≥ 99%
  hint: Success rate trên flow trading critical trong scope
- label: Pre-market check
  value: > 99%
  hint: % ngày giao dịch có report đúng giờ 8:30
- label: Service coverage
  value: 100%
  hint: % service trong scope có metric + alert
- label: Handover
  value: ≥ 3 người
  hint: MSVN pass workshop assessment, xử lý alert độc lập
:::

**Bảng 6.** KPI đo hiệu quả dự án (Basic vs Advanced)

| Hạng mục | Gói | Kết quả đo lường | KPI theo dõi |
|---|:---:|---|---|
| **Phát hiện issue qua synthetic + alert** | Basic | Issue phát hiện qua synthetic API + alert trước khi user mở ticket | Synthetic API success rate, Pre-market readiness compliance, % alert Critical/High có owner + routing |
| **Coverage cơ bản (metric + alert)** | Basic | % service trong scope có metric + alert | % service được monitor, số dashboard active, weekly usage by role |
| **Knowledge transfer** | Basic + Advanced | Nhân sự MSVN pass workshop và xử lý alert độc lập | Số người pass workshop, onboarding time, % ticket không cần escalate Vero/vendor |
| **Phát hiện issue mở rộng (UI + APM + RUM)** | **Advanced only** | Bắt thêm class lỗi UI + frontend + cross-service | Synthetic Web success rate, RUM error rate, APM latency p99, distributed trace coverage |
| **Coverage đầy đủ (log + trace + runbook)** | **Advanced only** | Mở rộng coverage sang log centralize + tracing + Runbook | % service đẩy log về center, % service có correlation/trace ID nếu vendor đã output field, số runbook đã đóng gói |
| **Runbook** | **Advanced only** | % action production được MSVN phê duyệt đi qua workflow Request → Approve → Bot Execute thay vì SSH thủ công | % deploy/restart/rollback qua bot; audit trail compliance; mean time to deploy |

# VIII. Tiến độ & nhân sự

**Bảng 7.** Roadmap triển khai

| Giai đoạn | Basic | Advanced | Trọng tâm | Output |
|---|:---:|:---:|---|---|
| 1. Discovery & Assessment | 2 tuần | 3 tuần | Thu thập thông tin, phân loại, chọn pilot | Assessment report, inventory, dependency map |
| 2. Solution Design | 1 tuần | 3 tuần | Dashboard, monitoring, alert | Solution design, wireframe, metric/alert catalog |
| 3. Pilot Implementation | 3 tuần | 6 tuần | Triển khai trên 1 nhóm hệ thống pilot | Pilot dashboard, alert active, tài liệu xử lý alert draft |
| 4. Rollout | 4 tuần | 12 tuần | Mở rộng diện rộng, tuning, handover | Production dashboard, alert tuned, training |
| 5. Stabilize & Improve | 2 tuần | 8 tuần | Tuning, continuous improvement | Improvement backlog, monthly review note |
| 6. Handover & Close | 1 tuần | 4 tuần | Bàn giao đầy đủ tài liệu, workshop | Handover package, sign-off |
| **Tổng thời lượng** | **13 tuần (~3 tháng)** | **36 tuần (~9 tháng)** | | |

:::roadmap
- name: Basic Scope
  duration: 3 tháng
  phases:
    - Discovery | 2
    - Design | 1
    - Pilot | 3
    - Rollout | 4
    - Stabilize | 2
    - Handover | 1
- name: Advanced Scope
  duration: 9 tháng
  phases:
    - Discovery | 3
    - Design | 3
    - Pilot | 6
    - Rollout | 12
    - Stabilize | 8
    - Handover | 4
:::

**Bảng 8.** Nhân sự đề xuất

| Vai trò | Đơn vị | Số người | Trách nhiệm chính |
|---|:---:|:---:|---|
| Project Sponsor | **MSVN** | 1 | Phê duyệt scope, milestone, ngân sách; quyết định khi có blocker chiến lược |
| System Owner (Core / WEB / Mobile) | **MSVN** | 3 | Đại diện business + technical cho từng khối, phê duyệt design dashboard / alert |
| Operation Lead | **MSVN** | 1 | Nghiệm thu tài liệu xử lý alert/runbook theo scope, training đội vận hành, vận hành sau handover |
| Project Manager / Delivery Lead | **Vero** | 0.3 | Plan, scope, risk, communication, vendor coordination |
| Monitoring Architect | **Vero** | 0.3 | 4-tầng model, tool stack, alert design, integration |
| Senior DevOps / SRE | **Vero** | 1–2 | Deploy Vero Stack Monitor + metric collector, dashboard build, ELK log center + tracing tuning |
| Test Engineer (Synthetic) | **Vero** | 0.5 | Synthetic script API/web, regression automation |
| Business Analyst | **Vero** | 0.2 | Pain point mapping, KPI catalog, workshop |
| Tech Writer / Trainer | **Vero** | 0.2 | Tài liệu xử lý alert/runbook theo scope, training material, handover workshop |

:::callout{kind="note" title="Ghi chú nhân sự Vero"}
Tổng nhân sự phía Vero dự kiến là **2–3 senior engineer**. Các vai trò 0.x trong bảng là phân bổ trách nhiệm theo function, không phải headcount tách riêng toàn thời gian.
:::

# IX. Thanh toán

**Bảng 9.** Payment milestones

| Milestone | % Basic | % Advanced | Điều kiện nghiệm thu |
|---|:---:|:---:|---|
| Kickoff | 0% | 0% | Ký hợp đồng + kickoff meeting |
| Hoàn thành Assessment + Design | 0% | 0% | Bàn giao System Inventory, Dependency Map, Solution Design |
| Pilot go-live | 0% | 0% | Pilot dashboard + alert active trên 1 nhóm hệ thống |
| Rollout production | 90% | 90% | Production dashboard + alert tuned + tài liệu xử lý alert/runbook theo scope |
| Handover & Close | 10% | 10% | Workshop + handover checklist + sign-off |

:::callout{kind="warn" title="Phạm vi không bao gồm"}
- Chi phí license tool thương mại (Datadog, New Relic, Dynatrace…) nếu MSVN chọn.
- Hardware / server / storage.
- Vendor customization fee (Freewill / Innotech / Goline) — phần app/service do từng vendor báo giá riêng cho scope Advanced.
:::

:::box{title="Điều khoản chung"}
- **Hiệu lực đề xuất:** 30 ngày kể từ ngày đề xuất.
- **Thanh toán:** chuyển khoản theo milestone, NET 15 ngày kể từ ngày nhận invoice.
- **Bảo mật:** Vero ký NDA với MSVN; toàn bộ dữ liệu khảo sát/log không lưu trữ ngoài môi trường MSVN.
- **Sở hữu trí tuệ:** dashboard, runbook, tài liệu handover thuộc sở hữu Vero; MSVN được cấp quyền sử dụng nội bộ sau nghiệm thu, không chuyển nhượng cho bên thứ ba.
:::

:::signature
- party: Đại diện Vero Technology  
  title: TỔNG GIÁM ĐỐC
  name: BÙI LÊ NỮ PHƯỢNG TIÊN
:::

# X. Phụ lục — Project Risk Analysis

*Phần tham khảo thêm cho MSVN, không nằm trong scope nghiệm thu.*

## Vendor coordination & RACI

MSVN có 3 vendor (Freewill, Innotech, Goline) với release cycle khác nhau. Để tránh chậm tiến độ pilot và rollout, phần phối hợp giữa các bên được thiết kế như sau:

**Bảng A1.** Phân vai RACI giữa các bên

| Hoạt động | Maybank | Vero | Freewill (Core) | Innotech (WEB) | Goline (Mobile) |
|---|:---:|:---:|:---:|:---:|:---:|
| System inventory + dependency map | A | R | C | C | C |
| Dashboard design + setup | A | R | I | I | C |
| Monitoring agent setup | A | R | C | C | C |
| Instrumentation app (correlation/trace ID) | I | C | **R** | **R** | **R** |
| Alert rule + runbook | A | R | C | C | C |
| Synthetic script (API/web) | I | R | – | C | C |
| Incident response process | A | R | C | C | C |
| Knowledge handover | A | R | I | I | I |

:::callout{kind="info"}
**R** = Responsible (người làm) · **A** = Accountable (chịu trách nhiệm cuối) · **C** = Consulted (cần hỏi ý kiến) · **I** = Informed (cần được thông báo).
:::

## Rủi ro & giảm thiểu

Dự án monitoring/observability có nhiều lợi ích về theo dõi, vận hành và mở rộng — nhưng cần quản trị rủi ro ngay từ giai đoạn assessment. Rủi ro chính tập trung vào dữ liệu hiện trạng, mức độ sẵn sàng application/service, chất lượng alert, bảo mật dữ liệu và khả năng thay đổi quy trình vận hành.

**Bảng A2.** Rủi ro chính và phương án giảm thiểu

| Rủi ro | Tác động | Phương án giảm thiểu | Owner |
|---|---|---|---|
| Thiếu dữ liệu hiện trạng | Khó xác định scope ưu tiên, dashboard thiếu dữ liệu | Assessment + workshop với system owner; checklist | MSVN + Vero |
| Chất lượng log/metric chưa đủ | Khó truy vết root cause, alert không chính xác | Vero chuẩn hoá metric catalog/tagging ở lớp monitoring; việc chuẩn hoá log output và instrumentation trong app do MSVN thực hiện hoặc điều phối vendor hệ thống thực hiện | Application / Vendor |
| Alert noise | Đội vận hành bỏ qua cảnh báo | Phân loại severity, tuning threshold sau pilot, gắn tài liệu xử lý alert/runbook theo scope | Vero + MSVN |
| Phụ thuộc vendor cho instrumentation | Chậm tiến độ rollout | Lock scope per vendor sớm, RACI rõ ràng | Maybank PMO |
| Bảo mật dữ liệu monitoring | Lộ thông tin nội bộ qua dashboard/log | RBAC + data masking + audit log | Security team |
