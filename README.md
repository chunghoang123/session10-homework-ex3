# EX3 - So sánh Single-stage vs Multi-stage Build

## 1. Mô tả

Tái sử dụng app Node.js/Express từ ex2 (2 endpoint `GET /` và `GET /health`),
build 2 cách để so sánh kích thước và số layer:

- `Dockerfile.single`: build single-stage (giống ex2).
- `Dockerfile` (mặc định): build multi-stage 2 giai đoạn (`builder` + `runtime`).

Mục tiêu: kiểm chứng thực tế multi-stage có nhẹ hơn bao nhiêu,
hiểu layer caching và vì sao app Node đơn giản khó đạt mức giảm 50-70%
hay dưới 100MB.

## 2. Cấu trúc

- `server.js` - Express app, `GET /` trả `{status, message}`, `GET /health` trả `{status: ok}`.
- `package.json` - `ex3-backend@1.0.0`, dependency `express ^4.18.2`.
- `Dockerfile` - Multi-stage mặc định (2 stage, cùng base `node:18-alpine`).
- `Dockerfile.single` - Single-stage để so sánh.
- `.dockerignore` - Loại `node_modules`, `*.log`, `Dockerfile*`, `.git`, `logs` khỏi build context.
- `build-single.log`, `build-multistage.log` - Log build thực tế.
- `images-compare.log`, `history-single.log`, `history-multistage.log` - Kết quả đo.
- `run-multistage.log`, `curl-result.log` - Log chạy và kết quả curl.

### Dockerfile multi-stage (`Dockerfile`)

```dockerfile
# Stage 1: builder - cai dat dependencies
FROM node:18-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm install --omit=dev
COPY . .

# Stage 2: runtime - chi mang nhung gi can thiet de chay
FROM node:18-alpine
WORKDIR /app
ENV NODE_ENV=production
ENV PORT=3000
COPY --from=builder /app/package*.json ./
COPY --from=builder /app/node_modules ./node_modules
COPY --from=builder /app/server.js ./
EXPOSE 3000
CMD ["node", "server.js"]
```

### Dockerfile single-stage (`Dockerfile.single`)

```dockerfile
FROM node:18-alpine
WORKDIR /app
COPY package*.json ./
RUN npm install --production
COPY . .
ENV PORT=3000
EXPOSE 3000
CMD ["node", "server.js"]
```

## 3. Cách build (PowerShell, không dùng sudo)

```powershell
# Build single-stage
docker build -f Dockerfile.single -t hw-backend:single .

# Build multi-stage (Dockerfile mặc định)
docker build -t hw-backend:multistage .
```

Đo dung lượng và số layer:

```powershell
docker images hw-backend --format "table {{.Repository}}:{{.Tag}}`t{{.ID}}`t{{.Size}}"
docker images | Select-String "hw-backend"
docker history hw-backend:single
docker history hw-backend:multistage
docker inspect hw-backend:single --format='Size={{.Size}} Layers={{len .RootFS.Layers}}'
docker inspect hw-backend:multistage --format='Size={{.Size}} Layers={{len .RootFS.Layers}}'
```

## 4. Kết quả so sánh thực tế

Máy test: Windows + Docker Desktop 29.7.2, base `node:18-alpine`.

| Tiêu chí | Single-stage (`hw-backend:single`) | Multi-stage (`hw-backend:multistage`) |
|---|---|---|
| `docker images` SIZE | **196MB** | **186MB** |
| `docker inspect .Size` (bytes) | 47.348.637 (~45,2 MiB + base dùng chung) | 45.620.676 (~43,5 MiB + base dùng chung) |
| Chênh lệch | — | **Nhẹ hơn ~10MB (~5,1%)** |
| `docker history` (số dòng, gồm cả layer 0B config) | **16 dòng** | **17 dòng** |
| `RootFS.Layers` (filesystem layers thực) | **8** (4 base + 4 app) | **8** (4 base + 4 app) |
| Layer app lớn nhất | `RUN npm install --production`: **12,8MB** | `COPY node_modules`: **4,67MB** |
| Layer `COPY` còn lại | `COPY . .`: 16,4kB; `COPY package*.json`: 12,3kB | `COPY package*.json`: 45,1kB (kéo theo cả `package-lock.json` sinh ra ở stage builder); `COPY server.js`: 12,3kB |

Chi tiết xem file: `images-compare.log`, `history-single.log`, `history-multistage.log`.

### Vì sao chỉ giảm ~5% chứ không phải 50-70%?

1. **Cùng base image nặng.** Cả 2 đều `FROM node:18-alpine` (~180MB uncompressed,
   `docker inspect node:18-alpine` Size ~44,9MB compressed, 4 layers).
   Phần base chiếm >90% image nên tối ưu phần app chỉ tác động vài MB.
2. **App không có bước build.** Đây là Express chạy trực tiếp `node server.js`,
   không có `tsc` / bundler / native build, không có `devDependencies` hay build-tools
   (python, make, g++) để loại bỏ ở stage runtime. `npm install --omit=dev` và
   `npm install --production` cho ra cùng `node_modules`.
3. **Phần tiết kiệm được duy nhất là npm cache + file thừa.**
   - Single: layer `RUN npm install` giữ cả npm cache (`/root/.npm`) nên phình lên 12,8MB.
   - Multi: chỉ `COPY --from=builder /app/node_modules` (4,67MB), bỏ lại npm cache
     và các file không `COPY` (Dockerfile, `.dockerignore`, `*.log`).
   - Đó chính là ~8-10MB chênh lệch.
4. **Số layer không giảm.** Multi-stage còn nhiều hơn 1 dòng history
   (thêm `ENV NODE_ENV` + tách `COPY` thành 3 lệnh), nhưng `RootFS.Layers` bằng nhau (8).
   Multi-stage tối ưu **nội dung** chứ không phải lúc nào cũng giảm **số lượng** layer.

### Vì sao khó dưới 100MB với Node?

`node:18-alpine` (~170-180MB) đã là bản Alpine tối giản.
Muốn <100MB phải đổi kiến trúc, ví dụ:

- Dùng image runtime nhỏ hơn: `node:18-alpine` -> `gcr.io/distroless/nodejs18` (~120MB)
  hoặc build static rồi chạy bằng `nginx:alpine` / `scratch` (chỉ cho frontend tĩnh, không cho Express).
- Với backend Node: dùng `npm prune`, `node --experimental-sea` (single executable),
  hoặc chuyển runtime nhẹ (Bun ~80MB) nếu chấp nhận rủi ro tương thích.
- Với bài này, ~186MB là kết quả đúng và trung thực, không cố "làm đẹp số".

## 5. Layer caching là gì và Dockerfile này tận dụng ra sao?

Docker cache từng layer theo thứ tự. Layer nào không đổi thì build lại dùng cache,
không chạy lại lệnh đó.

Thứ tự trong cả 2 Dockerfile đều cố ý:

```dockerfile
COPY package*.json ./   # manifest ít đổi -> cache lâu
RUN npm install ...     # tốn ~8-10s, chỉ chạy lại khi package.json đổi
COPY . .                # source đổi thường xuyên, để cuối để không phá cache npm
```

Kiểm chứng: build lần 2 sau khi thêm `*.log` vào `.dockerignore` cho thấy
`WORKDIR`, `COPY package*.json`, `RUN npm install` đều `CACHED`, chỉ `COPY . .`
chạy lại trong 0,1s. Nếu sửa `server.js`, không phải `npm install` lại.

`.dockerignore` cũng là một phần của caching + giảm size: loại `node_modules`,
`*.log`, `Dockerfile*`, `.git` khỏi build context (lần build cuối context chỉ còn 91B).

## 6. Cách run và test bản multi-stage

Lưu ý: port 8081 trên máy test đã bị container ex5 chiếm,
nên dùng port host **8091** để tránh đụng độ.

```powershell
# Xóa container cũ nếu tồn tại (bỏ qua lỗi nếu chưa có)
docker rm -f app-multi

# Chạy multi-stage, map host 8091 -> container 3000
docker run -d --name app-multi -p 8091:3000 hw-backend:multistage

# Xem log (phải thấy "Server dang chay tai cong 3000")
docker logs app-multi

# Test endpoint chính
curl.exe -s http://localhost:8091/

# Test health check
curl.exe -s http://localhost:8091/health

# Kiểm tra file thực sự có trong runtime (chỉ package*, node_modules, server.js)
docker exec app-multi ls -lh /app

# Dọn dẹp
docker rm -f app-multi
```

### Kết quả thực tế (đã lưu trong `run-multistage.log`, `curl-result.log`)

```text
# docker logs app-multi
Server dang chay tai cong 3000

# curl http://localhost:8091/
{"status":"success","message":"Hello from Docker Multi-stage!"}

# curl http://localhost:8091/health
{"status":"ok"}

# docker exec app-multi ls -lh /app
total 44K
drwxr-xr-x   70 root     root  4.0K node_modules
-rw-r--r--    1 root     root  28.8K package-lock.json
-rwxr-xr-x    1 root     root   173 package.json
-rwxr-xr-x    1 root     root   363 server.js
```

Nhận xét: runtime không chứa `Dockerfile`, `.dockerignore`, `*.log` —
đúng ý đồ `COPY --from=builder` chọn lọc. `package-lock.json` có mặt vì
`COPY --from=builder /app/package*.json` khớp cả 2 file (npm sinh ra ở stage builder).

## 7. Dọn dẹp

```powershell
docker rm -f app-multi
# Muốn xóa image (tùy chọn):
# docker rmi hw-backend:single hw-backend:multistage
```
