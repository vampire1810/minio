# Hướng dẫn cài đặt MinIO với Docker Swarm

## Giới thiệu
MinIO là một object storage server tương thích với Amazon S3 API, được thiết kế để chạy trên các môi trường cloud-native và containerized. Cấu hình này được tối ưu cho Docker Swarm.

## Cài đặt và chạy

### 1. Deploy MinIO stack
```bash
docker stack deploy -c docker-compose.yml minio
```

### 3. Kiểm tra trạng thái
```bash
docker stack services minio
docker service ps minio_storage
```

### 4. Xem logs
```bash
docker service logs -f minio_storage
```

## Truy cập MinIO

### MinIO API
- URL: http://localhost:9000
- Access Key: `sysadmin`
- Secret Key: `Thinklabs@36`

**Lưu ý**: Cấu hình này chỉ chạy MinIO API mà không có Web Console để tiết kiệm tài nguyên cho môi trường local.

## Cấu hình

### Thay đổi thông tin đăng nhập
Chỉnh sửa file `docker-compose.yml`:
```yaml
environment:
  MINIO_ROOT_USER: your_username
  MINIO_ROOT_PASSWORD: your_secure_password
  MINIO_IDENTITY_PLUGIN_URL: https://your_domain/storage/api/v1/credentials
```

### Ports
- `9000`: MinIO API endpoint

### Cấu hình Identity Plugin
Cấu hình hiện tại sử dụng identity plugin để xác thực:
- `MINIO_IDENTITY_PLUGIN_URL`: URL endpoint cho xác thực
- `MINIO_IDENTITY_PLUGIN_ROLE_POLICY`: Policy role
- `MINIO_IDENTITY_PLUGIN_ROLE_ID`: Role ID

## Dữ liệu
Dữ liệu được lưu trữ trong Docker volume `minioapi` để đảm bảo persistence.

## Network
MinIO được cấu hình để chạy trong mạng `minio-network` với driver overlay. Đây là internal network riêng cho MinIO.

### Cách 1: Sử dụng external network (Khuyến nghị)
Nếu tRadar đã có network riêng, bạn có thể sử dụng external network:
```yaml
networks:
  tradar-network:
    external: true
```

### Cách 2: Tạo shared network
Tạo một network chung cho cả MinIO và tRadar:
```yaml
networks:
  shared-network:
    driver: bridge
```

### Cách 3: Sử dụng host network
Để đơn giản nhất, có thể sử dụng host network:
```yaml
services:
  minio:
    network_mode: "host"
```

## Lệnh hữu ích cho Docker Swarm

### Dừng MinIO stack
```bash
docker stack rm minio
```

### Update service
```bash
docker service update minio_storage
```

### Scale service (nếu cần)
```bash
docker service scale minio_storage=1
```

### Xem thông tin chi tiết
```bash
docker service inspect minio_storage
```

### Xem nodes trong swarm
```bash
docker node ls
```

## Sử dụng MinIO Client (mc) với Docker Swarm

### Cài đặt mc
```bash
# Linux/macOS
curl https://dl.min.io/client/mc/release/linux-amd64/mc -o mc
chmod +x mc
sudo mv mc /usr/local/bin/

# Windows (PowerShell)
Invoke-WebRequest -Uri "https://dl.min.io/client/mc/release/windows-amd64/mc.exe" -OutFile "mc.exe"
```

### Cấu hình mc
```bash
mc alias set local http://localhost:9000 sysadmin Thinklabs@36
```

### Tạo bucket
```bash
mc mb local/my-bucket
```

### Upload file
```bash
mc cp myfile.txt local/my-bucket/
```

### List buckets
```bash
mc ls local
```

## Giải thích cấu hình Docker Swarm

### Cấu hình Service
```yaml
services:
  storage:
    image: minio/minio:latest
    deploy:
      replicas: 1               # Chạy 1 instance duy nhất
      restart_policy:
        condition: any          # Restart trong mọi trường hợp
```

### Cấu hình Network
```yaml
networks:
  minio-network:
    driver: overlay            # Overlay network cho Docker Swarm
```
- **Network name**: `minio-network`
- **Driver**: `overlay` - cho phép containers giao tiếp qua nhiều nodes
- **Internal network**: Không phải external, tạo network riêng cho MinIO
- **Note**: Thay đổi tên network trùng với network của tRadar

### Cấu hình Volume
```yaml
volumes:
  minioapi:
    driver: local             # Lưu trữ local trên node
```
- **Volume name**: `minioapi`
- **Driver**: `local` - dữ liệu lưu trên node hiện tại
- **Persistence**: Dữ liệu được bảo toàn khi container restart

### Cấu hình Environment
- **MINIO_ROOT_USER**: `sysadmin` - Username admin
- **MINIO_ROOT_PASSWORD**: `Thinklabs@36` - Password admin
- **MINIO_IDENTITY_PLUGIN_URL**: URL endpoint cho xác thực external
- **MINIO_IDENTITY_PLUGIN_ROLE_POLICY**: Policy role `consoleAdmin`
- **MINIO_IDENTITY_PLUGIN_ROLE_ID**: Role ID `thinklabsMinioRoleArn`

### Cấu hình Ports
- **9000**: MinIO S3 API endpoint
- **Không có console**: Chỉ API, không có web interface

### Deployment Strategy
- **Placement**: Chỉ chạy trên manager nodes để đảm bảo tính ổn định
- **Replicas**: 1 instance để tránh conflict dữ liệu
- **Restart**: Tự động restart trong mọi trường hợp lỗi

### Service Naming Convention
- **Stack name**: `minio`
- **Service name trong compose**: `storage`
- **Full service name**: `minio_storage` (stack_service)
- **Container name**: `minio_storage.1.xxx`