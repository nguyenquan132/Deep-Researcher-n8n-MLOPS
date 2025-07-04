# Deep-Researcher-n8n-MLOPS

Bộ **workflow n8n** kết hợp **stack giám sát MLOps** trọn gói giúp bạn:

* Tự động hóa *Deep Research* (thu thập, tổng hợp và tạo báo cáo) bằng LLM.
* Giám sát hạ tầng (Prometheus + Grafana + Alertmanager) và gửi cảnh báo tới n8n để định dạng email thân thiện, kèm đề xuất khắc phục.

---

## Mục lục

1. [Kiến trúc tổng quan](#kien-truc-tong-quan)
2. [Thành phần & tính năng](#thanh-phan-va-tinh-nang)
3. [Yêu cầu hệ thống](#yeu-cau-he-thong)
4. [Cài đặt & khởi chạy](#cai-dat-khoi-chay)
5. [Cấu hình n8n](#cau-hinh-n8n)
6. [Triển khai giám sát](#trien-khai-giam-sat)
7. [Sử dụng Deep‑Researcher](#su-dung-deep-researcher)

---

## Kiến trúc tổng quan

```text
┌──────────────┐          ┌────────────────┐       ┌──────────────┐
│   Người dùng │──HTTP──▶│ n8n DeepResearch │───▶ │  Notion DB   │
└──────────────┘          └────────────────┘       └──────────────┘
        ▲                         │
        │                         ▼
        │                  ┌──────────────┐
        │  Cảnh báo (JSON) │ AlertWorkflow│─SMTP─▶Email/Gmail
        │                  └──────────────┘
        │                         ▲
        │                         │ Webhook `/webhook/alertmanager`
┌──────────────┐  metrics  ┌──────────────┐ ────────┤
│  Node Export │──────────▶│ Prometheus   │──Alerts─┘
└──────────────┘           └──────────────┘
                                │
                                ▼
                           Grafana UI
```

* **Deep Researcher** – Workflow `workflow n8n/Deep_Researcher_workflow.json` thu thập thông tin, sinh truy vấn SERP, tổng hợp *learnings* và viết báo cáo Notion (mặc định GPT‑4o‑mini).
* **Alert Prometheus** – Workflow `workflow n8n/Alert_Prometheus_workflow.json` nhận webhook Alertmanager, dùng LLM để phân tích & gửi email cảnh báo.
* **monitor/** – Stack Docker‑Compose khởi tạo Prometheus, Node‑Exporter, Alertmanager & Grafana kèm rule và dashboard mẫu.

---

## Thành phần & tính năng

| Thành phần                     | Mô tả                                                                                                                                                                                           |
| ------------------------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **n8n**                        | Nền tảng workflow no‑/low‑code. **Repo không kèm docker‑compose cho n8n** – bạn tự cài theo [hướng dẫn chính thức](https://docs.n8n.io/hosting/) hoặc dùng n8n Cloud, sau đó *import* workflow. |
| **Deep\_Researcher workflow**  | Tự động hoá quy trình nghiên cứu chuyên sâu, lưu báo cáo vào Notion.                                                                                                                            |
| **Alert\_Prometheus workflow** | Phân loại mức độ cảnh báo, sinh email chi tiết kèm hướng dẫn khắc phục.                                                                                                                         |
| **Prometheus + Node Exporter** | Thu thập số liệu hệ thống & ứng dụng.                                                                                                                                                           |
| **Alertmanager**               | Xử lý rule, đẩy cảnh báo tới n8n qua webhook `/webhook/alertmanager`.                                                                                                                           |
| **Grafana**                    | Dashboard realtime, đã kèm `monitor/grafana.json`.                                                                                                                                              |

---

## Yêu cầu hệ thống

* **Docker 20+** và **Docker Compose v2**.
* Máy chủ Linux (RAM ≥ 2 GB, CPU ≥ 2 core).
* Mở port: `5678` (n8n), `3000` (Grafana), `9090` (Prometheus), `9093` (Alertmanager), `9100` (Node Exporter).
* Tài khoản **OpenAI** (hoặc LLM tương đương).
* Tài khoản Gmail đã cấu hình **OAuth2** (cho node Gmail).

---

## Cài đặt & khởi chạy

```bash
# 1. Clone dự án
git clone https://github.com/nguyenquan132/Deep-Researcher-n8n-MLOPS.git
cd Deep-Researcher-n8n-MLOPS

# 2. Tạo file .env rồi bổ sung khoá API & OAuth
autocomplete your own
cat <<'EOF' > .env
# OpenAI
API_KEY_CHAT_MODEL=sk-...
# Gmail OAuth2 (ID credential n8n)
OAUTH2_GMAIL=cred_...
# Host dùng cho Alertmanager → n8n
ALERT_WEBHOOK_HOST=http://my-n8n-host
EOF

# 3. Khởi chạy stack giám sát
cd monitor
docker compose up -d
```

> **Lưu ý:** n8n **không** nằm trong compose trên. Hãy cài & khởi chạy n8n riêng (Docker image `n8nio/n8n`, n8n Cloud, hoặc cách bất kỳ).

---

## Cấu hình n8n

1. Truy cập UI n8n (`http://<server>:5678/`) và tạo **Credentials**:

   * **OpenAI API** → dán `API_KEY_CHAT_MODEL`.
   * **Gmail OAuth2** → dán `OAUTH2_GMAIL` và hoàn tất OAuth flow.
2. **Import workflow**

   * `workflow n8n/Deep_Researcher_workflow.json` – Form endpoint `/form/deep_research`.
   * `workflow n8n/Alert_Prometheus_workflow.json` – Webhook `/webhook/alertmanager`.
3. Bấm **Activate** cho từng workflow.

---

## Triển khai giám sát

Prometheus, Alertmanager, Node‑Exporter và Grafana đã chạy bên trong `monitor/docker-compose.yml`.

* **Prometheus UI:** `http://<server>:9090`
* **Grafana UI:** `http://<server>:3000` (mặc định `admin`/`admin`).
* **Node‑Exporter:** `http://<server>:9100/metrics`

Rule cảnh báo mẫu trong `prometheus/rules.yml` sẽ kích hoạt Alertmanager → n8n.

---

## Sử dụng Deep‑Researcher

1. Mở form `http://<server>:5678/form/deep_research`.
2. Nhập **Query**, **Depth** (độ sâu) & **Breadth** (độ rộng).
3. Xem kết quả tại Notion database đã cấu hình.

---


> © 2025 – Made with ❤️ by US
