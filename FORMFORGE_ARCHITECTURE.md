I now have a thorough understanding of the ShowPass codebase patterns. Let me compile the complete FormForge architecture document.

---

# FormForge -- Form Builder & Submission API: Complete Architecture Document

---

## 1. Application Concept

### What is FormForge?

FormForge is a backend-only REST API for building, publishing, and collecting responses to dynamic forms. It serves as the engine behind any form-based data collection workflow: customer feedback surveys, job applications, event registration forms, product order forms, support ticket intake, or internal HR questionnaires. The API allows users to create form definitions with typed fields and validation rules, publish those forms to accept submissions, collect and validate responses, and analyze results through aggregation endpoints.

FormForge is designed around workspaces, which are team-based containers for forms. A workspace might represent a department, a project, or an entire company. Users can be owners or members of workspaces, and forms belong to workspaces. This mirrors how real SaaS products like Typeform, JotForm, and Google Forms operate in team contexts.

The core workflow is straightforward: a user creates a workspace, defines a form with fields (text, email, number, dropdown, rating, etc.), sets validation rules (required, min/max, regex patterns, conditional visibility), publishes the form, and then collects submissions. Each submission passes through a validation engine that checks every field response against its rules. Webhooks can be configured to notify external systems when submissions arrive. Rate limiting prevents spam. Caching accelerates form definition retrieval.

### Who are the users?

There are three user roles:

1. **admin** -- Platform administrators who can manage all workspaces and forms, view analytics, and configure system-wide settings.
2. **editor** -- Workspace members who can create and manage forms within their workspaces, review submissions, and configure webhooks.
3. **viewer** -- Workspace members with read-only access to forms and submissions.

Submitters (the people who fill out forms) do not need accounts. Submissions are made via a public endpoint that only requires the form ID. This mirrors real-world behavior where a company creates a form and shares a link with anonymous respondents.

### Comparison to Google Forms / Typeform / JotForm

Like those products, FormForge supports:
- Multiple field types with validation
- Form lifecycle management (draft, published, closed)
- Submission collection and storage
- Basic analytics/aggregation

Unlike consumer form builders, FormForge focuses on the backend patterns that make production form systems work:
- Webhook delivery with HMAC signature verification for secure event notification
- Idempotency keys to prevent duplicate submissions (e.g., double-click on submit)
- Rate limiting per form to prevent spam attacks
- A validation engine that processes conditional logic ("show field B only if field A = Yes")
- Cache invalidation when form definitions change
- Soft-delete to prevent data leakage from archived forms
- Form versioning so submissions reference the form definition at the time of submission

### Why is this a good assessment domain?

Forms are universally understood. Every developer has filled out an online form. The domain requires no specialized knowledge (unlike financial trading, healthcare, or logistics). The mental model is simple: forms have fields, fields have types and rules, submissions contain answers to those fields.

Yet beneath this simple surface, a form builder exercises a rich set of backend patterns:
- **Validation engines** require processing a list of rules in order, handling conditional logic, and returning structured error messages
- **Webhook delivery** requires HTTP client usage, retry logic, signature generation, and delivery logging
- **Idempotency** requires understanding of unique keys, race conditions, and duplicate detection
- **Pagination and filtering** requires proper query construction, cursor vs offset pagination, and sorting
- **State machines** require understanding of allowed transitions and enforcement at the data layer
- **Caching** requires understanding of cache keys, TTL, and invalidation strategies
- **Rate limiting** requires Redis-based counters and time windows

---

## 2. Data Models

### 2.1 User

```javascript
// src/models/User.js
import mongoose from 'mongoose';
import bcrypt from 'bcryptjs';
import { softDeletePlugin } from '../utils/softDelete.plugin.js';

export const USER_ROLES = ['admin', 'editor', 'viewer'];

const userSchema = new mongoose.Schema(
  {
    name: {
      type: String,
      required: true,
      maxlength: 100,
      trim: true,
    },
    email: {
      type: String,
      required: true,
      unique: true,
      lowercase: true,
      trim: true,
    },
    password: {
      type: String,
      required: true,
      minlength: 8,
    },
    role: {
      type: String,
      required: true,
      enum: USER_ROLES,
      default: 'editor',
    },
  },
  {
    timestamps: { createdAt: 'created_at', updatedAt: 'updated_at' },
  }
);

userSchema.pre('save', async function () {
  if (!this.isModified('password')) return;
  this.password = await bcrypt.hash(this.password, 12);
});

userSchema.methods.comparePassword = async function (candidatePassword) {
  return bcrypt.compare(candidatePassword, this.password);
};

userSchema.plugin(softDeletePlugin);

export const User = mongoose.model('User', userSchema);
```

**Fields:**
| Field | Type | Required | Default | Validation | Index |
|-------|------|----------|---------|------------|-------|
| name | String | Yes | - | maxlength: 100, trim | - |
| email | String | Yes | - | unique, lowercase, trim | unique |
| password | String | Yes | - | minlength: 8 | - |
| role | String | Yes | 'editor' | enum: admin, editor, viewer | - |
| deleted_at | Date | No | null | (from plugin) | - |
| created_at | Date | Auto | - | - | - |
| updated_at | Date | Auto | - | - | - |

**Relationships:** Users belong to workspaces via WorkspaceMember (many-to-many).

---

### 2.2 Workspace

```javascript
// src/models/Workspace.js
import mongoose from 'mongoose';
import { softDeletePlugin } from '../utils/softDelete.plugin.js';

const workspaceSchema = new mongoose.Schema(
  {
    name: {
      type: String,
      required: true,
      maxlength: 200,
      trim: true,
    },
    slug: {
      type: String,
      required: true,
      unique: true,
      lowercase: true,
      trim: true,
      match: /^[a-z0-9-]+$/,
    },
    owner_id: {
      type: mongoose.Schema.Types.ObjectId,
      ref: 'User',
      required: true,
    },
    description: {
      type: String,
      maxlength: 500,
      default: '',
    },
  },
  {
    timestamps: { createdAt: 'created_at', updatedAt: 'updated_at' },
  }
);

workspaceSchema.index({ owner_id: 1 });
workspaceSchema.index({ slug: 1 }, { unique: true });

workspaceSchema.plugin(softDeletePlugin);

export const Workspace = mongoose.model('Workspace', workspaceSchema);
```

**Fields:**
| Field | Type | Required | Default | Validation | Index |
|-------|------|----------|---------|------------|-------|
| name | String | Yes | - | maxlength: 200, trim | - |
| slug | String | Yes | - | unique, lowercase, match: /^[a-z0-9-]+$/ | unique |
| owner_id | ObjectId | Yes | - | ref: User | Yes |
| description | String | No | '' | maxlength: 500 | - |
| deleted_at | Date | No | null | (from plugin) | - |

**Relationships:** Has many Forms. Has many WorkspaceMembers. Owned by one User.

---

### 2.3 WorkspaceMember (embedded lookup, separate collection)

```javascript
// src/models/WorkspaceMember.js
import mongoose from 'mongoose';

export const MEMBER_ROLES = ['owner', 'editor', 'viewer'];

const workspaceMemberSchema = new mongoose.Schema(
  {
    workspace_id: {
      type: mongoose.Schema.Types.ObjectId,
      ref: 'Workspace',
      required: true,
    },
    user_id: {
      type: mongoose.Schema.Types.ObjectId,
      ref: 'User',
      required: true,
    },
    role: {
      type: String,
      required: true,
      enum: MEMBER_ROLES,
      default: 'editor',
    },
  },
  {
    timestamps: { createdAt: 'created_at', updatedAt: 'updated_at' },
  }
);

workspaceMemberSchema.index({ workspace_id: 1, user_id: 1 }, { unique: true });
workspaceMemberSchema.index({ user_id: 1 });

export const WorkspaceMember = mongoose.model('WorkspaceMember', workspaceMemberSchema);
```

**Note:** This model is NOT a task target. It exists to support workspace access control and is fully implemented in the skeleton code.

---

### 2.4 Form

```javascript
// src/models/Form.js
import mongoose from 'mongoose';
import { softDeletePlugin } from '../utils/softDelete.plugin.js';

export const FormStatus = {
  DRAFT: 'draft',
  PUBLISHED: 'published',
  CLOSED: 'closed',
  ARCHIVED: 'archived',
};

export const FORM_STATUSES = Object.values(FormStatus);

export const VALID_FORM_TRANSITIONS = {
  [FormStatus.DRAFT]: [FormStatus.PUBLISHED],
  [FormStatus.PUBLISHED]: [FormStatus.CLOSED],
  [FormStatus.CLOSED]: [FormStatus.PUBLISHED, FormStatus.ARCHIVED],
  [FormStatus.ARCHIVED]: [],
};

const formSchema = new mongoose.Schema(
  {
    title: {
      type: String,
      required: true,
      maxlength: 300,
      trim: true,
    },
    description: {
      type: String,
      maxlength: 2000,
      default: '',
    },
    workspace_id: {
      type: mongoose.Schema.Types.ObjectId,
      ref: 'Workspace',
      required: true,
    },
    creator_id: {
      type: mongoose.Schema.Types.ObjectId,
      ref: 'User',
      required: true,
    },
    status: {
      type: String,
      required: true,
      enum: FORM_STATUSES,
      default: 'draft',
    },
    version: {
      type: Number,
      default: 1,
      min: 1,
    },
    settings: {
      max_submissions: {
        type: Number,
        default: null,   // null = unlimited
        min: 1,
      },
      closes_at: {
        type: Date,
        default: null,   // null = no auto-close
      },
      requires_auth: {
        type: Boolean,
        default: false,
      },
      one_submission_per_user: {
        type: Boolean,
        default: false,
      },
    },
    submission_count: {
      type: Number,
      default: 0,
      min: 0,
    },
    fields: [
      {
        field_id: {
          type: String,
          required: true,
        },
        label: {
          type: String,
          required: true,
          maxlength: 200,
          trim: true,
        },
        type: {
          type: String,
          required: true,
          enum: [
            'text', 'textarea', 'number', 'email',
            'dropdown', 'multi_select', 'rating',
            'date', 'yes_no',
          ],
        },
        required: {
          type: Boolean,
          default: false,
        },
        placeholder: {
          type: String,
          maxlength: 200,
          default: '',
        },
        options: [String],          // for dropdown / multi_select
        validation: {
          min_length: Number,        // text, textarea
          max_length: Number,        // text, textarea
          min_value: Number,          // number
          max_value: Number,          // number
          pattern: String,            // regex for text
          min_selections: Number,     // multi_select
          max_selections: Number,     // multi_select
        },
        conditional: {
          depends_on: String,         // field_id of the dependency
          show_when: String,          // value that triggers visibility
        },
        position: {
          type: Number,
          required: true,
          min: 0,
        },
      },
    ],
  },
  {
    timestamps: { createdAt: 'created_at', updatedAt: 'updated_at' },
  }
);

formSchema.index({ workspace_id: 1 });
formSchema.index({ status: 1 });
formSchema.index({ creator_id: 1 });

formSchema.post('init', function () {
  this._original_status = this.status;
});

formSchema.pre('save', function () {
  // State machine: validate status transitions for existing documents
  if (!this.isNew && this.isModified('status')) {
    const original = this._original_status;
    const allowed = VALID_FORM_TRANSITIONS[original];
    if (!allowed || !allowed.includes(this.status)) {
      throw new Error(`Invalid status transition: '${original}' â†’ '${this.status}'`);
    }
  }
});

formSchema.plugin(softDeletePlugin);

export const Form = mongoose.model('Form', formSchema);
```

**Design decision:** Fields are embedded in the Form document rather than stored in a separate collection. This is because fields are always read together with the form, form definitions are typically small (under 100 fields), and embedding avoids N+1 query problems. The `field_id` is a string (e.g., `"field_abc123"`) generated on creation, used as a stable identifier in submissions and conditional logic.

**Fields:**
| Field | Type | Required | Default | Validation | Index |
|-------|------|----------|---------|------------|-------|
| title | String | Yes | - | maxlength: 300 | - |
| description | String | No | '' | maxlength: 2000 | - |
| workspace_id | ObjectId | Yes | - | ref: Workspace | Yes |
| creator_id | ObjectId | Yes | - | ref: User | Yes |
| status | String | Yes | 'draft' | enum: draft, published, closed, archived | Yes |
| version | Number | No | 1 | min: 1 | - |
| settings.max_submissions | Number | No | null | min: 1 | - |
| settings.closes_at | Date | No | null | - | - |
| settings.requires_auth | Boolean | No | false | - | - |
| settings.one_submission_per_user | Boolean | No | false | - | - |
| submission_count | Number | No | 0 | min: 0 | - |
| fields | Array | No | [] | (embedded subdocument array) | - |
| deleted_at | Date | No | null | (from plugin) | - |

---

### 2.5 Submission

```javascript
// src/models/Submission.js
import mongoose from 'mongoose';
import { softDeletePlugin } from '../utils/softDelete.plugin.js';

export const SubmissionStatus = {
  PENDING: 'pending',
  VALIDATED: 'validated',
  ACCEPTED: 'accepted',
  REJECTED: 'rejected',
};

export const SUBMISSION_STATUSES = Object.values(SubmissionStatus);

const fieldResponseSchema = new mongoose.Schema(
  {
    field_id: {
      type: String,
      required: true,
    },
    value: {
      type: mongoose.Schema.Types.Mixed,
      default: null,
    },
  },
  { _id: false }
);

const submissionSchema = new mongoose.Schema(
  {
    form_id: {
      type: mongoose.Schema.Types.ObjectId,
      ref: 'Form',
      required: true,
    },
    form_version: {
      type: Number,
      required: true,
      min: 1,
    },
    respondent_email: {
      type: String,
      default: null,
      lowercase: true,
      trim: true,
    },
    respondent_user_id: {
      type: mongoose.Schema.Types.ObjectId,
      ref: 'User',
      default: null,
    },
    responses: [fieldResponseSchema],
    status: {
      type: String,
      required: true,
      enum: SUBMISSION_STATUSES,
      default: 'pending',
    },
    validation_errors: [
      {
        field_id: String,
        message: String,
      },
    ],
    metadata: {
      ip_address: String,
      user_agent: String,
      submitted_at: {
        type: Date,
        default: Date.now,
      },
    },
    idempotency_key: {
      type: String,
      required: true,
      unique: true,
    },
  },
  {
    timestamps: { createdAt: 'created_at', updatedAt: 'updated_at' },
  }
);

submissionSchema.index({ form_id: 1, created_at: -1 });
submissionSchema.index({ form_id: 1, status: 1 });
submissionSchema.index({ idempotency_key: 1 }, { unique: true });
submissionSchema.index({ respondent_email: 1, form_id: 1 });

submissionSchema.plugin(softDeletePlugin);

export const Submission = mongoose.model('Submission', submissionSchema);
```

**Design decision:** FieldResponse is embedded in Submission as a subdocument array. Each entry has `field_id` (matching the form's field_id) and a `value` field of Mixed type. This is necessary because different field types produce different value shapes: strings, numbers, arrays, booleans.

**Fields:**
| Field | Type | Required | Default | Validation | Index |
|-------|------|----------|---------|------------|-------|
| form_id | ObjectId | Yes | - | ref: Form | Compound |
| form_version | Number | Yes | - | min: 1 | - |
| respondent_email | String | No | null | lowercase, trim | Compound |
| respondent_user_id | ObjectId | No | null | ref: User | - |
| responses | Array | No | [] | embedded fieldResponseSchema | - |
| status | String | Yes | 'pending' | enum: pending, validated, accepted, rejected | Compound |
| validation_errors | Array | No | [] | - | - |
| metadata.ip_address | String | No | - | - | - |
| metadata.user_agent | String | No | - | - | - |
| metadata.submitted_at | Date | No | Date.now | - | - |
| idempotency_key | String | Yes | - | unique | unique |
| deleted_at | Date | No | null | (from plugin) | - |

---

### 2.6 WebhookEndpoint

```javascript
// src/models/WebhookEndpoint.js
import mongoose from 'mongoose';
import { softDeletePlugin } from '../utils/softDelete.plugin.js';

export const WEBHOOK_EVENTS = [
  'submission.created',
  'submission.validated',
  'submission.accepted',
  'submission.rejected',
  'form.published',
  'form.closed',
];

const webhookEndpointSchema = new mongoose.Schema(
  {
    form_id: {
      type: mongoose.Schema.Types.ObjectId,
      ref: 'Form',
      required: true,
    },
    url: {
      type: String,
      required: true,
      maxlength: 2000,
    },
    secret: {
      type: String,
      required: true,
    },
    events: [
      {
        type: String,
        enum: WEBHOOK_EVENTS,
      },
    ],
    active: {
      type: Boolean,
      default: true,
    },
    description: {
      type: String,
      maxlength: 200,
      default: '',
    },
  },
  {
    timestamps: { createdAt: 'created_at', updatedAt: 'updated_at' },
  }
);

webhookEndpointSchema.index({ form_id: 1, active: 1 });

webhookEndpointSchema.plugin(softDeletePlugin);

export const WebhookEndpoint = mongoose.model('WebhookEndpoint', webhookEndpointSchema);
```

---

### 2.7 WebhookDelivery

```javascript
// src/models/WebhookDelivery.js
import mongoose from 'mongoose';

export const DeliveryStatus = {
  PENDING: 'pending',
  SUCCESS: 'success',
  FAILED: 'failed',
};

const webhookDeliverySchema = new mongoose.Schema(
  {
    webhook_id: {
      type: mongoose.Schema.Types.ObjectId,
      ref: 'WebhookEndpoint',
      required: true,
    },
    event_type: {
      type: String,
      required: true,
    },
    payload: {
      type: mongoose.Schema.Types.Mixed,
      required: true,
    },
    status: {
      type: String,
      required: true,
      enum: Object.values(DeliveryStatus),
      default: 'pending',
    },
    attempt_count: {
      type: Number,
      default: 0,
    },
    max_attempts: {
      type: Number,
      default: 3,
    },
    response_status: {
      type: Number,
      default: null,
    },
    response_body: {
      type: String,
      default: null,
      maxlength: 5000,
    },
    error_message: {
      type: String,
      default: null,
    },
    next_retry_at: {
      type: Date,
      default: null,
    },
    delivered_at: {
      type: Date,
      default: null,
    },
    signature: {
      type: String,
      required: true,
    },
  },
  {
    timestamps: { createdAt: 'created_at', updatedAt: 'updated_at' },
  }
);

webhookDeliverySchema.index({ webhook_id: 1 });
webhookDeliverySchema.index({ status: 1, next_retry_at: 1 });

export const WebhookDelivery = mongoose.model('WebhookDelivery', webhookDeliverySchema);
```

---

### 2.8 SubmissionQuota

```javascript
// src/models/SubmissionQuota.js
import mongoose from 'mongoose';

const submissionQuotaSchema = new mongoose.Schema(
  {
    form_id: {
      type: mongoose.Schema.Types.ObjectId,
      ref: 'Form',
      required: true,
    },
    window_start: {
      type: Date,
      required: true,
    },
    window_end: {
      type: Date,
      required: true,
    },
    count: {
      type: Number,
      default: 0,
    },
    max_per_window: {
      type: Number,
      default: 100,
    },
  },
  {
    timestamps: { createdAt: 'created_at', updatedAt: 'updated_at' },
  }
);

submissionQuotaSchema.index({ form_id: 1, window_start: 1 }, { unique: true });

export const SubmissionQuota = mongoose.model('SubmissionQuota', submissionQuotaSchema);
```

**Note:** SubmissionQuota is primarily managed via Redis for real-time rate limiting. This MongoDB model serves as a persistent log of quota windows, used by the analytics endpoint. The actual rate limit check uses Redis `INCR` with TTL, identical to the ShowPass rate limiter pattern.

---

## 3. API Endpoints

### 3.1 Auth Endpoints

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| POST | `/api/v1/auth/register` | No | Register a new user |
| POST | `/api/v1/auth/login` | No | Login and get JWT |
| GET | `/api/v1/auth/me` | Yes | Get current user profile |

**POST /api/v1/auth/register**
- Body: `{ name, email, password, role? }`
- Response: `201 { user: { _id, name, email, role, created_at } }`
- Errors: 400 (missing fields, invalid email, short password), 400 (email already exists)

**POST /api/v1/auth/login**
- Body: `{ email, password }`
- Response: `200 { token, user: { _id, name, email, role } }`
- Errors: 400 (missing fields), 401 (invalid credentials)

**GET /api/v1/auth/me**
- Headers: `Authorization: Bearer <token>`
- Response: `200 { user: { _id, name, email, role, created_at, updated_at } }`

---

### 3.2 Workspace Endpoints

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| POST | `/api/v1/workspaces` | Yes | Create a workspace |
| GET | `/api/v1/workspaces` | Yes | List user's workspaces |
| GET | `/api/v1/workspaces/:id` | Yes | Get workspace by ID |
| PATCH | `/api/v1/workspaces/:id` | Yes | Update workspace |
| DELETE | `/api/v1/workspaces/:id` | Yes | Soft-delete workspace |
| POST | `/api/v1/workspaces/:id/members` | Yes | Add member |
| GET | `/api/v1/workspaces/:id/members` | Yes | List members |
| DELETE | `/api/v1/workspaces/:id/members/:userId` | Yes | Remove member |

**POST /api/v1/workspaces**
- Body: `{ name, slug, description? }`
- Response: `201 { workspace }`
- Logic: Creates workspace with owner_id = req.user._id. Creates WorkspaceMember with role "owner".
- Errors: 400 (missing name/slug), 400 (slug already exists), 400 (invalid slug format)

**GET /api/v1/workspaces**
- Query: `page, limit`
- Response: `200 { workspaces, pagination }`
- Logic: Returns workspaces where user is a member. Pagination with page/limit.

---

### 3.3 Form Endpoints

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| POST | `/api/v1/forms` | Yes | Create a form |
| GET | `/api/v1/forms` | Yes | List forms (with filters) |
| GET | `/api/v1/forms/:id` | Yes | Get form by ID |
| PATCH | `/api/v1/forms/:id` | Yes | Update form (title, desc, settings) |
| PATCH | `/api/v1/forms/:id/status` | Yes | Transition form status |
| PUT | `/api/v1/forms/:id/fields` | Yes | Replace all fields |
| DELETE | `/api/v1/forms/:id` | Yes | Soft-delete form |
| GET | `/api/v1/forms/public/:id` | No | Get published form (public) |

**POST /api/v1/forms**
- Body: `{ title, description?, workspace_id, settings?, fields? }`
- Response: `201 { form }`
- Logic: Validates workspace membership. Creates form in draft status. Generates field_ids for any provided fields.
- Errors: 400 (missing title/workspace_id), 404 (workspace not found), 403 (not a member)

**GET /api/v1/forms**
- Query: `workspace_id, status, page, limit`
- Response: `200 { forms, pagination }`
- Logic: Returns forms from workspaces the user belongs to. Supports status and workspace_id filter. Uses Redis cache with key `forms:list:<hash>`. Pagination with page/limit/totalPages/hasNextPage/hasPrevPage.
- Cache key: `forms:list:${JSON.stringify(filters)}`

**PATCH /api/v1/forms/:id/status**
- Body: `{ status }`
- Response: `200 { form }`
- Logic: Enforces state machine transitions. Cannot publish without at least one field. Cannot publish without at least one required field. Increments version when transitioning from draft to published or from closed back to published.
- Errors: 400 (invalid transition), 400 (no fields), 404 (not found)

**PUT /api/v1/forms/:id/fields**
- Body: `{ fields: [...] }`
- Response: `200 { form }`
- Logic: Replaces all fields. Only allowed in draft status. Validates field types, options for dropdown/multi_select, conditional references. Assigns field_ids. Invalidates cache.
- Errors: 400 (form not in draft), 400 (invalid field definitions), 400 (conditional depends_on references nonexistent field)

**GET /api/v1/forms/public/:id**
- No auth required
- Response: `200 { form: { _id, title, description, fields, settings, version, status } }`
- Logic: Returns only published forms. Excludes internal fields (creator_id, workspace_id). Uses Redis cache with key `forms:public:<id>`.
- Errors: 404 (not found or not published)

---

### 3.4 Submission Endpoints

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| POST | `/api/v1/forms/:formId/submissions` | No* | Submit a form response |
| GET | `/api/v1/forms/:formId/submissions` | Yes | List submissions (paginated) |
| GET | `/api/v1/forms/:formId/submissions/:id` | Yes | Get single submission |
| PATCH | `/api/v1/forms/:formId/submissions/:id/status` | Yes | Accept/reject submission |
| GET | `/api/v1/forms/:formId/analytics` | Yes | Get submission analytics |

*Auth depends on form's `settings.requires_auth`.

**POST /api/v1/forms/:formId/submissions**
- Body: `{ responses: [{ field_id, value }], respondent_email?, idempotency_key }`
- Headers: `X-Idempotency-Key: <key>` (alternative to body param)
- Response: `201 { submission }` (on success with all validations passing)
- Response: `422 { submission }` (if validation errors exist -- submission saved as 'rejected' with validation_errors populated)
- Logic:
  1. Check form exists and is published (not draft, closed, archived, or soft-deleted)
  2. Check form has not reached max_submissions
  3. Check form has not passed closes_at
  4. Check rate limit (Redis: `submission_rate:<form_id>`, 100 per 60 seconds)
  5. Check idempotency key (return existing submission if duplicate)
  6. If form requires_auth, validate JWT; if one_submission_per_user, check for existing submission
  7. Run validation engine on all responses
  8. If validation passes: status = 'validated', then auto-accept -> status = 'accepted'
  9. If validation fails: status = 'rejected', populate validation_errors
  10. Increment form.submission_count
  11. Fire webhook (submission.created event, then submission.accepted or submission.rejected)
- Errors: 404 (form not found/not published), 409 (idempotency -- duplicate), 429 (rate limited), 400 (missing idempotency_key)

**GET /api/v1/forms/:formId/submissions**
- Query: `status, page, limit, sort_by (created_at), sort_order (desc/asc), respondent_email`
- Response: `200 { submissions, pagination }`
- Logic: Returns submissions for the form. Auth required -- must be workspace member. Supports filtering by status and respondent_email. Supports sorting. Pagination.
- Errors: 403 (not workspace member), 404 (form not found)

**GET /api/v1/forms/:formId/analytics**
- Response: `200 { analytics }`
- Logic: Aggregates submission data. Returns:
  - `total_submissions`: count of all submissions (excluding soft-deleted)
  - `status_breakdown`: { pending, validated, accepted, rejected }
  - `field_summaries`: per-field aggregation (see Field Types Reference section 11)
  - `submission_rate`: submissions per day over last 30 days
  - `completion_rate`: percentage of accepted vs total
- Errors: 403 (not workspace member), 404 (form not found)

---

### 3.5 Webhook Endpoints

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| POST | `/api/v1/forms/:formId/webhooks` | Yes | Register webhook endpoint |
| GET | `/api/v1/forms/:formId/webhooks` | Yes | List webhook endpoints |
| GET | `/api/v1/forms/:formId/webhooks/:id` | Yes | Get webhook details |
| PATCH | `/api/v1/forms/:formId/webhooks/:id` | Yes | Update webhook |
| DELETE | `/api/v1/forms/:formId/webhooks/:id` | Yes | Soft-delete webhook |
| GET | `/api/v1/forms/:formId/webhooks/:id/deliveries` | Yes | List deliveries |
| POST | `/api/v1/forms/:formId/webhooks/:id/test` | Yes | Send test delivery |

**POST /api/v1/forms/:formId/webhooks**
- Body: `{ url, secret, events: ['submission.created', ...], description? }`
- Response: `201 { webhook }`
- Logic: Validates URL format (must be https:// or http://localhost for dev). Validates events array against WEBHOOK_EVENTS. Generates webhook secret if not provided.
- Errors: 400 (invalid URL), 400 (invalid events), 404 (form not found)

---

## 4. Project Structure

```
formforge/
  config.json                      # Mocha reporter config
  hackerrank.yml                   # HackerRank platform config
  Makefile                         # Build/run/test commands
  package.json                     # Dependencies and scripts
  setup.sh                         # Platform setup script
  setup_redis.sh                   # Redis setup script
  curl_scripts/
    HELPER.md                      # Curl script usage guide
    arguments.json                 # Runtime curl arguments
    arguments.template.json        # Template for reset
    01_register_user.sh            # Auth: register user
    02_login.sh                    # Auth: login
    03_create_workspace.sh         # Create workspace
    04_create_form.sh              # Create form with fields
    05_update_fields.sh            # Update form fields
    06_publish_form.sh             # Publish form
    07_submit_form.sh              # Submit response
    08_list_submissions.sh         # List submissions
    09_get_analytics.sh            # Get analytics
    10_create_webhook.sh           # Create webhook
    11_test_webhook.sh             # Test webhook delivery
    12_close_form.sh               # Close form
  documents/
    ARCHITECTURE.md                # Architecture overview for candidates
  problem-statements/
    form_lifecycle.md              # Task 1 problem statement
    form_lifecycle.html            # HTML version
    webhook_delivery.md            # Task 2 problem statement
    webhook_delivery.html
    submission_idempotency.md      # Task 3 problem statement
    submission_idempotency.html
    submission_listing.md          # Task 4 problem statement
    submission_listing.html
    validation_engine.md           # Task 5 problem statement
    validation_engine.html
    cache_invalidation.md          # Task 6 problem statement
    cache_invalidation.html
    rate_limiting.md               # Task 7 problem statement
    rate_limiting.html
    soft_delete_leak.md            # Task 8 problem statement
    soft_delete_leak.html
    field_type_validation.md       # Task 9 problem statement
    field_type_validation.html
  technical-specs/
    form_lifecycle.md              # Detailed spec for task 1
    webhook_delivery.md
    submission_idempotency.md
    submission_listing.md
    validation_engine.md
    cache_invalidation.md
    rate_limiting.md
    soft_delete_leak.md
    field_type_validation.md
  src/
    app.js                         # Express app setup (middleware, routes, swagger)
    server.js                      # Server entry point (DB + Redis connect)
    seed.js                        # Seed database with sample data
    reset.js                       # Reset DB and Redis
    config/
      db.js                        # MongoDB connection
      env.js                       # Environment variables
      redis.js                     # Redis client
      swagger.js                   # Swagger/OpenAPI spec
    controllers/
      auth.controller.js           # Auth endpoints
      workspace.controller.js      # Workspace CRUD
      form.controller.js           # Form CRUD + status transitions
      submission.controller.js     # Submission creation + listing
      webhook.controller.js        # Webhook management
      analytics.controller.js      # Analytics aggregation
    middleware/
      auth.js                      # JWT verification middleware
      authorize.js                 # Role-based access control
      errorHandler.js              # Global error handler
      rateLimiter.js               # Global rate limiter (IP-based)
      sanitize.js                  # NoSQL injection prevention
      workspaceAccess.js           # Workspace membership check
    models/
      User.js                      # User model
      Workspace.js                 # Workspace model
      WorkspaceMember.js           # Workspace membership model
      Form.js                      # Form model with embedded fields
      Submission.js                # Submission model with embedded responses
      WebhookEndpoint.js           # Webhook endpoint config
      WebhookDelivery.js           # Webhook delivery log
      SubmissionQuota.js           # Rate limit quota tracking
    routes/
      index.js                     # Route aggregator
      auth.routes.js               # Auth routes
      workspace.routes.js          # Workspace routes
      form.routes.js               # Form routes
      submission.routes.js         # Submission routes
      webhook.routes.js            # Webhook routes
      analytics.routes.js          # Analytics routes
    services/
      auth.service.js              # Auth business logic
      workspace.service.js         # Workspace business logic
      form.service.js              # Form CRUD + status + cache
      submission.service.js        # Submission creation + validation
      webhook.service.js           # Webhook delivery logic
      analytics.service.js         # Analytics aggregation
      cache.service.js             # Redis get/set/invalidate
      validation.service.js        # Field validation engine
    utils/
      AppError.js                  # Custom error classes
      helpers.js                   # Utility functions
      softDelete.plugin.js         # Mongoose soft-delete plugin
    pkg/
      features/
        form_lifecycle/            # Task 1: Form status state machine
          controller.js
          routes.js
          service.js
          tests/
            integration_test.js
            test_fixtures.js
        webhook_delivery/          # Task 2: Webhook delivery with HMAC
          controller.js
          routes.js
          service.js
          tests/
            integration_test.js
            test_fixtures.js
        submission_idempotency/    # Task 3: Idempotent submissions
          controller.js
          routes.js
          service.js
          tests/
            integration_test.js
            test_fixtures.js
        submission_listing/        # Task 4: Paginated submission listing
          controller.js
          routes.js
          service.js
          tests/
            integration_test.js
            test_fixtures.js
  test/
    task5/
      app.spec.js                  # Validation engine tests
    task6/
      app.spec.js                  # Cache invalidation tests
    task7/
      app.spec.js                  # Rate limiting tests
    task8/
      app.spec.js                  # Soft-delete leak tests
    task9/
      app.spec.js                  # Field type validation tests
```

Total source files: ~15 source files in `src/` (models, services, controllers, middleware, config, utils) plus 4 feature packages.

---

## 5. Status Lifecycles

### 5.1 Form Status Machine

```
                  +----[has fields]----+
                  |                    |
                  v                    |
  +---------+   +-----------+   +--------+   +----------+
  |  DRAFT  |-->| PUBLISHED |-->| CLOSED |-->| ARCHIVED |
  +---------+   +-----------+   +--------+   +----------+
                      ^              |
                      |              |
                      +-[reopen]-----+

  Terminal state: ARCHIVED (no transitions out)
  
  Allowed transitions:
    draft     -> published   (requires >= 1 field, >= 1 required field)
    published -> closed      (stops accepting submissions)
    closed    -> published   (reopen; increments version)
    closed    -> archived    (permanent archive)
    archived  -> (none)      (terminal)
```

### 5.2 Submission Status Machine

```
  +---------+   +------------+   +----------+
  | PENDING |-->| VALIDATED  |-->| ACCEPTED |
  +---------+   +------------+   +----------+
       |
       |        +------------+
       +------->| REJECTED   |
                +------------+

  Flow:
    1. Submission arrives -> status = 'pending'
    2. Validation engine runs
       a. All rules pass  -> status = 'validated' -> auto-advance -> 'accepted'
       b. Any rule fails  -> status = 'rejected' (validation_errors populated)
    3. Manual override: workspace editor can PATCH status
       accepted -> rejected (mark as spam/invalid after review)
       rejected -> accepted (override false rejection)
```

### 5.3 Webhook Delivery Status Machine

```
  +---------+   +---------+
  | PENDING |-->| SUCCESS |
  +---------+   +---------+
       |
       |  (HTTP error or timeout)
       |
       +---[retry up to max_attempts]---+
       |                                |
       v                                v
  +---------+                     +---------+
  | PENDING | (attempt_count++)   | FAILED  |
  +---------+                     +---------+
  
  Retry schedule (exponential backoff):
    Attempt 1: immediate
    Attempt 2: 60 seconds later
    Attempt 3: 300 seconds later
    After 3 failures: status = 'failed'
```

---

## 6. The 9 Tasks (4 Features + 5 Bugs)

---

### Task 1: Form Lifecycle State Machine
**Type:** Feature | **Difficulty:** Medium

**Problem Statement:**

The FormForge platform allows users to create forms in draft status, but there is currently no way to transition forms through their lifecycle. We need to build the status transition system so forms can be published to accept submissions, closed to stop accepting, and eventually archived.

Build an API endpoint that allows workspace editors to change a form's status. The transitions must follow strict rules: a draft form can only be published (not skipped directly to closed). A published form can only be closed. A closed form can be reopened (back to published) or archived. An archived form cannot be changed at all. Publishing requires the form to have at least one field and at least one required field. Reopening a closed form must increment the form version number so future submissions reference the new version.

The state machine must be enforced at the Mongoose model layer (pre-save hook) so that even direct database operations cannot bypass the rules. The service layer should add additional business rule checks (field count, required field check) before delegating to the model.

**Affected Files:**
- `src/pkg/features/form_lifecycle/service.js` (implement transition logic)
- `src/pkg/features/form_lifecycle/controller.js` (wire request to service)
- `src/pkg/features/form_lifecycle/routes.js` (register PATCH route)
- `src/models/Form.js` (pre-save hook for state machine -- already has skeleton, candidate must ensure it works)

**Solution Involves:**
- Implementing the `transitionFormStatus(formId, newStatus, userId)` service function
- Checking workspace membership before allowing transitions
- Validating field count and required field presence before publishing
- Incrementing `version` when reopening (closed -> published)
- Using the model's pre-save hook for transition validation
- Invalidating form cache after status change

**Test Cases (10):**
1. Returns 400 when transitioning from draft directly to closed (invalid transition)
2. Returns 400 when transitioning from draft directly to archived (invalid transition)
3. Allows draft -> published when form has fields with at least one required
4. Returns 400 when publishing form with zero fields
5. Returns 400 when publishing form with no required fields
6. Allows published -> closed
7. Allows closed -> published (reopen) and increments version
8. Allows closed -> archived
9. Returns 400 when transitioning from archived (terminal state)
10. Enforces hooks when saving directly via model (bypassing service)

**Anti-Gaming:** Tests create forms with randomized field counts and names. The version increment check uses the actual before/after value comparison, not a hardcoded number. Tests verify the pre-save hook fires independently of the service layer.

**Production Relevance:** State machines are fundamental in any workflow system -- order processing, content publishing, approval pipelines. Enforcing them at the data layer prevents bugs where business logic is bypassed.

---

### Task 2: Webhook Delivery with HMAC Signature
**Type:** Feature | **Difficulty:** Hard

**Problem Statement:**

When a form receives a submission, external systems need to be notified. FormForge supports webhook endpoints that are called when events occur, but the delivery system is not yet implemented. We need to build the webhook delivery pipeline that sends HTTP POST requests to registered endpoints with HMAC-SHA256 signatures for security.

When a submission is created, the system should look up all active webhooks for that form that subscribe to the relevant event (e.g., `submission.created`). For each webhook, it should construct a JSON payload, generate an HMAC-SHA256 signature using the webhook's secret, and send the request. The signature should be sent in the `X-FormForge-Signature` header. If delivery fails (non-2xx response or network error), the system should retry with exponential backoff, up to 3 attempts. Each delivery attempt must be logged in WebhookDelivery.

Also implement a test delivery endpoint that sends a sample payload to the webhook URL so users can verify their integration.

**Affected Files:**
- `src/pkg/features/webhook_delivery/service.js` (implement delivery logic)
- `src/pkg/features/webhook_delivery/controller.js` (wire test delivery)
- `src/pkg/features/webhook_delivery/routes.js` (register routes)
- `src/services/submission.service.js` (call webhook after submission created -- hook point provided)

**Solution Involves:**
- Implementing `deliverWebhook(webhookEndpoint, eventType, payload)` that:
  1. Serializes payload to JSON
  2. Computes HMAC-SHA256: `crypto.createHmac('sha256', secret).update(jsonPayload).digest('hex')`
  3. Sends POST request with headers: `Content-Type: application/json`, `X-FormForge-Signature: sha256=<hex>`, `X-FormForge-Event: <event_type>`
  4. Creates WebhookDelivery record with status/response
  5. On failure: schedules retry (next_retry_at = now + backoff)
- Implementing `triggerWebhooks(formId, eventType, payload)` called after submission
- Implementing test delivery endpoint

**Test Cases (12):**
1. Sends webhook with correct X-FormForge-Signature header
2. Signature is valid HMAC-SHA256 of the payload using webhook secret
3. Payload contains submission_id, form_id, event_type, and timestamp
4. Creates WebhookDelivery record with status 'success' on 200 response
5. Creates WebhookDelivery record with status 'failed' on non-2xx response
6. Increments attempt_count on each delivery attempt
7. Sets next_retry_at with exponential backoff (60s, 300s)
8. Marks delivery as 'failed' after max_attempts exhausted
9. Only delivers to active webhooks that subscribe to the event type
10. Does not deliver to soft-deleted webhooks
11. Test delivery endpoint sends sample payload and returns delivery result
12. Handles network errors gracefully (no unhandled promise rejections)

**Anti-Gaming:** Tests use a local HTTP server (created in test setup) that captures requests, so the signature and payload are verified against actual received data. The webhook secret is randomized per test. Tests verify the signature computation independently.

**Production Relevance:** Webhook delivery is a core integration pattern used by Stripe, GitHub, Shopify, and virtually every SaaS platform. HMAC signatures prevent spoofing. Retry with backoff handles transient failures.

---

### Task 3: Submission Idempotency
**Type:** Feature | **Difficulty:** Easy

**Problem Statement:**

Users sometimes double-click the submit button or their network retries a request. Currently, each submission creates a new record even if the same data is sent twice. We need to implement idempotency so that duplicate submissions are detected and the original submission is returned instead of creating a new one.

Build the idempotency check into the submission creation flow. The client provides an `idempotency_key` (either in the request body or in the `X-Idempotency-Key` header). If a submission with the same idempotency_key already exists, return the existing submission with a `duplicate: true` flag instead of creating a new one. The idempotency_key must be required -- reject submissions without one.

**Affected Files:**
- `src/pkg/features/submission_idempotency/service.js` (implement idempotency check)
- `src/pkg/features/submission_idempotency/controller.js` (extract key from body or header)
- `src/pkg/features/submission_idempotency/routes.js` (register submission route)

**Solution Involves:**
- Extracting idempotency_key from `req.body.idempotency_key` or `req.headers['x-idempotency-key']`
- Checking if a Submission with that key already exists
- If exists: return `200 { submission, duplicate: true }`
- If not: create new submission with the key, return `201 { submission }`
- Handling the unique index constraint on idempotency_key (race condition: use findOneAndUpdate with upsert, or try/catch on duplicate key error)

**Test Cases (10):**
1. Returns 400 when no idempotency_key is provided (neither body nor header)
2. Creates submission on first request with idempotency_key
3. Returns existing submission with `duplicate: true` on second request with same key
4. Does not increment form.submission_count on duplicate
5. Accepts idempotency_key from X-Idempotency-Key header
6. Accepts idempotency_key from request body
7. Header takes precedence over body when both provided
8. Different idempotency_keys create separate submissions
9. Returns 201 on first, 200 on duplicate (different status codes)
10. Does not fire webhooks on duplicate submission

**Anti-Gaming:** Tests use randomized idempotency keys (UUID-like). Tests verify the database has exactly one record after duplicate submission. Tests check both status code and the `duplicate` flag.

**Production Relevance:** Idempotency is critical for any payment or data-creation API. Stripe, AWS, and every serious API implements this pattern to handle network retries safely.

---

### Task 4: Submission Listing with Pagination & Filtering
**Type:** Feature | **Difficulty:** Hard

**Problem Statement:**

Workspace editors need to review submissions for their forms. Currently there is no endpoint to list submissions. We need to build a paginated listing endpoint that supports filtering by status, sorting, and search by respondent email.

Build a GET endpoint that returns submissions for a given form. The endpoint must support cursor-based or offset pagination (offset is acceptable), filtering by status, sorting by created_at (ascending or descending), and filtering by respondent_email. The response must include pagination metadata (page, limit, total, totalPages, hasNextPage, hasPrevPage). Only workspace members can access submissions. Soft-deleted submissions must be excluded.

The endpoint should also return a summary header with the total count per status, so the UI can show filter tabs with counts.

**Affected Files:**
- `src/pkg/features/submission_listing/service.js` (implement listing logic)
- `src/pkg/features/submission_listing/controller.js` (wire query params)
- `src/pkg/features/submission_listing/routes.js` (register GET route)

**Solution Involves:**
- Building the MongoDB query with optional filters: `{ form_id, status?, respondent_email? }`
- Using `findActive()` to exclude soft-deleted
- Implementing pagination with `skip/limit` and total count
- Clamping page to >= 1, limit to 1-100 range
- Sorting by `created_at` with configurable direction
- Computing status_summary via aggregation: `{ accepted: N, rejected: N, pending: N }`
- Validating workspace membership before returning data

**Test Cases (12):**
1. Returns 200 with paginated submissions for a form
2. Filters by status when status query param is provided
3. Filters by respondent_email (case-insensitive)
4. Sorts by created_at descending (default)
5. Sorts by created_at ascending when sort_order=asc
6. Clamps page to 1 when page is 0 or negative
7. Caps limit at 100 when limit exceeds 100
8. Clamps limit to 1 when limit is 0 or negative
9. Returns pagination metadata (totalPages, hasNextPage, hasPrevPage)
10. Returns status_summary with counts per status
11. Excludes soft-deleted submissions from results and counts
12. Returns 403 when user is not a workspace member

**Anti-Gaming:** Tests create a dynamic number of submissions (20-50) with random respondent emails and statuses. Pagination assertions check exact page boundaries. Status summary counts are verified against the actual inserted data, not hardcoded values.

**Production Relevance:** Every data-listing API needs proper pagination, filtering, and sorting. Getting boundary conditions right (page 0 clamping, limit caps, empty result sets) is a common source of bugs.

---

### Task 5: Validation Engine (Bug)
**Type:** Bug | **Difficulty:** Hard

**Problem Statement:**

The validation engine that processes submission field responses has several bugs. It is supposed to validate each field response against the field's type and rules, handle conditional fields (skip validation for hidden fields), and return structured error messages. However, several issues have been reported:

1. The validation engine does not check for required fields that are missing from the submission entirely -- it only validates fields that are present. If a required field is simply omitted from the responses array, no error is raised.

2. Conditional field validation is broken. Fields with a `conditional.depends_on` configuration should be skipped (not validated) when the parent field's value does not match `conditional.show_when`. Currently, the conditional check is inverted -- it validates the field when it should be hidden and skips it when it should be visible.

3. The regex pattern validation for text fields does not catch invalid patterns. If `validation.pattern` is set to an invalid regex string, the engine throws an unhandled error instead of returning a validation error message.

The candidate must fix all three bugs in the validation service.

**Affected Files:**
- `src/services/validation.service.js` (fix the three bugs)

**Solution Involves:**
- Bug 1: After validating present responses, iterate over all form fields and check if any required fields are missing from the responses array. Add a validation error for each missing required field.
- Bug 2: Fix the conditional logic. The condition `responseMap[field.conditional.depends_on] === field.conditional.show_when` should determine visibility. If the condition is NOT met, skip validation. Currently the logic is inverted.
- Bug 3: Wrap `new RegExp(field.validation.pattern)` in a try/catch. If the pattern is invalid, add a system-level validation error instead of letting the exception propagate.

**Test Cases (11):**
1. Returns error when a required text field is missing from responses
2. Returns error when a required email field is missing from responses
3. Returns no error when an optional field is missing
4. Skips validation for conditional field when parent value does not match show_when
5. Validates conditional field when parent value matches show_when
6. Required conditional field: no error when hidden (parent mismatch)
7. Required conditional field: error when visible and missing (parent matches)
8. Handles invalid regex pattern in validation.pattern without throwing
9. Validates correct regex pattern (e.g., /^\d{5}$/ for zip code)
10. Returns multiple validation errors when multiple fields fail
11. Returns field_id in each validation error for client-side field highlighting

**Anti-Gaming:** Tests use dynamically generated field names and IDs. The conditional field references are constructed at test time. Invalid regex patterns are varied (e.g., `"[unterminated"`, `"*invalid"`). Tests check both the error message content and the field_id reference.

**Production Relevance:** Validation engines are the backbone of data quality in any form system. Missing field detection, conditional logic, and graceful error handling are real bugs that ship in production code.

---

### Task 6: Cache Invalidation (Bug)
**Type:** Bug | **Difficulty:** Medium

**Problem Statement:**

FormForge caches form definitions in Redis to reduce database load. When a form is fetched via the public endpoint (`GET /api/v1/forms/public/:id`), the result is cached with a TTL of 5 minutes. When form listings are fetched (`GET /api/v1/forms`), results are also cached.

Two cache-related bugs have been reported:

1. When a form's fields are updated (`PUT /api/v1/forms/:id/fields`), the cache is not invalidated. Users see stale field definitions from the public endpoint after updating fields.

2. When a form's status changes (e.g., published -> closed), the public cache key (`forms:public:<id>`) is not invalidated. The closed form continues to appear as published in the public endpoint.

The candidate must add cache invalidation calls in the appropriate service functions.

**Affected Files:**
- `src/services/form.service.js` (add invalidation calls in updateFields and updateFormStatus)

**Solution Involves:**
- In `updateFields()`: after saving the form, call `invalidateCache('forms:public:' + formId)` and `invalidateCache('forms:list:*')`
- In `updateFormStatus()` (or the lifecycle service): after saving, call `invalidateCache('forms:public:' + formId)` and `invalidateCache('forms:list:*')`
- Pattern: mirror what ShowPass does -- invalidate both the specific entity cache and the list cache

**Test Cases (10):**
1. Public form endpoint returns cached result on second call (cache hit)
2. Cache key exists in Redis after first GET of public form
3. Updating form fields invalidates public form cache
4. Updated fields visible on next GET after field update (no stale data)
5. Form list cache exists after GET /forms
6. Creating a new form invalidates list cache
7. Changing form status invalidates public form cache
8. Closed form not returned as published in public endpoint after status change
9. Invalidation uses correct key patterns (forms:public:* and forms:list:*)
10. Cache works correctly for different forms (updating form A does not invalidate form B)

**Anti-Gaming:** Tests use Redis key inspection (`redisClient.keys()`) to verify cache state. Tests create multiple forms to ensure invalidation is scoped correctly. Dynamic form titles prevent hardcoding.

**Production Relevance:** Cache invalidation is one of the two hardest problems in computer science. Stale cache serving outdated form definitions is a real bug that impacts user experience.

---

### Task 7: Submission Rate Limiting (Bug)
**Type:** Bug | **Difficulty:** Medium

**Problem Statement:**

FormForge has a per-form rate limit to prevent spam submissions. Each form should allow at most 100 submissions per 60-second window. The rate limiting is implemented using Redis counters with TTL. However, two bugs have been reported:

1. The rate limit key does not include the form_id. It uses a global counter (`submission_rate:global`) instead of per-form counters (`submission_rate:<form_id>`). This means rate limits are shared across all forms -- if form A gets 100 submissions, form B is also blocked.

2. The TTL (expiry) is not set on the Redis key after the first increment. The counter increments indefinitely without ever resetting, permanently blocking forms after they hit 100 total submissions (not 100 per window).

The candidate must fix both bugs in the submission rate limiter.

**Affected Files:**
- `src/services/submission.service.js` (fix rate limit key and TTL logic)

**Solution Involves:**
- Bug 1: Change the Redis key from `submission_rate:global` to `submission_rate:${formId}`
- Bug 2: After the `INCR` command, if the new count is 1 (first submission in window), set `EXPIRE` on the key with the window duration (60 seconds). Pattern: `if (count === 1) await redisClient.expire(key, 60)`
- This is identical to the ShowPass `rateLimiter.js` pattern

**Test Cases (10):**
1. First submission to a form succeeds (rate limit not exceeded)
2. 100th submission to a form succeeds (at limit)
3. 101st submission returns 429 Too Many Requests
4. Different forms have independent rate limits (form A at limit, form B still accepts)
5. Rate limit resets after the window expires (submit, wait, submit again)
6. Response includes `Retry-After` header when rate limited
7. Rate limit counter uses form-specific key (verify Redis key pattern)
8. Rate limit window has correct TTL (verify Redis TTL on key)
9. Rate limit does not block submissions to other forms
10. Error message includes "rate limit" or "too many" in response

**Anti-Gaming:** Tests submit to two different forms in parallel to verify per-form isolation. The window expiry test uses a short window (2 seconds in test mode) to avoid slow tests. Redis keys are inspected directly.

**Production Relevance:** Rate limiting misconfiguration is a top-10 production bug. Shared counters and missing TTLs are exactly the kind of bugs that cause outages when a form goes viral.

---

### Task 8: Soft-Delete Data Leak (Bug)
**Type:** Bug | **Difficulty:** Easy

**Problem Statement:**

FormForge uses soft-delete (setting `deleted_at` instead of removing records) for forms and submissions. However, several endpoints are using `Model.find()` instead of `Model.findActive()`, causing soft-deleted records to leak into API responses.

Three specific leaks have been reported:

1. The public form endpoint (`GET /api/v1/forms/public/:id`) returns soft-deleted forms. A form that has been deleted by its creator is still accessible via the public URL.

2. The submission listing endpoint returns soft-deleted submissions in the results and counts them in the total.

3. The analytics endpoint includes soft-deleted submissions in its aggregation, inflating the total_submissions count and skewing field summaries.

**Affected Files:**
- `src/services/form.service.js` (fix public form fetch to use findOneActive)
- `src/services/submission.service.js` or `src/pkg/features/submission_listing/service.js` (fix listing to use findActive)
- `src/services/analytics.service.js` (fix aggregation to exclude deleted_at != null)

**Solution Involves:**
- Bug 1: Change `Form.findOne({ _id: formId })` to `Form.findOneActive({ _id: formId })` in the public form service
- Bug 2: Change `Submission.find({ form_id })` to `Submission.findActive({ form_id })` in the listing service
- Bug 3: Add `{ deleted_at: null }` to the MongoDB aggregation pipeline's `$match` stage in the analytics service

**Test Cases (10):**
1. Public endpoint returns 404 for soft-deleted form
2. Soft-deleted form is not returned in form listing
3. Soft-deleted submission is excluded from submission listing
4. Soft-deleted submission does not count in pagination total
5. Soft-deleted submission excluded from analytics total_submissions
6. Soft-deleted submission excluded from analytics field_summaries
7. Non-deleted form accessible via public endpoint (regression check)
8. Non-deleted submissions appear in listing (regression check)
9. Analytics correct when mix of deleted and non-deleted submissions exist
10. Soft-deleting a form does not delete its submissions (they remain but form is inaccessible)

**Anti-Gaming:** Tests create forms and submissions, soft-delete specific ones, then verify exact counts. The soft-deleted records are verified to exist in the database (via raw `findById`) but not in API responses. Dynamic data prevents hardcoding.

**Production Relevance:** Soft-delete data leaks are one of the most common security vulnerabilities in web applications. GDPR and data privacy regulations make this a compliance issue.

---

### Task 9: Field Type Input Validation (Bug)
**Type:** Bug | **Difficulty:** Easy

**Problem Statement:**

When submissions arrive, the validation engine checks field rules (required, min/max, pattern), but it does not validate that the submitted value matches the expected field type. A "number" field happily accepts a string value. An "email" field accepts "not-an-email". A "rating" field accepts the number 99.

Two bugs need to be fixed:

1. Type coercion is missing. The `number` field type should reject non-numeric values. The `email` field type should validate email format. The `rating` field type should only accept integers 1-5. The `date` field type should validate ISO date format. The `yes_no` field type should only accept boolean values.

2. The `dropdown` field type does not check that the submitted value is one of the defined options. A user can submit any arbitrary string for a dropdown field.

**Affected Files:**
- `src/services/validation.service.js` (add type-specific validation)

**Solution Involves:**
- Adding type checks in the validation loop:
  - `number`: `typeof value !== 'number' || isNaN(value)` -> error
  - `email`: `!/^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(value)` -> error
  - `rating`: `!Number.isInteger(value) || value < 1 || value > 5` -> error
  - `date`: `isNaN(new Date(value).getTime())` -> error
  - `yes_no`: `typeof value !== 'boolean'` -> error
  - `dropdown`: `!field.options.includes(value)` -> error
  - `multi_select`: `!Array.isArray(value) || value.some(v => !field.options.includes(v))` -> error

**Test Cases (10):**
1. Number field rejects string value "abc"
2. Number field accepts numeric value 42
3. Email field rejects "not-an-email"
4. Email field accepts "user@example.com"
5. Rating field rejects value 0 (below minimum)
6. Rating field rejects value 6 (above maximum)
7. Rating field accepts value 3
8. Dropdown field rejects value not in options list
9. Dropdown field accepts value in options list
10. Yes_no field rejects string "yes" (must be boolean true/false)

**Anti-Gaming:** Tests use randomized option lists for dropdowns. Email format validation uses edge cases (missing @, missing domain). Tests create complete form definitions with varied field types in a single form.

**Production Relevance:** Type validation is the first line of defense against bad data. Without it, downstream systems break when they receive a string where they expect a number.

---

## 7. Test Strategy

### Framework
- **mocha** (test runner) + **chai** (assertions) + **chai-http** (HTTP testing)
- **supertest** available as alternative
- **mocha-junit-reporter** for HackerRank XML output
- **cross-env** for environment variable management across platforms

### Test Structure
Following ShowPass patterns exactly:
- Features (tasks 1-4): tests in `src/pkg/features/<name>/tests/integration_test.js` with `test_fixtures.js`
- Bugs (tasks 5-9): tests in `test/task<N>/app.spec.js`
- Each task runs on a separate port (8081-8089) to allow parallel execution
- Tests connect to `formforge_test` database (separate from dev database)

### Test Data Generation
```javascript
// Dynamic fixture generation pattern (from ShowPass)
export const createUser = (overrides = {}) =>
  User.create({
    name: 'Test User',
    email: `test-${new mongoose.Types.ObjectId()}@test.com`,
    password: 'password123',
    role: 'editor',
    ...overrides,
  });

export const createForm = (overrides = {}) =>
  Form.create({
    title: `Form ${Date.now()}`,
    workspace_id: new mongoose.Types.ObjectId(),
    creator_id: new mongoose.Types.ObjectId(),
    status: 'draft',
    fields: [],
    ...overrides,
  });
```

Key anti-gaming techniques:
- **Random field names:** `field_${new mongoose.Types.ObjectId()}` so candidates cannot hardcode field IDs
- **Random validation rules:** min/max values, regex patterns generated per test
- **Random form structures:** number of fields, which fields are required, conditional dependencies
- **Dynamic counts:** submission counts vary so hardcoded pagination values fail
- **Email uniqueness:** each test user gets a unique email via ObjectId suffix

### Test Isolation
```javascript
before(async () => {
  // Connect to test database
  process.env.MONGO_URI = 'mongodb://localhost:27017/formforge_test';
  await mongoose.connect(process.env.MONGO_URI);
  
  // Safety check: verify test database
  const dbName = mongoose.connection.db?.databaseName;
  if (!dbName.includes('test')) throw new Error('Not test DB!');
  
  await redisClient.connect();
  await cleanupModels();
});

afterEach(async () => {
  // Clean Redis cache between tests
  const keys = await redisClient.keys('forms:*');
  if (keys.length > 0) await redisClient.del(...keys);
  // Clean task-specific models between tests
  await cleanupModels([Submission, Form]);
});

after(async () => {
  await cleanupModels();
  await redisClient.flushdb();
  await redisClient.quit();
  await mongoose.connection.close();
});
```

### package.json Test Scripts
```json
{
  "test:task1": "cross-env NODE_ENV=test PORT=8081 MOCHA_FILE=output/task1.xml mocha ... src/pkg/features/form_lifecycle/tests/integration_test.js --exit",
  "test:task2": "cross-env NODE_ENV=test PORT=8082 MOCHA_FILE=output/task2.xml mocha ... src/pkg/features/webhook_delivery/tests/integration_test.js --exit",
  "test:task3": "cross-env NODE_ENV=test PORT=8083 MOCHA_FILE=output/task3.xml mocha ... src/pkg/features/submission_idempotency/tests/integration_test.js --exit",
  "test:task4": "cross-env NODE_ENV=test PORT=8084 MOCHA_FILE=output/task4.xml mocha ... src/pkg/features/submission_listing/tests/integration_test.js --exit",
  "test:task5": "cross-env NODE_ENV=test PORT=8085 MOCHA_FILE=output/task5.xml mocha ... test/task5/*.js --exit",
  "test:task6": "cross-env NODE_ENV=test PORT=8086 MOCHA_FILE=output/task6.xml mocha ... test/task6/*.js --exit",
  "test:task7": "cross-env NODE_ENV=test PORT=8087 MOCHA_FILE=output/task7.xml mocha ... test/task7/*.js --exit",
  "test:task8": "cross-env NODE_ENV=test PORT=8088 MOCHA_FILE=output/task8.xml mocha ... test/task8/*.js --exit",
  "test:task9": "cross-env NODE_ENV=test PORT=8089 MOCHA_FILE=output/task9.xml mocha ... test/task9/*.js --exit"
}
```

---

## 8. Key Business Rules

### 8.1 Validation Engine Processing

The validation engine processes field responses in this order:

1. **Build response map:** Create a lookup `{ field_id: value }` from the responses array.
2. **Iterate over all form fields** (not just submitted responses):
   a. **Conditional check:** If field has `conditional.depends_on`, look up the parent field's value in the response map. If `responseMap[depends_on] !== show_when`, skip this field entirely (it is hidden).
   b. **Presence check:** If field is `required` and the value is missing/null/undefined/empty-string, add error `{ field_id, message: '<label> is required' }`.
   c. **Type check:** Validate the value matches the field type (number is numeric, email matches format, etc.).
   d. **Rule check:** Apply validation rules (min_length, max_length, min_value, max_value, pattern, min_selections, max_selections).
3. **Return errors array.** Empty array means all validations passed.

### 8.2 Conditional Logic

Conditional fields have:
```javascript
conditional: {
  depends_on: 'field_abc123',   // field_id of parent
  show_when: 'Yes',              // value that makes this field visible
}
```

**Rule:** If the parent field's submitted value equals `show_when`, the conditional field is visible and must be validated. If the parent value does not match (or parent is not submitted), the conditional field is hidden and skipped (even if marked required).

**Nested conditionals** are not supported in v1. A field can only depend on one parent. The parent cannot itself be conditional.

### 8.3 Submission Rate Limiting

Per-form rate limiting uses Redis:
```
Key:    submission_rate:<form_id>
Value:  integer counter
TTL:    60 seconds (set on first increment)
Limit:  100 submissions per window
```

When a submission arrives:
1. `INCR submission_rate:<form_id>`
2. If result is 1, `EXPIRE submission_rate:<form_id> 60`
3. If result > 100, reject with 429

### 8.4 Form Versioning

- New forms start at version 1
- Version increments ONLY when a closed form is reopened (closed -> published transition)
- Submissions record `form_version` at submission time
- This allows analytics to track submissions by form version
- Updating fields on a draft form does NOT increment version (drafts are pre-publication)

### 8.5 Webhook Signature Generation

```javascript
import crypto from 'crypto';

const generateSignature = (payload, secret) => {
  const jsonPayload = JSON.stringify(payload);
  const hmac = crypto.createHmac('sha256', secret).update(jsonPayload).digest('hex');
  return `sha256=${hmac}`;
};
```

Verification by the recipient:
```javascript
const expected = crypto.createHmac('sha256', secret).update(rawBody).digest('hex');
const received = req.headers['x-formforge-signature'].replace('sha256=', '');
const isValid = crypto.timingSafeEqual(Buffer.from(expected), Buffer.from(received));
```

Webhook payload structure:
```json
{
  "event": "submission.created",
  "timestamp": "2027-01-15T10:30:00.000Z",
  "form_id": "...",
  "submission_id": "...",
  "data": {
    "respondent_email": "...",
    "status": "accepted",
    "responses": [...]
  }
}
```

### 8.6 Analytics Aggregation

The analytics endpoint computes:

```javascript
{
  total_submissions: 150,
  status_breakdown: {
    pending: 0,
    validated: 0,
    accepted: 140,
    rejected: 10,
  },
  completion_rate: 93.3,  // (accepted / total) * 100
  field_summaries: [
    {
      field_id: 'field_abc',
      label: 'Full Name',
      type: 'text',
      response_count: 150,
      // text fields: no further aggregation
    },
    {
      field_id: 'field_def',
      label: 'Rating',
      type: 'rating',
      response_count: 148,
      average: 4.2,
      distribution: { 1: 2, 2: 5, 3: 20, 4: 50, 5: 71 },
    },
    {
      field_id: 'field_ghi',
      label: 'Department',
      type: 'dropdown',
      response_count: 150,
      option_counts: { 'Engineering': 60, 'Marketing': 45, 'Sales': 45 },
    },
  ],
  submission_rate: [
    { date: '2027-01-01', count: 5 },
    { date: '2027-01-02', count: 12 },
    // ... last 30 days
  ],
}
```

### 8.7 Soft-Delete Effects

When a form is soft-deleted (`deleted_at` set):
- The form no longer appears in listings (`findActive` excludes it)
- The public form endpoint returns 404
- Existing submissions remain in the database but are not accessible via the form's submission endpoints (form not found)
- Webhooks for the form stop firing

When a submission is soft-deleted:
- It no longer appears in submission listings
- It is excluded from analytics aggregation
- The form's `submission_count` is NOT decremented (it represents total ever submitted)

---

## 9. Swagger / API Documentation

### Setup

Following ShowPass's pattern, use `swagger-jsdoc` with inline OpenAPI comments:

```javascript
// src/config/swagger.js
import swaggerJsdoc from 'swagger-jsdoc';

const swaggerDefinition = {
  openapi: '3.0.0',
  info: {
    title: 'FormForge API',
    version: '1.0.0',
    description: 'Form Builder & Submission API â€” create forms, collect responses, analyze results.',
  },
  servers: [
    { url: '/api/v1', description: 'API v1' },
  ],
  components: {
    securitySchemes: {
      bearerAuth: {
        type: 'http',
        scheme: 'bearer',
        bearerFormat: 'JWT',
      },
    },
    schemas: {
      Error: { type: 'object', properties: { error: { type: 'string' } } },
      User: { /* ... */ },
      Workspace: { /* ... */ },
      Form: { /* ... */ },
      Field: { /* ... */ },
      Submission: { /* ... */ },
      WebhookEndpoint: { /* ... */ },
      WebhookDelivery: { /* ... */ },
    },
  },
};

export const swaggerSpec = swaggerJsdoc({
  swaggerDefinition,
  apis: ['./src/routes/*.js', './src/pkg/features/*/routes.js'],
});
```

The swagger file should define all schemas inline (as ShowPass does), providing complete documentation of all 9 field types, all status enums, all error responses, and all query parameters.

Candidates access docs at `http://localhost:3000/api-docs` (redirect from `/`).

---

## 10. Seed Data

```javascript
// src/seed.js
const seed = async () => {
  // Clear all collections
  await Promise.all([
    User.deleteMany({}),
    Workspace.deleteMany({}),
    WorkspaceMember.deleteMany({}),
    Form.deleteMany({}),
    Submission.deleteMany({}),
    WebhookEndpoint.deleteMany({}),
    WebhookDelivery.deleteMany({}),
    SubmissionQuota.deleteMany({}),
  ]);

  // --- Users ---
  const users = await User.create([
    { name: 'Admin User', email: 'admin@formforge.com', password: 'password123', role: 'admin' },
    { name: 'Alice Editor', email: 'alice@formforge.com', password: 'password123', role: 'editor' },
    { name: 'Bob Editor', email: 'bob@formforge.com', password: 'password123', role: 'editor' },
    { name: 'Charlie Viewer', email: 'charlie@formforge.com', password: 'password123', role: 'viewer' },
  ]);

  const admin = users[0], alice = users[1], bob = users[2], charlie = users[3];

  // --- Workspaces ---
  const workspace1 = await Workspace.create({
    name: 'Acme Corp',
    slug: 'acme-corp',
    owner_id: alice._id,
    description: 'Acme Corporation workspace',
  });
  
  const workspace2 = await Workspace.create({
    name: 'Beta Labs',
    slug: 'beta-labs',
    owner_id: bob._id,
    description: 'Beta Labs research workspace',
  });

  // --- Workspace Members ---
  await WorkspaceMember.create([
    { workspace_id: workspace1._id, user_id: alice._id, role: 'owner' },
    { workspace_id: workspace1._id, user_id: bob._id, role: 'editor' },
    { workspace_id: workspace1._id, user_id: charlie._id, role: 'viewer' },
    { workspace_id: workspace2._id, user_id: bob._id, role: 'owner' },
  ]);

  // --- Forms ---
  const contactForm = await Form.create({
    title: 'Contact Us',
    description: 'General contact form for customer inquiries',
    workspace_id: workspace1._id,
    creator_id: alice._id,
    status: 'published',
    version: 1,
    settings: { max_submissions: 1000, requires_auth: false },
    submission_count: 5,
    fields: [
      {
        field_id: 'field_name',
        label: 'Full Name',
        type: 'text',
        required: true,
        placeholder: 'Enter your full name',
        position: 0,
        validation: { min_length: 2, max_length: 100 },
      },
      {
        field_id: 'field_email',
        label: 'Email Address',
        type: 'email',
        required: true,
        placeholder: 'your@email.com',
        position: 1,
      },
      {
        field_id: 'field_subject',
        label: 'Subject',
        type: 'dropdown',
        required: true,
        options: ['General Inquiry', 'Bug Report', 'Feature Request', 'Billing'],
        position: 2,
      },
      {
        field_id: 'field_message',
        label: 'Message',
        type: 'textarea',
        required: true,
        placeholder: 'How can we help?',
        position: 3,
        validation: { min_length: 10, max_length: 2000 },
      },
      {
        field_id: 'field_priority',
        label: 'Priority',
        type: 'rating',
        required: false,
        position: 4,
      },
    ],
  });

  const feedbackForm = await Form.create({
    title: 'Product Feedback',
    description: 'Customer satisfaction survey',
    workspace_id: workspace1._id,
    creator_id: alice._id,
    status: 'published',
    version: 1,
    settings: { one_submission_per_user: true, requires_auth: true },
    submission_count: 3,
    fields: [
      {
        field_id: 'field_satisfaction',
        label: 'Overall Satisfaction',
        type: 'rating',
        required: true,
        position: 0,
      },
      {
        field_id: 'field_recommend',
        label: 'Would you recommend us?',
        type: 'yes_no',
        required: true,
        position: 1,
      },
      {
        field_id: 'field_why',
        label: 'Why or why not?',
        type: 'textarea',
        required: true,
        position: 2,
        conditional: {
          depends_on: 'field_recommend',
          show_when: 'true',   // Note: stored as string in show_when
        },
      },
      {
        field_id: 'field_features',
        label: 'Features you use most',
        type: 'multi_select',
        required: false,
        options: ['Dashboard', 'Reports', 'API', 'Mobile App', 'Integrations'],
        position: 3,
        validation: { min_selections: 1, max_selections: 3 },
      },
    ],
  });

  const draftForm = await Form.create({
    title: 'Employee Survey (Draft)',
    description: 'Internal employee satisfaction survey -- still being designed',
    workspace_id: workspace1._id,
    creator_id: alice._id,
    status: 'draft',
    version: 1,
    fields: [
      {
        field_id: 'field_dept',
        label: 'Department',
        type: 'dropdown',
        required: true,
        options: ['Engineering', 'Marketing', 'Sales', 'HR', 'Finance'],
        position: 0,
      },
    ],
  });

  const closedForm = await Form.create({
    title: 'Event Registration (Closed)',
    description: 'Registration for the Q4 company event',
    workspace_id: workspace2._id,
    creator_id: bob._id,
    status: 'closed',
    version: 2,
    settings: { max_submissions: 200 },
    submission_count: 150,
    fields: [
      {
        field_id: 'field_attendee_name',
        label: 'Attendee Name',
        type: 'text',
        required: true,
        position: 0,
      },
      {
        field_id: 'field_attendee_email',
        label: 'Email',
        type: 'email',
        required: true,
        position: 1,
      },
      {
        field_id: 'field_dietary',
        label: 'Dietary Restrictions',
        type: 'multi_select',
        required: false,
        options: ['None', 'Vegetarian', 'Vegan', 'Gluten-Free', 'Nut Allergy'],
        position: 2,
      },
      {
        field_id: 'field_attendance_date',
        label: 'Preferred Date',
        type: 'date',
        required: true,
        position: 3,
      },
    ],
  });

  // --- Submissions for Contact Form ---
  const submissionData = [
    {
      form_id: contactForm._id, form_version: 1,
      respondent_email: 'customer1@example.com',
      responses: [
        { field_id: 'field_name', value: 'John Smith' },
        { field_id: 'field_email', value: 'customer1@example.com' },
        { field_id: 'field_subject', value: 'General Inquiry' },
        { field_id: 'field_message', value: 'I have a question about your pricing plans.' },
        { field_id: 'field_priority', value: 3 },
      ],
      status: 'accepted',
      idempotency_key: 'seed-sub-1',
      metadata: { ip_address: '192.168.1.1', user_agent: 'Mozilla/5.0' },
    },
    {
      form_id: contactForm._id, form_version: 1,
      respondent_email: 'customer2@example.com',
      responses: [
        { field_id: 'field_name', value: 'Jane Doe' },
        { field_id: 'field_email', value: 'customer2@example.com' },
        { field_id: 'field_subject', value: 'Bug Report' },
        { field_id: 'field_message', value: 'Found a bug in the export feature when trying to download CSV.' },
        { field_id: 'field_priority', value: 5 },
      ],
      status: 'accepted',
      idempotency_key: 'seed-sub-2',
    },
    {
      form_id: contactForm._id, form_version: 1,
      respondent_email: 'spammer@spam.com',
      responses: [
        { field_id: 'field_name', value: 'X' },
        { field_id: 'field_email', value: 'spammer@spam.com' },
        { field_id: 'field_subject', value: 'General Inquiry' },
        { field_id: 'field_message', value: 'Buy now' },
      ],
      status: 'rejected',
      validation_errors: [
        { field_id: 'field_name', message: 'Full Name must be at least 2 characters' },
        { field_id: 'field_message', message: 'Message must be at least 10 characters' },
      ],
      idempotency_key: 'seed-sub-3',
    },
  ];

  await Submission.create(submissionData);

  // --- Webhook Endpoints ---
  await WebhookEndpoint.create([
    {
      form_id: contactForm._id,
      url: 'https://example.com/webhooks/formforge',
      secret: 'whsec_test_secret_12345',
      events: ['submission.created', 'submission.accepted'],
      active: true,
      description: 'Notify CRM on new submissions',
    },
    {
      form_id: contactForm._id,
      url: 'https://example.com/webhooks/slack',
      secret: 'whsec_slack_secret_67890',
      events: ['submission.created'],
      active: true,
      description: 'Post to Slack channel',
    },
  ]);

  console.log('\nSeed complete!');
  console.log('\nTest credentials:');
  console.log('  Admin:   admin@formforge.com   / password123');
  console.log('  Editor:  alice@formforge.com   / password123');
  console.log('  Editor:  bob@formforge.com     / password123');
  console.log('  Viewer:  charlie@formforge.com / password123');
  console.log('\nSample data:');
  console.log('  2 workspaces (Acme Corp, Beta Labs)');
  console.log('  4 forms (2 published, 1 draft, 1 closed)');
  console.log('  3 submissions (2 accepted, 1 rejected)');
  console.log('  2 webhook endpoints on Contact Form');
};
```

---

## 11. Field Types Reference

### text
- **Storage format:** String
- **Validation rules:** `required`, `min_length`, `max_length`, `pattern` (regex)
- **Response format:** `{ field_id: "field_x", value: "Hello world" }`
- **Type check:** `typeof value === 'string'`
- **Analytics aggregation:** `response_count` only (text is not aggregated)

### textarea
- **Storage format:** String
- **Validation rules:** `required`, `min_length`, `max_length`
- **Response format:** `{ field_id: "field_x", value: "Long text here..." }`
- **Type check:** `typeof value === 'string'`
- **Analytics aggregation:** `response_count`, `avg_length`

### number
- **Storage format:** Number
- **Validation rules:** `required`, `min_value`, `max_value`
- **Response format:** `{ field_id: "field_x", value: 42 }`
- **Type check:** `typeof value === 'number' && !isNaN(value)`
- **Analytics aggregation:** `response_count`, `average`, `min`, `max`, `sum`

### email
- **Storage format:** String
- **Validation rules:** `required` (format always checked)
- **Response format:** `{ field_id: "field_x", value: "user@example.com" }`
- **Type check:** `typeof value === 'string' && /^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(value)`
- **Analytics aggregation:** `response_count`, `unique_domains` (top 10)

### dropdown
- **Storage format:** String (single value from options)
- **Validation rules:** `required`, value must be in `options` array
- **Response format:** `{ field_id: "field_x", value: "Option A" }`
- **Type check:** `typeof value === 'string' && field.options.includes(value)`
- **Analytics aggregation:** `response_count`, `option_counts: { "Option A": 50, "Option B": 30 }`

### multi_select
- **Storage format:** Array of Strings (subset of options)
- **Validation rules:** `required`, each value must be in `options`, `min_selections`, `max_selections`
- **Response format:** `{ field_id: "field_x", value: ["A", "B"] }`
- **Type check:** `Array.isArray(value) && value.every(v => typeof v === 'string' && field.options.includes(v))`
- **Analytics aggregation:** `response_count`, `option_counts` (each option counted independently)

### rating
- **Storage format:** Number (integer 1-5)
- **Validation rules:** `required`
- **Response format:** `{ field_id: "field_x", value: 4 }`
- **Type check:** `Number.isInteger(value) && value >= 1 && value <= 5`
- **Analytics aggregation:** `response_count`, `average`, `distribution: { 1: N, 2: N, 3: N, 4: N, 5: N }`

### date
- **Storage format:** String (ISO 8601 date: `"2027-01-15"`)
- **Validation rules:** `required`
- **Response format:** `{ field_id: "field_x", value: "2027-01-15" }`
- **Type check:** `typeof value === 'string' && !isNaN(new Date(value).getTime())`
- **Analytics aggregation:** `response_count`, `earliest`, `latest`

### yes_no
- **Storage format:** Boolean
- **Validation rules:** `required`
- **Response format:** `{ field_id: "field_x", value: true }`
- **Type check:** `typeof value === 'boolean'`
- **Analytics aggregation:** `response_count`, `yes_count`, `no_count`, `yes_percentage`

---

## Summary of Task Distribution

| Task | Name | Type | Difficulty | Files Changed | Pattern Tested |
|------|------|------|------------|---------------|----------------|
| 1 | Form Lifecycle State Machine | Feature | Medium | pkg/features/form_lifecycle/* | State machine transitions |
| 2 | Webhook Delivery with HMAC | Feature | Hard | pkg/features/webhook_delivery/* | Async patterns, crypto |
| 3 | Submission Idempotency | Feature | Easy | pkg/features/submission_idempotency/* | Idempotency keys |
| 4 | Submission Listing | Feature | Hard | pkg/features/submission_listing/* | Pagination, filtering |
| 5 | Validation Engine Bugs | Bug | Hard | services/validation.service.js | Validation, conditional logic |
| 6 | Cache Invalidation | Bug | Medium | services/form.service.js | Cache patterns |
| 7 | Rate Limiting | Bug | Medium | services/submission.service.js | Redis rate limiting |
| 8 | Soft-Delete Data Leak | Bug | Easy | services/form.service.js, services/analytics.service.js | Soft-delete awareness |
| 9 | Field Type Validation | Bug | Easy | services/validation.service.js | Input validation |

**Total tests:** 10 + 12 + 10 + 12 + 11 + 10 + 10 + 10 + 10 = **95 tests** (target 100; add 1-2 tests to the larger suites to reach exactly 100)

---

### Critical Files for Implementation
- `C:/Users/RajkumarM/Desktop/HackerRank/ShowPass/MERN/src/models/Event.js` - Pattern to follow for Form.js state machine (pre-save hook, VALID_TRANSITIONS, post-init for _original_status)
- `C:/Users/RajkumarM/Desktop/HackerRank/ShowPass/MERN/src/services/cache.service.js` - Exact cache service to replicate (getCache, setCache, invalidateCache)
- `C:/Users/RajkumarM/Desktop/HackerRank/ShowPass/MERN/src/utils/softDelete.plugin.js` - Soft-delete plugin to reuse verbatim (findActive, findOneActive, countActive)
- `C:/Users/RajkumarM/Desktop/HackerRank/ShowPass/MERN/test/task5/app.spec.js` - Test pattern to follow for bug task tests (fixture creation, cleanup, Redis integration)
- `C:/Users/RajkumarM/Desktop/HackerRank/ShowPass/MERN/src/pkg/features/event_schedule/tests/integration_test.js` - Test pattern to follow for feature task tests (full integration with fixtures)