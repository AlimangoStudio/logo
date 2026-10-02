# PAPPS Backend Patch Specification
## Doctor's Corner + Vision Assist + SMS Gateway
**Target:** Laravel 11 / PAPPS backend only  
**Do not modify frontend. Do not deploy.**

---

# 1. Scope

Implement backend support for:

1. Doctor's Corner screening-first workflow.
2. Breast HA Vision Assist.
3. Gluteal HA Vision Assist.
4. Hiko Nose Lift Vision Assist.
5. Patient-facing generated-image presentation sessions.
6. SMS Gateway platform configuration in Landlord.
7. SMS Gateway tenant phone pairing by QR.
8. Android gateway device enrollment, authentication, heartbeat, and revocation.

This must integrate with the existing PAPPS tenant, membership, patient, auth, storage, audit, and routing conventions. Inspect the repository before implementing and reuse existing systems instead of creating duplicates.

---

# 2. Core Clinical Workflow

All Vision Assist procedures must follow:

```text
SCREEN
  ↓
CLEAR / PHYSICIAN REVIEW / DEFER
  ↓
CAPTURE
  ↓
ANALYZE
  ↓
MAP
  ↓
PLAN
  ↓
SIMULATE
  ↓
DOCTOR CONFIRM
  ↓
PATIENT PRESENTATION
  ↓
SAVE TO EMR
```

If screening status is `defer`, photo capture / visual planning must not proceed.

Screening is not a diagnosis engine. It should classify findings into:

- `clear`
- `review`
- `defer`

The physician must be able to override a `review` status with a documented decision. A `defer` state requires an explicit physician action before planning can continue.

---

# 3. Procedure Keys

Use stable backend keys:

```text
breast_ha
gluteal_ha
hiko
```

Future procedures can extend the same architecture.

---

# 4. Screening

## 4.1 Common screening fields

Support structured answers for:

- active infection / inflamed skin in treatment area
- fever / currently unwell
- pregnancy
- breastfeeding
- severe allergy / anaphylaxis history
- bleeding disorder
- anticoagulant / antiplatelet use
- immunosuppression
- active autoimmune / inflammatory disease
- previous local anesthetic reaction

## 4.2 HA-specific screening

For Breast HA and Gluteal HA:

- previous reaction to HA filler
- previous filler in the treatment area
- previous unknown injectable material
- previous filler complication
- previous infection / migration / nodules / granuloma
- allergy to components of the selected product
- lidocaine allergy if product contains lidocaine

Do not hard-code all product contraindications into one generic checklist. Store generic screening and allow product-specific rules later.

## 4.3 Breast HA screening

Add:

- new or unexplained breast lump
- unexplained nipple discharge
- new nipple / skin change
- current mastitis / breast infection
- breast abnormality under investigation
- previous breast filler
- previous breast surgery
- previous implant
- history of breast cancer
- upcoming / current breast imaging
- unexplained significant breast pain

Important: some items should trigger `review`, while unresolved abnormal findings / active infection should trigger `defer`.

## 4.4 Gluteal HA screening

Add:

- previous gluteal filler
- previous PMMA / silicone / biopolymer / unknown injectable
- previous BBL / fat grafting
- previous gluteal implant
- unexplained palpable nodules
- current wound / infection
- recent procedure in the area

## 4.5 Hiko screening

Add:

- previous reaction to sutures / threads
- previous Hiko / nose thread lift
- previous thread complication
- previous nasal filler
- previous rhinoplasty
- prior nose trauma / fracture
- previous reaction to local anesthetic
- reaction to povidone-iodine / Betadine
- thread material if known:
  - PDO
  - PCL
  - PLLA
  - surgical suture
  - unknown

Do not label “iodine allergy” as a generic allergy. Store the actual preparation and reaction.

---

# 5. Screening Backend Requirements

Create a screening entity/table with at least:

```text
id
tenant_id
patient_id
procedure_key
answers JSON
flags JSON
status
reviewed_by
reviewed_at
physician_note
created_by
created_at
updated_at
```

Required behavior:

- tenant scoped
- patient scoped
- procedure scoped
- immutable audit trail or versioned revisions if the repo already has snapshot/version conventions
- physician review decision stored
- API response must include:
  - screening result
  - flags
  - whether visual planning may start

Do not allow creation of a Vision Assist plan from a screening belonging to:
- another tenant
- another patient
- another procedure
- a deferred state unless explicitly physician-overridden

---

# 6. Vision Assist Plan

Create one generic backend model capable of supporting all three procedures.

Suggested fields:

```text
id
tenant_id
patient_id
clinical_screening_id
procedure_key
status
capture_views JSON
landmarks JSON
entry_map JSON
baseline_measurements JSON
volume_plan JSON
symmetry_plan JSON
planned_changes JSON
simulation_meta JSON
created_by
updated_by
created_at
updated_at
```

Statuses:

```text
draft
ready
finalized
cancelled
```

---

# 7. Breast HA Planning

Backend must support:

## Capture views

- front
- left side
- right side
- optional oblique

## Baseline measurements

Store AI-generated or manually corrected estimates such as:

- estimated left/right size or volume
- breast base width
- breast height
- visible projection
- nipple level
- IMF level
- upper-pole fullness
- lower-pole fullness
- left/right asymmetry
- confidence / source metadata

Important:
Plain 2D photo output should be stored as `estimated`, not represented as an exact measured physical volume unless a calibrated method is used.

## Planning

Support:

- entry / access reference map
- target regions
- volume per side
- volume per region
- doctor override
- symmetry optimization

Example structure:

```json
{
  "left": {
    "total_ml": 180,
    "regions": {
      "upper_pole": 80,
      "medial": 40,
      "central": 60
    }
  },
  "right": {
    "total_ml": 140,
    "regions": {
      "upper_pole": 60,
      "medial": 30,
      "central": 50
    }
  }
}
```

## Symmetry planning

Support equal-volume vs differential-volume plans.

The backend should allow:

- baseline asymmetry metrics
- candidate symmetry-balanced plan
- doctor-selected final plan
- reason / notes
- AI model metadata
- doctor override

The AI must not be treated as the prescriber. The physician selects the final plan.

---

# 8. Gluteal HA Planning

Capture:

- posterior
- left side
- right side
- optional oblique

Support regional planning:

- hip dip
- upper pole
- superolateral contour
- central projection
- lower contour if used by clinic workflow

Store:

- left/right total volume
- region-specific volumes
- symmetry plan
- contour measurements
- entry / access reference overlay
- physician override

Example:

```json
{
  "left": {
    "hip_dip": 30,
    "upper_pole": 20,
    "superolateral": 20,
    "central": 10
  },
  "right": {
    "hip_dip": 20,
    "upper_pole": 30,
    "superolateral": 25,
    "central": 10
  }
}
```

---

# 9. Hiko Nose Lift Planning

Hiko must be separate from HA filler planning.

No volume-plan requirement.

Support selectable treatment targets:

```text
bridge
tip
alar
```

Capture views:

- front
- left side
- right side
- optional 45° oblique
- optional basal

Store landmarks such as:

- nasal midline
- radix
- dorsum
- tip
- columella
- alar rims
- alar base
- nasolabial angle
- left/right alar symmetry

Planning stores:

- selected target sites
- entry / access reference points
- treatment-axis overlay
- possible outcome simulation metadata
- physician-confirmed plan

Do not expose backend labels such as “safe entry point.”
Use neutral terms such as:

```text
entry_reference
access_reference
surface_planning_reference
```

---

# 10. Vision Assist Assets

Create a dedicated generated-asset registry.

Suggested fields:

```text
id
vision_assist_plan_id
asset_type
view_key
disk
path
mime_type
metadata JSON
created_by
created_at
updated_at
```

Supported asset types:

```text
original
overlay
possible_outcome
before_after_generated
```

Metadata may include:

- model name/version
- prompt/config hash
- source asset ids
- generation timestamp
- doctor-selected plan revision
- procedure key
- view
- disclaimer state

Generated images must remain linked to the patient and plan in the EMR.

---

# 11. Patient / TV Presentation Backend

The TV must never receive the CMS itself.

Create a presentation-session backend that exposes only explicitly selected patient-facing assets.

Suggested fields:

```text
id
tenant_id
vision_assist_plan_id
public_token_hash
current_asset_id
patient_display_name
title
subtitle
plan_summary JSON
disclaimer
status
expires_at
created_by
created_at
updated_at
```

Presentation states:

```text
active
stopped
expired
```

Required behavior:

- short-lived presentation token
- token stored hashed
- tenant-scoped creation/update
- presentation endpoint exposes only:
  - patient display name
  - title
  - subtitle
  - selected image URL
  - asset type
  - view
  - plan summary
  - disclaimer
  - expiry
- no patient chart data
- no screening answers
- no internal notes
- no CMS routes
- no private AI analysis

Patient-facing image metadata should support:

```text
Patient: Maria Santos
Breast HA Enhancement
Symmetry-Optimized Possible Outcome

Before | After-Generated

Left 180 cc
Right 140 cc

Illustrative possible outcome only.
Actual results may vary.
```

Presentation sessions should expire automatically, e.g. 4 hours or configurable shorter duration.

Use signed / temporary URLs for private image storage.

---

# 12. SMS Gateway Architecture

The Android phone is only a gateway device.

PAPPS / Alimango One server remains the source of business logic.

Two separate backend scopes are required.

---

# 13. Landlord SMS Gateway Configuration

Landlord manages platform-level configuration.

Create platform settings for:

```text
enabled
apk_url
minimum_app_version
pairing_ttl_seconds
gateway_base_url
platform_secret
metadata
updated_by
timestamps
```

The platform secret must be encrypted at rest using the project’s existing secret-encryption convention.

Landlord endpoints should support:

- read settings
- update settings
- rotate platform secret
- list paired devices across tenants
- inspect:
  - tenant
  - device name
  - phone number
  - status
  - last seen
  - app version
  - battery percentage if supplied
  - revoked state

Do not expose raw secrets in list responses.

---

# 14. Tenant SMS Gateway

Tenant setup must require no API key entry by the tenant.

Tenant endpoint should:

1. create a short-lived pairing token
2. generate QR payload / deep-link payload
3. return:
   - pairing id
   - QR payload
   - expiry
   - APK download URL
   - minimum app version

Recommended deep-link format:

```text
alimango://sms-gateway/pair?payload=<encoded-payload>
```

Payload should contain only what the Android client needs to pair securely.

Pairing token:
- one-time use
- short expiry
- stored hashed
- tenant scoped
- invalidated after consumption

---

# 15. Android Gateway Device Enrollment

Device pairing endpoint should accept:

```text
pairing_id
pairing_secret
device_uuid
device_name
phone_number
app_version
metadata
```

After successful pairing:

- invalidate pairing token
- generate unique device credential
- return credential only once
- store only credential hash
- bind device to tenant
- mark device online
- record paired_by
- record last_seen_at

A repeated device UUID should not silently cross tenants.

---

# 16. Device Authentication

Gateway device API calls should require:

```text
X-Alimango-Device: <device_uuid>
Authorization: Bearer <device_credential>
```

Middleware should:

- locate active device
- reject revoked device
- compare credential hash using constant-time comparison
- inject authenticated gateway device into request context

---

# 17. Heartbeat

Android device heartbeat endpoint should accept:

```text
battery_percent
app_version
phone_number
metadata
```

Server updates:

```text
status = online
last_seen_at
battery_percent
app_version
phone_number
metadata
```

Tenant UI can later infer offline status when heartbeat is stale.

---

# 18. Revoke / Replace Device

Tenant must be able to revoke a device.

Store:

```text
status = revoked
revoked_at
```

Revoked credentials must stop working immediately.

Do not hard-delete device history unless required by existing project conventions.

---

# 19. Suggested Tables

Implement equivalent tables using repository naming conventions.

## clinical_screenings
## vision_assist_plans
## vision_assist_assets
## vision_presentations
## sms_gateway_platform_settings
## sms_gateway_pairing_tokens
## sms_gateway_devices

Add proper indexes for:

- tenant_id
- patient_id
- procedure_key
- status
- expires_at
- last_seen_at
- public_id / token hash
- device_uuid
- revoked_at

Use existing FK conventions if repository schema permits it.

---

# 20. Suggested API Routes

Adapt route prefixes to current project conventions.

## Doctor's Corner

```text
POST   /api/doctor-corner/screenings
POST   /api/doctor-corner/screenings/{id}/review

POST   /api/doctor-corner/vision-plans
GET    /api/doctor-corner/vision-plans/{id}
PATCH  /api/doctor-corner/vision-plans/{id}

POST   /api/doctor-corner/vision-plans/{id}/assets

POST   /api/doctor-corner/vision-plans/{id}/presentations
PATCH  /api/doctor-corner/presentations/{id}

GET    /api/doctor-corner/presentation/{token}
```

## Landlord SMS Gateway

```text
GET    /api/landlord/sms-gateway/settings
PUT    /api/landlord/sms-gateway/settings
POST   /api/landlord/sms-gateway/rotate-secret
GET    /api/landlord/sms-gateway/devices
```

## Tenant SMS Gateway

```text
GET    /api/tenant/sms-gateway
POST   /api/tenant/sms-gateway/pairing
DELETE /api/tenant/sms-gateway/devices/{id}
```

## Android Gateway

```text
POST   /api/gateway/v1/pair
POST   /api/gateway/v1/heartbeat
```

---

# 21. Authorization

Use existing PAPPS authorization patterns.

Requirements:

- all Doctor's Corner data tenant scoped
- all patient records tenant scoped
- clinical routes require appropriate staff/doctor access
- Landlord SMS routes require Landlord access
- Tenant SMS routes require tenant owner/general access according to existing access model
- gateway pairing endpoint is unauthenticated but protected by:
  - single-use pairing token
  - expiry
  - rate limiting
- gateway heartbeat uses device credentials

Do not invent a second tenant identity system.

---

# 22. Auditability

Record at minimum:

- who created screening
- who reviewed screening
- who created/updated Vision Assist plan
- which plan revision generated a simulation
- who started presentation
- SMS device pairing actor
- secret rotation actor
- device revocation actor

If the repo already has an audit/event system, use it.

---

# 23. Validation Rules

At minimum:

- Hiko must reject `volume_plan`
- HA plans can accept `volume_plan`
- Vision plan must match screening patient
- Vision plan must match screening procedure
- deferred screening blocks planning
- presentation asset must belong to the selected plan
- presentation plan must belong to current tenant
- SMS pairing token must:
  - exist
  - not be consumed
  - not be expired
  - match hashed secret
- device credential must be hashed
- revoked devices cannot authenticate

---

# 24. Tests

Add feature/unit tests covering:

## Screening
- clear screening
- review screening
- defer screening
- breast-specific defer
- gluteal prior injectable review
- Hiko suture reaction review
- tenant isolation
- patient/procedure mismatch

## Vision Assist
- deferred screening blocks plan creation
- Hiko rejects volume plan
- breast/gluteal volume plan accepted
- symmetry plan persists
- asset must belong to plan
- tenant isolation

## Presentation
- only selected asset exposed
- no EMR/screening fields exposed
- expired token fails
- stopped session fails
- token cannot access another plan

## SMS Gateway
- Landlord settings authorization
- pairing token creation
- pairing token expiry
- one-time token enforcement
- invalid pairing secret rejected
- device credential returned only at pairing
- heartbeat succeeds with valid device auth
- heartbeat fails for revoked device
- tenant cannot revoke another tenant's device
- landlord device listing works

---

# 25. Non-Goals for This Patch

Do not implement frontend in this patch.

Do not implement:

- font unification
- shared tabs
- shared window/card components
- TV browser UI
- QR rendering UI
- Android app changes
- actual SMS sending queue if it does not already exist
- PubMed frontend
- image generation model integration if not already available
- deployment

Backend APIs and data model only.

---

# 26. Integration Rules

Before writing code:

1. inspect current PAPPS repository
2. identify:
   - tenant resolver
   - membership authorization
   - patient model / PK type
   - user model
   - API route registration
   - storage abstraction
   - audit/event conventions
   - encryption conventions
3. adapt this specification to those existing patterns
4. do not create duplicate concepts
5. keep changes additive and isolated
6. run relevant tests
7. create a branch / patch only
8. do not deploy

---

# 27. Expected Agent Output

The coding agent should return:

```text
Implemented:
- migrations
- models
- controllers
- services
- middleware
- routes
- tests

Validation:
- tests run
- passing / inherited failures clearly separated

Git:
- branch name
- commit SHA
- PR URL if requested

Deployment:
- NOT PERFORMED
```

---

# 28. Final Functional Goal

Doctor's Corner backend should support:

```text
Screen patient
↓
Doctor review if needed
↓
Create procedure-specific visual plan
↓
Store anatomy / mapping / volume / symmetry data
↓
Register generated original / overlay / possible-outcome images
↓
Present only selected patient-facing image on TV
↓
Save plan and generated visual references to the patient record
```

SMS Gateway backend should support:

```text
Landlord configures gateway platform
↓
Tenant clicks Connect Phone
↓
Server creates secure pairing payload
↓
Android scans QR
↓
Device pairs to tenant
↓
Device authenticates with its own credential
↓
Server tracks heartbeat/status
↓
Tenant can replace/revoke device
```

This patch is backend-only and must be built against the current PAPPS repository conventions.
