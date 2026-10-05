# Microservices Demo — Traefik + Flask + PostgreSQL + Redis

Implements the architecture:

```
Internet → Traefik (:80 / :8080) → frontend (:3000)
                                  → user-service (:5000)    ─┐
                                  → product-service (:5001) ─┼→ order-service (:5002) → PostgreSQL / Redis
```

## Services

| Service          | Tech            | Role                                                                 |
|-------------------|-----------------|----------------------------------------------------------------------|
| traefik           | Traefik v2.11   | Reverse proxy / API gateway. Dashboard at `:8080`.                  |
| frontend           | nginx + HTML/JS | Static UI that calls the API through `/api/*`.                      |
| user-service       | Flask + Postgres| CRUD for users (`users_db`).                                        |
| product-service    | Flask + Postgres| CRUD for products + a `/reserve` endpoint to decrement stock.       |
| order-service      | Flask + Postgres + Redis | Validates the user (calls user-service), reserves stock (calls product-service), persists the order, and caches reads in Redis. |
| postgres           | PostgreSQL 16   | One database per service: `users_db`, `products_db`, `orders_db`.   |
| redis              | Redis 7         | Caches order reads (30s TTL) to cut DB load.                        |

Routing is done entirely with **Traefik Docker labels** — no static Traefik config file is needed:

- `PathPrefix('/')` → frontend
- `PathPrefix('/api/users')` → user-service (prefix stripped)
- `PathPrefix('/api/products')` → product-service (prefix stripped)
- `PathPrefix('/api/orders')` → order-service (prefix stripped)

`user-service`/`product-service` sit on both the `web` network (so Traefik can reach them) and the `backend` network (so they can reach Postgres/Redis and each other without being exposed publicly).

## Run it

```bash
cd microservices-project
docker compose up --build
```

Then open:

- **UI:** http://localhost/
- **Traefik dashboard:** http://localhost:8080/dashboard/

## Try the API directly

```bash
# Health checks
curl http://localhost/api/users/health
curl http://localhost/api/products/health
curl http://localhost/api/orders/health

# List seed data
curl http://localhost/api/users/
curl http://localhost/api/products/

# Create a user
curl -X POST http://localhost/api/users/ \
  -H "Content-Type: application/json" \
  -d '{"name": "Ada Lovelace", "email": "ada@example.com"}'

# Create a product
curl -X POST http://localhost/api/products/ \
  -H "Content-Type: application/json" \
  -d '{"name": "USB-C Cable", "price": 9.99, "stock": 50}'

# Place an order (order-service calls user-service + product-service internally)
curl -X POST http://localhost/api/orders/ \
  -H "Content-Type: application/json" \
  -d '{"user_id": 1, "product_id": 1, "quantity": 2}'

# Read it back (served from Redis cache on repeat calls within 30s)
curl http://localhost/api/orders/1
```

## What happens on `POST /api/orders`

1. `order-service` calls `GET user-service/<user_id>` to confirm the user exists.
2. `order-service` calls `POST product-service/<product_id>/reserve` to atomically
   check and decrement stock (`UPDATE ... WHERE stock >= quantity`).
3. If both succeed, the order is written to `orders_db` in Postgres.
4. The order is cached in Redis (`order:<id>`, 30s TTL) and the `orders:all`
   list cache is invalidated so the next list call reflects the new order.

## Notes / next steps for production

- Traefik's dashboard (`--api.insecure=true`) is enabled for local dev only — disable it
  or put it behind auth before deploying anywhere public.
- Add HTTPS via Traefik's Let's Encrypt resolver for a real domain.
- Add a healthcheck-based `depends_on` + retry/backoff in `order-service`'s HTTP calls
  to the other two services for resilience during rolling restarts.
- Swap the hardcoded DB credentials for secrets (Docker secrets / Vault / env files
  excluded from version control).
