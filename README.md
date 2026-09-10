# report-hub

Cấu hình Docker Compose để tự host [ReportPortal](https://reportportal.io/) — công cụ tổng hợp báo cáo test tự động — chạy tại domain `reports.phpfox.us`.

## Kiến trúc

- **gateway** (Traefik) — reverse proxy nội bộ, tự động discover các service qua Docker labels, expose UI ở cổng `8090` và dashboard Traefik ở `8091`.
- **postgres** — lưu metadata, user, kết quả test.
- **rabbitmq** — message broker cho giao tiếp giữa các service (reporting, analyzer, jobs).
- **opensearch** — index log test phục vụ auto-analysis (chỉ chạy khi bật profile phân tích).
- **migrations** — chạy migration schema DB một lần khi khởi động.
- **index, ui, api, uat, jobs** — các service lõi của ReportPortal (routing, giao diện web, REST API, xác thực, job nền).
- **analyzer** (+ `analyzer-storage-init`) — service auto-analysis dựa trên OpenSearch, dùng volume riêng đã được chỉnh quyền sở hữu cho user rootless.
- **nginx/reports.phpfox.us.conf** — cấu hình reverse proxy ngoài (nginx trên host), nhận HTTPS/QUIC và forward vào Traefik gateway; SSL certificate do Certbot quản lý.
- **rabbitmq/** — file cấu hình bổ sung (`extra.conf`) và danh sách plugin (`enabled_plugins`) được mount vào container RabbitMQ.

## Yêu cầu

- Docker và Docker Compose v2
- Domain đã trỏ về server, chứng chỉ SSL qua Certbot nếu dùng cấu hình nginx đi kèm

## Cài đặt

1. Sao chép file môi trường mẫu và điều chỉnh giá trị (mật khẩu DB, RabbitMQ, admin mặc định, cổng...):

   ```bash
   cp env.example .env
   ```

2. Khởi động stack theo các profile cần dùng:

   ```bash
   docker compose --profile core --profile infra --profile analyzer up -d
   ```

   Profile được cấu hình sẵn trong `.env` qua biến `COMPOSE_PROFILES`.

3. Truy cập ReportPortal UI tại `http://localhost:${RP_UI_PORT}/ui` (mặc định cổng `8090`), hoặc qua domain `reports.phpfox.us` nếu đã cấu hình nginx + Certbot.

4. Đăng nhập lần đầu bằng tài khoản `superadmin` với mật khẩu ở `RP_INITIAL_ADMIN_PASSWORD`, sau đó đổi mật khẩu ngay trong UI.

## Profiles

| Profile | Nội dung |
|---|---|
| `core` | Các service bắt buộc để chạy ReportPortal (gateway, postgres, rabbitmq, migrations, index, ui, api, uat, jobs) |
| `infra` | Hạ tầng phụ trợ (postgres, rabbitmq, gateway) |
| `analyzer` | Service auto-analysis (opensearch, analyzer, analyzer-storage-init) |
| `""` (rỗng) | Profile mặc định của upstream, cần giữ để OpenSearch và các service optional khác được load đúng |

## Lưu ý

- File `.env` chứa thông tin nhạy cảm (mật khẩu DB, RabbitMQ, admin) và đã được `.gitignore` — không commit file này.
- Dữ liệu được lưu ở các named volume: `postgres`, `storage` (dùng chung cho api/uat/jobs), `analyzer-storage`, `opensearch`.
- Muốn dùng S3/MinIO cho storage thay vì filesystem, xem các biến `DATASTORE_*` được comment sẵn trong `docker-compose.yml`.
- HTTPS được để tắt ở tầng Traefik (`gateway`); TLS thực tế được xử lý ở tầng nginx ngoài (`nginx/reports.phpfox.us.conf`) qua Certbot.
