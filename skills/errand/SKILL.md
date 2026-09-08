---
name: errand
description: Send a real person to a physical place in Seoul (Gangnam area) to check, photograph, verify, queue, or pick something up, and get GPS+photo evidence back. Use when the user needs eyes or hands on site — "is this store open?", "how long is the line?", "take a photo of the menu", "is the product on the shelf?".
---

# Errand — dispatch a human

Errand sends a nearby worker (a real person with the Errand iOS app) to a physical location. The worker accepts the mission on their phone, travels there, takes photos in-app, answers your questions, and the evidence is verified automatically — GPS distance from the target, capture time, and an AI vision check against the criteria you wrote. You get photos, structured answers, and a pass/fail verdict.

This plugin connects the Errand MCP server, so you call it through tools, not curl:

| Tool | Does | Costs money |
|---|---|---|
| `errand_check_coverage` | How many workers a dispatch would reach near a point | no |
| `errand_list_capabilities` | Balance, caps, allowed task types, fee schedule | no |
| `errand_quote` | Exact total for a proposed errand, plus whether it can go ahead | no |
| `errand_dispatch` | Create the mission | **yes — escrows immediately** |
| `errand_get_status` | Progress and timeline | no |
| `errand_get_result` | Evidence, answers, verdict | no |
| `errand_cancel` | Cancel while still unclaimed; full refund | no |

**Where it works:** Seoul, Gangnam district and nearby. Nowhere else yet. Always check coverage before promising anything.

**Money:** Korean won from a prepaid balance. The worker receives `reward_krw` in full. The account is charged `reward_krw + fee`, where `fee = max(500, 20% × reward_krw)`. The charge is held in escrow at dispatch and refunded in full if the mission fails, expires, or is cancelled.

Full API reference: <https://errand.be/docs>

## The one rule

**Never call `errand_dispatch` without the user's explicit yes to a stated total price.** It moves money. Everything else is free to call and safe to explore — including `errand_quote`, which exists precisely so you can state that price accurately instead of computing it yourself.

## Flow

```
check_coverage (free) → quote (free) → user confirms → dispatch → poll status → get_result
```

`errand_quote` folds the coverage check, the cap check, the fee arithmetic and the balance check into one call, so in practice you can start there and fall back to `errand_check_coverage` only when you want coverage without a reward in mind.

### 1. Coverage

You need latitude/longitude. Geocode the address or place name yourself; if you cannot, ask the user for a map link or the address.

Call `errand_check_coverage` with `lat`, `lng`, `radius_m` (3000 is a good default).

- `reachable_workers` — workers who would get the push (app switched on, seen within 24 hours, last position inside the radius). **This is the number that matters.**
- `online_workers` — app open this second. Zero here just means acceptance takes minutes rather than seconds, not that nobody will come.
- `dispatchable: false` → say so plainly: "No Errand worker is reachable near that address right now." Do not dispatch into an empty area unless the user insists after being told the escrow will simply sit until expiry and then refund automatically.

### 2. Quote

Pick a task type and a reward, then call `errand_quote` with `task_type`, `lat`, `lng`, `reward_krw` and optionally `radius_m` (default 3000). It creates nothing and reserves nothing.

**If `errand_quote` is not in the tool list**, you are talking to an older server. Work the total out yourself — `total = reward + max(500, reward × 0.2)` — call `errand_list_capabilities` to check `balance_krw` covers it, and confirm with the user exactly as below. Everything else in this section still applies.

| task_type | Use for | Typical reward (KRW) |
|---|---|---|
| `check` | Quick status: open? crowded? line length? | 3,000–6,000 |
| `verify` | Confirm a fact on site: product on shelf, price tag, sign posted | 4,000–8,000 |
| `photograph` | Specific photos: menu, storefront, product, notice | 5,000–10,000 |
| `queue` | Check or hold a spot in a physical line | 10,000–20,000 |
| `pickup` | Pick up a small item and hold or deliver it nearby | 10,000–20,000 |

A higher reward gets accepted faster. Minimum reward is ₩1,000, and the account's `max_reward_per_task_krw` caps it (default ₩20,000).

The quote comes back with everything you need to ask the question:

- `total_charge_krw` — **quote this number to the user, verbatim.** Do not recompute the fee yourself; the server is the authority and the schedule can change.
- `reward_krw` and `platform_fee_krw` — the split, if the user asks what the fee is.
- `balance_sufficient` and `balance_krw` — if false, stop and send the user to <https://errand.be/dashboard> to top up.
- `coverage` — the same shape `errand_check_coverage` returns.
- `can_dispatch` — true only when workers are reachable **and** the balance covers it.
- `note` — a plain-language reason when something blocks the dispatch.

If `can_dispatch` is false, say why and stop. Do not dispatch and hope.

If it is true, put the total in front of the user and wait:

> I can send someone to check whether the store is open and photograph the entrance. That costs **₩6,000** from your Errand balance — ₩5,000 to the worker, ₩1,000 fee — refunded in full if nobody completes it. Go ahead?

A rejected quote costs nothing, so quote freely while the user is still deciding what to ask for.

### 3. Dispatch

Only after the user says yes to the total from `errand_quote`. Call `errand_dispatch` with the same `task_type`, `lat`, `lng`, `radius_m` and `reward_krw` you quoted — changing the reward changes the price the user agreed to.

```json
{
  "task_type": "check",
  "title": "강남역 11번 출구 팝업 대기줄 확인",
  "instructions": "11번 출구 앞 팝업스토어 입구로 가서 대기줄 전체가 보이도록 사진을 한 장 찍어주세요.",
  "validation_criteria": "팝업스토어 간판과 대기줄이 한 장에 보일 것. 줄이 없으면 비어 있는 입구가 보일 것.",
  "lat": 37.4979,
  "lng": 127.0276,
  "address_hint": "강남역 11번 출구 앞",
  "radius_m": 3000,
  "reward_krw": 5000,
  "expires_in_minutes": 180,
  "report_fields": [
    { "key": "wait_count", "label": "대기 인원", "type": "number", "unit": "명" }
  ]
}
```

Field rules — the API rejects violations with `VALIDATION` and an `issues` array naming the field:

- **Write `title`, `instructions`, `validation_criteria`, and every `report_fields[].label` in Korean.** A worker in Seoul reads them on their phone. English is accepted by the API but useless to the person doing the job.
- `title` 4–120 chars · `instructions` 10–2000 · `validation_criteria` 10–1000.
- `instructions` is what to do on site, step by step. `validation_criteria` is what the photo must show — this is exactly what the automatic check grades against, so be concrete.
- `radius_m` 100–10000 · `expires_in_minutes` 10–1440. Give at least 120 so someone can travel.
- `report_fields` (max 5) are the questions the worker types answers to on site. **Always set `unit` for quantities** — otherwise "예상 대기 시간" comes back as `11` and nobody knows if that is minutes or people.
- Optional `webhook_url` (https) receives `mission.completed` / `mission.failed` / `mission.cancelled` / `mission.expired` so you need not poll.

Tell the user the `dispatch_id` and that you will report when someone accepts.

### 4. Poll

`errand_get_status` with the dispatch id.

`open` (waiting) → `accepted` (someone is on the way) → `arrived` (GPS-confirmed on site) → `submitted` (photos uploaded, verifying) → **`completed`** (verified, worker paid).

Terminal failures: `failed` (evidence rejected on both attempts), `expired` (nobody completed it in time), `cancelled`. All three refund the full charge, fee included.

Poll every 60 seconds while `open`/`accepted`/`arrived`, every 20 seconds while `submitted`. Report transitions in plain language — "a worker accepted and is heading there" — rather than dumping the status object.

### 5. Result

`errand_get_result` returns the verdict, the answers the worker typed, and the evidence.

Give the user the **answers first** (that is what they asked for), then the photos, then the verification detail. Photo URLs are signed and expire after an hour, so show or download them promptly. If `result` is `null`, `note` says why — usually nothing has been submitted yet.

### Cancel

`errand_cancel` works only while `status` is `open`. Once a worker has accepted, they may already be walking there, so it cannot be cancelled — tell the user that rather than retrying.

## Errors

Every error is `{"error": CODE, "message": "…"}`.

| Code | What to tell the user |
|---|---|
| `UNAUTHORIZED` | The API key is missing, wrong, or revoked. Re-issue at <https://errand.be> (re-issuing revokes the old key), then update the plugin's API key setting. |
| `INSUFFICIENT_BALANCE` | Top up at <https://errand.be/dashboard>. |
| `CAP_MAX_REWARD` · `CAP_DAILY_BUDGET` · `CAP_TASK_TYPE` · `CAP_AREA` | The account owner set this limit on the key. Lower the reward or ask the owner to raise the cap. **Do not retry blindly.** |
| `VALIDATION` | A field broke a rule above; `issues` names it. Fix and retry. |
| `INVALID_TRANSITION` | Usually cancelling after acceptance. A worker is already en route. |
| `RATE_LIMITED` | Wait a minute. |

## When Errand cannot help

Be straight about it. The coverage area is one district of one city, and there may be nobody online at 3am. If `dispatchable` is false or the place is outside Seoul, say so and stop — do not dispatch and hope.

For a first mission, or when the user is unsure what to ask for, they can email **support@errand.be** with the place, what they need checked, and when. A human answers and can line up a worker manually.
