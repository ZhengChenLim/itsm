# IT Portal REST API (v1)

Django REST Framework backend on `http://127.0.0.1:8100`. The Next.js frontend (port 3000) proxies `/api/*` to it.

## Conventions
- **Auth:** sign in with `POST /api/auth/login`; the API sets an HttpOnly `portal_session` cookie (SameSite=Lax, Secure when `DJANGO_DEBUG=0`). Sessions end after 8 h idle, 24 h after sign-in, or on sign out. Not signed in → `401 {"error": "Please sign in"}`.
- **CSRF:** requests that change data (POST/PUT/PATCH/DELETE) must send `X-Requested-With: XMLHttpRequest`, otherwise `403`.
- JSON in/out, except claim create/resubmit (multipart) and file download (binary).
- Times are **unix seconds** (int) or null. Money is `amount_cents` (int) + `currency` (3 letters).
- **Pagination:** list endpoints marked *paged* accept `page` (from 1) and `page_size` (1-100, default 20). With `page` they answer `{results:[...], total, page, page_size, pages}`; without it, a plain array (as before).
- **List filters** (tickets, claims, audit log): `range` = `24h`, `3d`, `7d`, `30d`, or `custom` with `from` / `to` (YYYY-MM-DD, whole days, server time); `q` = text search. Tickets filter on when they were raised, claims on when they were submitted, audit events on when they happened.
- **Sorting:** `sort=<key>` ascending or `sort=-<key>` descending, on the whole list (all pages). Tickets: `created, ref, title, priority, status, pool, assignee, requester, due` (resolve due). Claims: `submitted, ref, title, amount, status, requester`, plus `decided` for `scope=history`. Audit log: `at, action, user`. Without `sort`: newest first (work queues `assigned` / `pool`: by SLA time left). Unknown key → 400.
- Errors: `{"error": "message"}` with 400 (validation), 401, 403 (not allowed), 404, 409 (duplicate claim, see below).

## Sign in
| Method | Path | Notes |
|---|---|---|
| POST | `/api/auth/login` | `{login, password}`. `login` is the user ID (e.g. `E1001`) or email, case-insensitive; users without email use the user ID. 400 `User ID / email or password is incorrect` (same message for unknown users). 5 wrong passwords lock the account for 15 min. Sets the session cookie |
| POST | `/api/auth/logout` | Ends the session and clears the cookie |
| POST | `/api/auth/password` | Signed in: `{current, new}`. New password: at least 10 characters, not common, not only numbers, not similar to the user ID / name / email. Other sessions of the user are signed out |

Sign-ins, failed attempts, lockouts and password changes are in the audit log (never the password).

## Users & meta
| Method | Path | Notes |
|---|---|---|
| GET | `/api/users` | Team lists (signed in): `[{id,login_id,name,email (null if none),role,department,department_id,pool_id,pool,skills,capacity,available,manager_id}]`. role ∈ `employee, it_member, it_manager, hod, vp, finance` |
| GET | `/api/me` | user fields above + `managed_pools:[pool ids]`, `pending_approvals` (ticket approvals waiting on me), `pending_claims` (claim approvals waiting on me), `unread`, `manager` (name or null), `is_finance_hod` (bool), `can_manage_catalog` (bool: HOD or IT manager), `is_cfo` (bool), `pending_limit_changes` (CFO: limit changes waiting), `theme` (dark|light, default dark) |
| PATCH | `/api/me` | Personal settings: `{theme: "dark"|"light"}` (nothing else is editable). Returns the GET shape |
| GET | `/api/meta` | `{pools:[{id,code,name,manager_id,manager}], categories:[{id,name,group_id,group,ticket_type,active,pool_id,pool,skill,approval_chain,default_priority}] (all Level 2 items incl. inactive; approval_chain is a comma string here for backwards compatibility), sla:[{priority,label,response_mins,resolve_mins,weight}], settings:{key:value}, metrics:{key:{label,unit,higher_better}}, claim_categories:[{value,label}], hod_limit_myr:number (the limit that applies to the signed-in user's claims; -1 = no limit), top_approver:"VP"|"C-suite"|null (who approves above the HOD for the user's department)}` |
| PATCH | `/api/users/<id>` | IT manager (own pool) / HOD: `{available:0|1, capacity, skills}` |

## ITSM - tickets
| Method | Path | Notes |
|---|---|---|
| POST | `/api/tickets` | `{type: "INCIDENT"|"REQUEST", category_id (an item of that type), title, description}`. Priority is **not** accepted from the requester: it is the item's `default_priority` → `201 {id}`. Ref is `INC000123` for incidents, `REQ000045` for requests |
| GET | `/api/tickets?scope=&status=&pool_id=&type=&priority=&q=&range=` | optional `type` = INCIDENT/REQUEST, `priority` = P1-P4, `q` searches ref, title, requester, assignee and category, plus the time filters. scope: `mine` (default), `assigned` (my open work), `pool` (IT only; pool_id optional), `confirm` (resolved, my managed pools), `closed_mine`. `assigned`/`pool` sorted by SLA time left. *Paged* |
| GET | `/api/tickets/<id>` | ticket + `events:[{id,at,action,detail,user_id,user}]`, `approvals:[{id,seq,step,approver_id,approver,status,comment,decided_at}]`, `alloc_explain` (obj or null), `can_manage`, `pool_members:[{id,name}]`, `claims:[{id,ref,title,amount_cents,currency,status}]` |
| POST | `/api/tickets/<id>/<action>` | action ∈ `pickup, hold, pending, resume, resolve, confirm, reopen, assign, auto_assign, cancel, comment, priority`. Body `{note, sla_met (confirm), user_id (assign), priority (priority: P1..P4, assignee or pool manager, note required; SLA deadlines are recalculated from the queue time)}` → `{ok:true}` |
| GET | `/api/tickets/<id>/candidates` | IT manager/HOD: allocator ranking |
| GET | `/api/approvals` | ticket approvals pending for me: `[{id,ticket_id,seq,step,status,ref,title,description,priority,created_at,requester,category}]` |
| GET | `/api/approvals?scope=history` | ticket approvals I decided, always paged: `{results:[{id,ticket_id,ref,title,type,priority,category,category_group,requester,step,decision (APPROVED\|REJECTED),comment,decided_at,ticket_status}], ...}`. `status` = decision filter; time filters use `decided_at`; `sort`: `decided, ref, title, requester, decision, step` |
| POST | `/api/approvals/<id>` | `{decision:"APPROVED"|"REJECTED", comment}` |
| POST | `/api/pools/<id>/allocate` | IT manager/HOD → `{assigned:n}` |
| POST | `/api/monitor/run` | IT manager/HOD: run SLA monitor now |
| GET | `/api/dashboard?pool_id=` | IT staff: `{pool_id,period_days,kpis:[{metric,target,label,unit,higher_better,actual,on_track}],actual:{sla_compliance,response_compliance,avg_resolution_hours,backlog,resolved},status_counts:[{status,n}],at_risk:[ticket],members:[{id,name,skills,available,capacity,load,open,at_risk,resolved,sla_pct,avg_hrs}],pending_confirm}` |
| PUT | `/api/kpis` | `{pool_id|null, targets:{metric:number}, period_days}` |
| PATCH | `/api/settings` | HOD: `{settings:{w_skill,w_load,w_perf,w_fair,warn_pct,critical_pct,auto_reassign_unpicked}, sla:[{priority,response_mins,resolve_mins}]}`. Values are range-checked; changes are audited |

**Ticket object** (lists & detail): `{id,ref,type,category_group_id,category_group,title,description,category_id,category,priority,requester_id,requester,pool_id,pool,assignee_id,assignee,status,created_at,queued_at,assigned_at,responded_at,resolved_at,closed_at,response_due,resolve_due,hold_reason,resolution,sla_confirmed,sla_confirm_note,response_breached,resolve_breached,sla:{clock:"not_started"|"running"|"paused"|"stopped",resolve_due,remaining_secs,pct_used,breached,response_due,response_met}}`
Statuses: `PENDING_APPROVAL, IN_POOL, ASSIGNED, IN_PROGRESS, ON_HOLD, PENDING, RESOLVED, CLOSED, REJECTED, CANCELLED`.

## Service catalog (Level 1 → Level 2, separate for incidents and requests)
Level 1 = group (e.g. *Software*), Level 2 = item (e.g. *Git*). The Level 2 item decides routing: IT pool, skill, default priority and approval chain. Admins = **HOD and IT managers**. Nothing is deleted: set `active:false` (inactive groups/items disappear from the raise form; old tickets keep them).

| Method | Path | Notes |
|---|---|---|
| GET | `/api/catalog?type=INCIDENT\|REQUEST&all=1` | `[{id,name,ticket_type,active,sort,updated_at,updated_by,items:[{id,name,group_id,pool_id,pool,skill,approval_chain:["LINE_MANAGER",...],default_priority,active,updated_at,updated_by}]}]` sorted by sort,name. Without `all=1`: only active groups/items (any signed-in user; used by the raise form). `all=1`: admins only. `can_edit` is in `/api/me` as `can_manage_catalog` |
| POST | `/api/catalog/groups` | `{ticket_type, name}` → `201 {id}`. Name unique per type (case-insensitive) |
| PATCH | `/api/catalog/groups/<id>` | `{name?, active?, sort?}` (ticket_type cannot change) |
| POST | `/api/catalog/items` | `{group_id, name, pool_id, skill, approval_chain:[subset of LINE_MANAGER, POOL_MANAGER, HOD], default_priority:P1..P4}` → `201 {id}`. Name unique within its group |
| PATCH | `/api/catalog/items/<id>` | any of the item fields + `active` (group_id may move it to another group of the same type) |

## IT claims (invoice / receipt approval: Manager → HOD → VP or C-suite → Finance)
| Method | Path | Notes |
|---|---|---|
| POST | `/api/claims` | **multipart/form-data**: `title, doc_type (INVOICE|RECEIPT), category, vendor, doc_no, doc_date (YYYY-MM-DD), amount, currency, cost_center, description, ticket_ref (optional, e.g. INC000141), confirm_duplicate ("true" to override), files (repeat, PDF/PNG/JPG, ≤5 × 10 MB)` → `201 {id}`. Duplicate vendor+doc_no → **409** `{"error": "...", "code": "duplicate"}` |
| GET | `/api/claims?scope=&status=&q=&range=` | `q` searches ref, title, vendor, document no. and requester, plus the time filters. scope: `mine` (default), `inbox` (waiting on me), `history` (I decided: rows add `my_decided_at`, `my_decision`; time filter and order use my latest decision), `all` (finance only). Newest first. *Paged* |
| GET | `/api/claims/<id>` | claim + `approvals:[{id,round,seq,step,approver_id,approver,status,comment,decided_by,decided_by_name,decided_at}]`, `attachments:[{id,filename,mime,size,uploaded_at,removed}]`, `events:[{id,at,action,detail,user_id,user}]` |
| POST | `/api/claims/<id>/approve` \| `reject` \| `return` | `{comment}` (required for reject/return) |
| POST | `/api/claims/<id>/resubmit` | multipart, same fields as create + `remove_files` (repeat attachment ids) + `note` |
| POST | `/api/claims/<id>/cancel` | requester |
| POST | `/api/claims/<id>/paid` | finance: `{payment_ref}` |
| GET | `/api/claims/files/<attachment id>` | binary, inline |
| GET | `/api/claims/summary` | finance: `{by_status:{STATUS:{count,amount_cents}}, by_step:{STEP:n}}` |
| GET | `/api/claims/settings` | `{hod_limit_myr (general), my_limit_myr, my_limit_source, can_edit (Finance HOD), can_approve (CFO), individual:[{user_id,name,kind (HOD|VP|C-suite),departments,limit_myr (null = default),updated_by,updated_at}], pending_changes:[{id,user_id,name,limit_myr (null = remove),requested_by,requested_at}], history:[{at,detail,user}]}`; the last three for finance and the CFO only |
| PATCH | `/api/claims/settings` | Finance HOD only: `{hod_limit_myr}` (general). MYR claims at or below the limit skip the VP or C-suite step; `-1` = every claim needs it |
| POST | `/api/claims/limits` | Finance HOD only: `{user_id, limit_myr}` individual limit for a department HOD, VP or C-suite approver (`-1` allowed). `{result: "applied"}` for an HOD; `{result: "pending"}` for a VP / C-suite (waits for the CFO; replaces an older pending request) |
| DELETE | `/api/claims/limits/<user_id>` | Finance HOD only: remove an individual limit (back to the default). Same `result` rule |
| POST | `/api/claims/limit-changes/<id>/approve` / `reject` | CFO only: `{comment}` (required to reject). Approving applies the limit |

**Claim object**: `{id,ref,title,doc_type,category,vendor,doc_no,doc_date,amount_cents,currency,cost_center,description,ticket_id,ticket_ref,requester_id,requester,department,status,round,created_at,submitted_at,decided_at,paid_at,payment_ref,current_step,current_approver,my_turn}`
Statuses: `PENDING, RETURNED, APPROVED (awaiting payment), PAID, REJECTED, CANCELLED`. Steps: `MANAGER, HOD, VP, FINANCE` (the `VP` step is held by a C-suite approver in departments without a VP; claims and approvals carry `current_step_label` / `step_label` = "VP" or "C-suite"); approval status `WAITING, PENDING, APPROVED, REJECTED, RETURNED, SKIPPED`.
Claim categories: `HARDWARE, SOFTWARE, SERVICES, TELECOM, CLOUD, OTHER`.

## System admin (role `admin` only)
| Method | Path | Body / response |
|---|---|---|
| GET | `/api/system/settings` | `{settings:[{key,label,value,type (int|float),min,max,updated_by,updated_at}]}` (the claims HOD limits are Finance settings, not here) |
| PATCH | `/api/system/settings` | `{settings:{key:value}}` for `w_skill,w_load,w_perf,w_fair,warn_pct,critical_pct,auto_reassign_unpicked,monitor_interval_secs`. Range-checked, warn % < critical %; each change is audited |
| GET | `/api/system/audit` | Query `source` (ticket|claim|system), the time filters, `q` (action/detail text). Always paged: `{results:[{at,source,ref,action,detail,user,user_id}], total, page, page_size, pages}`, newest first |
| GET | `/api/system/audit/export` | Same filters. CSV download (`time,source,ref,action,detail,user`, max 100,000 rows). The export itself is audited |

## User accounts - system admin only
| Method | Path | Body / response |
|---|---|---|
| GET | `/api/system/users?q=&role=&status=active\|inactive\|locked` | `{users:[{id,login_id,name,email,role,department_id,department,manager_id,manager,pool_id,pool,is_active,locked,must_change_password,has_password,last_login}], roles:[{value,label}], departments:[{id,name}], pools:[{id,code,name}]}` |
| POST | `/api/system/users` | `{login_id, name, email?, role, department_id?, manager_id?, pool_id? (IT roles only)}` → `201 {id, temp_password}`. The temporary password is returned once and must be changed at first sign-in |
| PATCH | `/api/system/users/<id>` | Any of the fields above plus `is_active`. Validated (unique user ID / email, no manager loops). Deactivating signs the user out everywhere. Admins cannot deactivate or demote themselves, and the last active admin cannot be removed |
| POST | `/api/system/users/<id>/reset-password` | `{temp_password}` (once). Unlocks the account, signs it out everywhere, and forces a password change |
| POST | `/api/system/users/<id>/unlock` | Clears a lockout. Returns the user row |

All changes are audited (`USER_CREATED`, `USER_UPDATED` with the changed fields, `PASSWORD_RESET`, `USER_UNLOCKED`); passwords never appear in the log.

A user signed in with a temporary password (`must_change_password` in `/api/me`) gets `403` everywhere except `/api/me`, `/api/meta`, `/api/auth/password` and `/api/auth/logout` until they change it.

## Email (SMTP) - system admin only
| Method | Path | Body / response |
|---|---|---|
| GET | `/api/system/smtp` | `{smtp_enabled, smtp_host, smtp_port, smtp_security (none|starttls|ssl), smtp_username, smtp_from_email, smtp_from_name, portal_url, password_set, ready, outbox:{pending,failed,sent_24h,last_error}}`. The password is never returned |
| PATCH | `/api/system/smtp` | Any of the fields above, plus `password` (replaces the saved one; leave out to keep) or `clear_password: true`. Validated; changes audited without the password. Returns the GET shape |
| POST | `/api/system/smtp/test` | `{to}` (default: the admin's email). Sends now with the saved settings; 400 with the reason on failure. Audited |

## Notifications
| GET | `/api/notifications` | `[{id,ticket_id,claim_id,ref,level (info|warn|critical|breach),message,at,read}]` newest first, 50 |
|---|---|---|
| POST | `/api/notifications/read` | mark all read |
| GET | `/api/notification-settings` | `{email_enabled, email, kinds:[{key,label,description,in_app,email}]}` (only the kinds that apply to the user's role) |
| PUT | `/api/notification-settings` | `{prefs:{kind:{in_app:bool,email:bool}}}`. Kinds: `ticket_mine, ticket_approval, ticket_work, sla, claim_mine, claim_approval, limit_change` |
