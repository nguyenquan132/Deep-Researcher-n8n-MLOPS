# Deep‑Researcher‑n8n‑MLOPS

Bộ **workflow n8n** và **stack giám sát MLOps** trọn gói giúp bạn:

* Tự động hóa *Deep Research* (thu thập, tổng hợp và tạo báo cáo) bằng LLM.
* Giám sát hạ tầng (Prometheus + Grafana + Alertmanager) và gửi cảnh báo tới n8n để định dạng email thân thiện, kèm đề xuất khắc phục.

> Dự án này được xây dựng hoàn toàn bằng mã nguồn mở, triển khai bằng Docker, dễ dàng tự host ở bất kỳ đâu.

---

## Mục lục

1. [Kiến trúc tổng quan](#kien-truc-tong-quan)
2. [Thành phần & tính năng](#thanh-phan-va-tinh-nang)
3. [Yêu cầu hệ thống](#yeu-cau-he-thong)
4. [Cài đặt](#cai-dat)
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
        │  Cảnh báo (JSON) │ AlertWorkflow│─SMTP─▶Gmail / Email
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

* **Deep Researcher** – Workflow `workflow n8n/Deep_Researcher_workflow.json` thu thập thông tin, lập kế hoạch truy vấn SERP, tổng hợp *learnings* và viết báo cáo Notion qua LLM (mặc định GPT‑4o‑mini) ([raw.githubusercontent.com](https://raw.githubusercontent.com/nguyenquan132/Deep-Researcher-n8n-MLOPS/38286f797e7efaaaf11ae33c166cfde33308d067/workflow%20n8n/Deep_Researcher_workflow.json)).
* **Alert Prometheus** – Workflow `workflow n8n/Alert_Prometheus_workflow.json` nhận webhook Alertmanager, dùng LLM để phân tích, tạo tiêu đề & nội dung email, và gửi qua Gmail OAuth ([github.com](https://github.com/nguyenquan132/Deep-Researcher-n8n-MLOPS/commit/38286f797e7efaaaf11ae33c166cfde33308d067)).
* **monitor/** – Docker‑Compose khởi tạo Prometheus, Node‑Exporter, Alertmanager & Grafana kèm cấu hình sẵn rule & dashboard ([github.com](https://github.com/nguyenquan132/Deep-Researcher-n8n-MLOPS/commit/2e62ab6c35339617b8df0b6c46a0d47b9cc41040)).

---

## Thành phần & tính năng

| Thành phần                 | Mô tả                                                                                                                                                                                                           |
| -------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| n8n                        | Nền tảng workflow no‑/low‑code, chạy dưới dạng dịch vụ Docker.                                                                                                                                                  |
| Deep\_Researcher workflow  | Tự động hoá quy trình nghiên cứu chuyên sâu, nhận truy vấn người dùng, tự sinh truy vấn SERP, gom & phân tích dữ liệu, lưu báo cáo Notion.                                                                      |
| Alert\_Prometheus workflow | Phân loại mức độ (`critical`/`warning`), sinh email chi tiết kèm hướng dẫn thủ công ([github.com](https://github.com/nguyenquan132/Deep-Researcher-n8n-MLOPS/commit/38286f797e7efaaaf11ae33c166cfde33308d067)). |
| Prometheus + Node Exporter | Thu thập số liệu hệ thống & ứng dụng.                                                                                                                                                                           |
| Alertmanager               | Ánh xạ rule ⇒ cảnh báo và đẩy sang n8n qua webhook `/webhook/alertmanager` ([github.com](https://github.com/nguyenquan132/Deep-Researcher-n8n-MLOPS/commit/2e62ab6c35339617b8df0b6c46a0d47b9cc41040)).          |
| Grafana                    | Dashboard quan sát real‑time, đã provision sẵn panel mẫu `monitor/grafana.json`.                                                                                                                                |

---

## Yêu cầu hệ thống

* **Docker 20+** và **Docker Compose v2**.
* Server Linux (có `systemd`), RAM ≥ 2 GB, CPU ≥ 2 core.
* Quyền mở port: `5678` (n8n), `3000` (Grafana), `9090` (Prometheus), `9093` (Alertmanager), `9100` (Node Exporter).
* Tài khoản **OpenAI** với model `gpt‑4o‑mini` hoặc thay thế.
* Tài khoản Gmail đã tạo **OAuth2 Client** (n8n node Gmail).
  *Biến môi trường bắt buộc:* `API_KEY_CHAT_MODEL`, `OAUTH2_GMAIL` ([github.com](https://github.com/nguyenquan132/Deep-Researcher-n8n-MLOPS/commit/38286f797e7efaaaf11ae33c166cfde33308d067)).

---

## Cài đặt

```bash
# 1. Clone dự án
git clone https://github.com/nguyenquan132/Deep-Researcher-n8n-MLOPS.git
cd Deep-Researcher-n8n-MLOPS

# 2. Tạo file .env (ở gốc repo) rồi bổ sung khoá API
cat <<'EOF' > .env
# OpenAI
API_KEY_CHAT_MODEL=sk-...
# Gmail OAuth2 (ID credential n8n)
OAUTH2_GMAIL=cred_...
# Tên host để Alertmanager gọi webhook (nếu khác)
ALERT_WEBHOOK_HOST=http://n8nmlops.io.vn
EOF

# 3. Chạy stack giám sát
cd monitor
docker compose up -d

# 4. Tự cài/khởi động n8n (ví dụ docker‑compose riêng)
#    hoặc dùng n8n Cloud, rồi import workflow ở bước tiếp theo.
```

> **Tip:** bạn có thể gom `n8n` vào cùng file `docker-compose.yml` nếu muốn “all‑in‑one”.

---

## Cấu hình n8n

1. Đăng nhập UI n8n (`http://<server>:5678/`), vào menu **Credentials** tạo:

   * **OpenAI API** → dán `API_KEY_CHAT_MODEL`.
   * **Gmail OAuth2** → dán `OAUTH2_GMAIL` và hoàn thiện OAuth flow.
2. **Import workflow**

   * `Deep_Researcher_workflow.json` – Đường dẫn form mặc định: `/form/deep_research` ([raw.githubusercontent.com](https://raw.githubusercontent.com/nguyenquan132/Deep-Researcher-n8n-MLOPS/38286f797e7efaaaf11ae33c166cfde33308d067/workflow%20n8n/Deep_Researcher_workflow.json)).
   * `Alert_Prometheus_workflow.json` – Webhook endpoint: `/webhook/alertmanager` (trùng `config.yml`).
3. Bấm **Activate** cho từng workflow.

---

## Triển khai giám sát

```bash
cd monitor
# Tùy ý chỉnh sửa rules Prometheus hoặc alertmanager/config.yml
vi prometheus/rules.yml

# Khởi động / cập nhật
docker compose up -d --build
```

* **Prometheus UI:** `http://<server>:9090`
* **Grafana UI:** `http://<server>:3000` (user/pass mặc định `admin`/`admin`).
* **Node‑Exporter metrics:** `http://<server>:9100/metrics`.

> Các rule cảnh báo mẫu trong `prometheus/rules.yml` sẽ kích hoạt Alertmanager khi vượt ngưỡng và gửi webhook tới n8n.

---

## Sử dụng Deep‑Researcher

1. Mở form tại `http://<server>:5678/form/deep_research`.
2. Nhập **Query**, **Depth** (độ sâu) & **Breadth** (độ rộng).
   *Lưu ý:* độ sâu/rộng càng lớn thì thời gian & chi phí càng cao.
3. Sau khi workflow hoàn thành, báo cáo được lưu về Notion database bạn cấu hình trong node *Notion*.
   (Thay truy cập Notion hoặc thay node khác tuỳ mục đích.)

---

> © 2025 – Made with ❤️ by Us
