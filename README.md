# Microservice POC Hands-On

兩支 FastAPI 微服務(商品 + 訂單)＋ 跨服務 HTTP，用 docker compose 一鍵啟動。
本 repo 是**課程主入口**：compose、Postman、逐步指南都放這裡。

```
        ┌─────────────── ms-order :8002 ───────────────┐
  用戶  │  interfaces → application → domain (Order)    │
        │                  │                           │
        └──────────────────┼───────────────────────────┘
                           │ HTTP GET /products/{id}(取價錢)
                           ▼
        ┌─────────────── ms-product :8001 ─────────────┐
        │  interfaces → application → domain (Product) │
        └──────────────────────────────────────────────┘
```

- [ms-product](https://github.com/a1234321606/ms-product) — 商品服務
- [ms-order](https://github.com/a1234321606/ms-order) — 訂單服務

## 0. 前置

- Docker Desktop(running)
- [uv](https://docs.astral.sh/uv/)(開發測試用)
- (選用)Postman(或直接 curl)

## 1. Clone 三個 repo(必須同一層，compose 用相對路徑 build)

```bash
git clone https://github.com/a1234321606/ms-product.git
git clone https://github.com/a1234321606/ms-order.git
git clone https://github.com/a1234321606/ms-bootstrap.git
```

## 2. 先跑單元測試

```bash
# 測試商品服務
cd ms-product && uv sync && pytest

# 測試訂單服務
cd ms-order && uv sync && pytest
```

## 3. 一鍵啟動

```bash
cd ms-bootstrap
docker compose up -d
docker compose ps     # 兩支都 healthy 才往下走
```

## 4. curl 快速驗證

資料存於兩支 repo 內的 SQLite 檔(`ms-product/data/product.db`、`ms-order/data/orders.db`)，compose 已掛 volume，容器重啟時資料可保留。要重置就刪掉兩支 repo 的 `data/` 資料夾。

```bash
# 建商品(201，複製回傳的 product_id)
curl localhost:8001/products -H 'Content-Type: application/json' \
     -d '{"name":"機械式鍵盤","price":350}'

# 下單(201，訂單總價應為 700)
curl localhost:8002/orders -H 'Content-Type: application/json' \
     -d '{"customer_name":"Alice","lines":[{"product_id":"p-換成你的id","quantity":2}]}'

# 查詢訂單
curl localhost:8002/orders/o-換成你的id
```

## 5. Postman(可選)

Import `postman/microservice-poc.postman_collection.json`，照 01→08 依序跑；
把 `product_id` / `order_id` 變數設成前一步回傳的值，03/07 就直接可用。

## 6. 實驗(課堂用，都是 1 分鐘可完成)

| 實驗 | 操作 | 觀察 |
|------|------|------|
| 服務掛掉的語意 | `docker compose stop ms-product`，再下一單 | **502 Bad Gateway**(ProductUnavailableError) |
| 價格權威 | 改商品 `price`，再下一單 | 新訂單用**新價格**(不是舊的、更不是客戶端傳的) |
| 上游 404 傳播 | 下單一個不存在的 `product_id` | **404**(ProductNotFoundError 忠實傳播) |

## 7. 收尾

```bash
# 停止所有服務
docker compose down
```

## 常見問題

- **port 被佔**：改 `docker-compose.yml` 左側 `8001:8001` → `18001:8001`。
- **order 一直 502**：
  1. `docker compose logs ms-order` 查看訂單服務的錯誤訊息。
  2. 檢查ms-order的`PRODUCT_API_URL` 在 compose 裡是 `http://ms-product:8001`(服務名解析，不要改 localhost)。
