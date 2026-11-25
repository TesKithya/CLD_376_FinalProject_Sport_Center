# Sports Facility Reservation & Player Matching System

Cloud-native microservices (Node.js · Express · MongoDB · Docker) that let customers reserve courts and fields — basketball, football, badminton, tennis — and optionally **open a reservation for matching** so other players or groups can find and join the session. Maintenance tickets automatically make facilities unreservable.

## Quick start
```bash
docker compose up --build -d          # first run builds all 7 images
curl http://localhost:3000/health/services
node scripts/smoke-test.js            # optional end-to-end check (Node 18+)
docker compose logs -f api-gateway
docker compose down                   # add -v to wipe the database
```
Everything is reached through **http://localhost:3000**. The `.env` files ship with development values — change `JWT_SECRET`, `INTERNAL_API_KEY` and `STAFF_SIGNUP_CODE` before deploying anywhere real (`.env` is git-ignored; copy from `.env.example`).

## Architecture
```
                      ┌────────────────────────── Docker network (private) ───────────────────────────┐
 Client ──► API Gateway ──► Registration ◄── Authentication (issues JWT)
 :3000       verifies JWT ─► Facility ◄──────────────┐
             checks role  ─► Reservation ──► Facility (exists + active?)
             adds identity│                ──► Maintenance (slot blocked?)
                          │                ──► Matching (open / close session)
                          ─► Matching ─────► Reservation (still valid?)
                          ─► Maintenance ──► Facility (set status)
                                          ──► Reservation (conflicts / cancel)
                      └────────────────────────────────────────────────────────────────────────────────┘
                                   MongoDB: registration_db · auth_db · facility_db · reservation_db · matching_db · maintenance_db
```

| # | Service | Port | Database | Answers |
|---|---|---|---|---|
| 1 | API Gateway | 3000 (published) | — | Where does the request go, and is the user authorized? |
| 2 | Registration | 3001 | registration_db | Who are the users? |
| 3 | Authentication | 3002 | auth_db | Is the user authenticated? |
| 4 | Facility | 3003 | facility_db | What facilities exist? |
| 5 | Reservation | 3004 | reservation_db | Who reserved a facility and when? |
| 6 | Matching | 3005 | matching_db | Who wants to play together? |
| 7 | Maintenance | 3006 | maintenance_db | What facilities need repair? |

### Design decisions
- **Database per service.** One Mongo container, six databases; no service reads another's collections — only HTTP.
- **Zero-trust inside.** Only the gateway is published. Every service rejects requests lacking `x-internal-key`, and only trusts the identity headers the gateway derives from a verified JWT. `/internal/*` routes are unreachable from outside and restricted to the `system` role.
- **Roles:** `customer`, `manager`, `admin`. Staff roles need `STAFF_SIGNUP_CODE` at registration.
- **Resilience:** `/health` on every service, Docker health checks + `depends_on: service_healthy`, 5 s timeouts on service calls, clean 503s when a dependency is down, matching notifications are best-effort.
- **Correctness under concurrency:** double-booking guard (check, insert, re-check), atomic join with `$expr` capacity check, atomic group join.
- **Stateless containers**, config entirely from environment, non-root images, graceful SIGTERM.

### Typical flow
1. `POST /api/auth/register` → `POST /api/auth/login` → use `Authorization: Bearer <token>`.
2. Admin creates facilities: `POST /api/facilities`.
3. Customer books: `POST /api/reservations` with `isOpenForMatching: true, playersNeeded: 3`.
4. Others browse `GET /api/matching/matches?type=badminton` and `POST /api/matching/matches/:id/join` (optionally `{ "groupId": "..." }`).
5. Staff schedules repairs: `POST /api/maintenance` — conflicting reservations return 409 unless `force: true`.

## API endpoints

All client traffic goes to **http://localhost:3000** (nginx load balancer → two API Gateway instances). Send `Authorization: Bearer <token>` on every protected route. Any path containing `/internal` is blocked at the gateway (404). Per-service details are in each service's README.

### Gateway
| Method | Path | Access |
|---|---|---|
| GET | `/health` | Public |
| GET | `/health/services` | Public — aggregated status of all downstream services (503 if any is not `up`) |

### Auth — `/api/auth` (rate limit: 50 requests / 15 min)
| Method | Path | Access | Handled by |
|---|---|---|---|
| POST | `/api/auth/register` | Public | Registration `/register` |
| POST | `/api/auth/login` | Public | Authentication `/login` |
| GET | `/api/auth/me` | Authenticated | Authentication `/me` |

`register` accepts `name, email, password (min 8), phone, role, staffCode`. Everyone is a `customer` unless `role: "admin"` is sent with a valid `staffCode` (`STAFF_SIGNUP_CODE`).

### Facilities — `/api/facilities`
| Method | Path | Access |
|---|---|---|
| GET | `/api/facilities` | Authenticated |
| GET | `/api/facilities/:id` | Authenticated |
| POST | `/api/facilities` | Admin |
| PUT | `/api/facilities/:id` | Admin |
| PATCH | `/api/facilities/:id/status` | Admin |
| DELETE | `/api/facilities/:id` | Admin |

### Reservations — `/api/reservations`
| Method | Path | Access |
|---|---|---|
| GET | `/api/reservations/availability` | Authenticated |
| GET | `/api/reservations/my` | Authenticated |
| GET | `/api/reservations` | Admin |
| POST | `/api/reservations` | Authenticated |
| GET | `/api/reservations/:id` | Authenticated (ownership / staff rules enforced in the service) |
| PATCH | `/api/reservations/:id/cancel` | Authenticated |
| PATCH | `/api/reservations/:id/matching` | Authenticated |

### Matching — `/api/matching` (all Authenticated)
| Method | Path | Description |
|---|---|---|
| GET | `/api/matching/matches` | List open matches |
| GET | `/api/matching/matches/mine` | My matches |
| GET | `/api/matching/matches/:id` | Match details |
| POST | `/api/matching/matches/:id/join` | Join a match (optional `groupId`) |
| POST | `/api/matching/matches/:id/leave` | Leave a match |
| POST | `/api/matching/groups` | Create a group |
| GET | `/api/matching/groups` | List groups |
| GET | `/api/matching/groups/:id` | Group details |
| POST | `/api/matching/groups/:id/join` | Join a group |
| POST | `/api/matching/groups/:id/leave` | Leave a group |
| DELETE | `/api/matching/groups/:id` | Delete a group |

### Maintenance — `/api/maintenance` (Admin only)
| Method | Path |
|---|---|
| POST | `/api/maintenance` |
| GET | `/api/maintenance` |
| GET | `/api/maintenance/:id` |
| PUT | `/api/maintenance/:id` |
| PATCH | `/api/maintenance/:id/status` |

### Internal endpoints (service-to-service only, never exposed through the gateway)
| Service (port) | Method | Path | Purpose |
|---|---|---|---|
| Registration (3001) | POST | `/internal/verify` | Credential check, called by Authentication |
| Registration (3001) | GET | `/internal/users/:id` | User lookup |
| Authentication (3002) | POST | `/verify` | JWT verification |
| Reservation (3004) | GET | `/internal/conflicts` | Find reservations conflicting with a slot |
| Reservation (3004) | POST | `/internal/cancel-conflicts` | Cancel conflicting reservations (maintenance `force`) |
| Matching (3005) | POST | `/internal/matches` | Create/update a match from a reservation |
| Matching (3005) | POST | `/internal/matches/reservation/:reservationId/close` | Close the match when its reservation is cancelled |
| Maintenance (3006) | GET | `/facility/:facilityId/check` | Is the facility under maintenance? (`system` role) |

Every service also exposes its own `GET /health`.

## Known limitations / next steps
- **No notification service:** participants aren't told when a session is cancelled (the match just flips to `cancelled`). A message queue (RabbitMQ/Kafka) would replace the synchronous HTTP calls and enable this.
- Availability is computed per **UTC** day; add a venue time zone and opening hours for production.
- JWTs are not revocable before expiry (no refresh/blacklist).
- No automated unit tests yet; `scripts/smoke-test.js` is the end-to-end check.
- Single Mongo node; use a replica set for production (also enables transactions).
