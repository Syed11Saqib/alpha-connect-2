# Database Design

## Conventions

PostgreSQL is the operational system of record. Use UUID primary keys, `timestamptz` in UTC, `created_at`, `updated_at`, optimistic concurrency where concurrent edits matter, and `deleted_at` only where soft deletion is justified. All tenant-owned tables include non-null `tenant_id` and indexes that begin with it. Store secrets encrypted and never store plaintext access tokens.

## Core relational model

| Area                    | Tables / purpose                                                                                                                                                                |
| ----------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Tenancy & access        | `tenants`, `users`, `memberships`, `roles`, `permissions`, `role_permissions`, `sessions`, `agent_presence`, `api_keys`, `audit_logs`                                           |
| Channels                | `whatsapp_accounts`, `whatsapp_phone_numbers`, `channel_credentials`, `provider_webhook_subscriptions`                                                                          |
| Media storage           | `media_assets` stores provider-neutral metadata and object-storage references for attachments, template media, exports, and retained documents.                                 |
| Contacts                | `contacts`, `contact_identifiers`, `contact_attributes`, `tags`, `contact_tags`, `consent_records`, `contact_merge_history`, `segments`, `segment_versions`                     |
| Inbox                   | `conversations`, `conversation_participants`, `messages`, `message_attachments`, `message_status_events`, `conversation_assignments`, `conversation_notes`, `conversation_tags` |
| Templates               | `templates`, `template_versions`, `template_components`, `template_media_assets`                                                                                                |
| Campaigns               | `campaigns`, `campaign_audiences`, `campaign_recipients`, `campaign_dispatches`, `campaign_events`, `suppression_entries`                                                       |
| Automation              | `bot_flows`, `bot_flow_versions`, `bot_sessions`, `bot_session_steps`, `automation_rules`, `notification_jobs`                                                                  |
| Integration reliability | `webhook_inbox_events`, `outbox_events`, `integration_deliveries`, `idempotency_keys`, `dead_letter_jobs`                                                                       |
| CRM                     | `crm_connections`, `crm_entity_mappings`, `crm_sync_cursors`, `crm_sync_jobs`, `crm_conflicts`                                                                                  |
| AI & analytics          | `ai_requests`, `ai_suggestions`, `usage_records`, `analytics_events`, `metric_daily_rollups`                                                                                    |

## Important relationships

- A tenant has many memberships, channels, contacts, conversations, templates, campaigns, and CRM connections.
- A contact has many identifiers and consent records. WhatsApp identifiers require normalized E.164 values and tenant-scoped uniqueness.
- A conversation belongs to one tenant and WhatsApp phone number; it has participants and ordered messages.
- `agent_presence` is tenant- and membership-scoped. It records availability state, `last_seen_at`, and active authenticated socket/session references needed for real-time Inbox routing. Presence is ephemeral operational state with expiry/heartbeat semantics; it is not an attendance system or permanent audit record.
- Contact tags classify people; `conversation_tags` classify individual threads independently. A conversation may carry many tags from the tenant's shared `tags` catalog, allowing the same contact's separate conversations to be routed or reported differently.
- A campaign references an immutable audience snapshot; campaign recipients are materialized to ensure reproducible delivery and reporting.
- Template versions are immutable once submitted/published; messages retain template/version/provider identifiers used at send time.
- Webhook inbox events and outbox events have unique idempotency keys and immutable payload/audit metadata.

## Media asset abstraction

`media_assets` is the single provider-neutral registry for media and documents. It owns tenant ID, storage object key, content type, byte size, checksum, original filename where permitted, media classification, lifecycle/scanning status, retention/deletion timestamps, and opaque provider references. It must not use a mutable provider URL as the durable application reference.

`message_attachments` is a join table between `messages` and `media_assets`; `template_media_assets` is a join table between template versions/components and `media_assets`. This permits an asset to have consistent authorization, malware-scanning, lifecycle, audit, and storage behavior regardless of whether Meta, a future BSP, or a tenant uploaded it. Short-lived provider retrieval URLs or Meta media IDs are stored only as opaque integration metadata and may be refreshed by the WhatsApp Provider module.

## ER-style relationship diagram

```text
Tenant 1---* Membership *---1 User
   |              |
   |              +---0..* AgentPresence
   |
   +---* Contact 1---* ContactIdentifier
   |       |
   |       +---* ContactTag *---1 Tag
   |
   +---* WhatsAppPhoneNumber 1---* Conversation 1---* Message
   |                                  |                 |
   |                                  +---* ConversationTag *---1 Tag
   |                                  |                 |
   |                                  |                 +---* MessageAttachment *---1 MediaAsset
   |                                  |
   |                                  +---* ConversationAssignment *---1 Membership
   |
   +---* Template 1---* TemplateVersion 1---* TemplateMediaAsset *---1 MediaAsset
   |
   +---* Campaign 1---* CampaignRecipient 1---* CampaignDispatch
```

Cardinality in the diagram represents logical ownership/association; all tenant-owned relations are constrained to the same tenant.

## Indexing and scale

Index Inbox queries by `(tenant_id, status, updated_at DESC)`, messages by `(conversation_id, sent_at, id)`, contacts by `(tenant_id, normalized_identifier)`, campaign work by `(campaign_id, status, scheduled_at)`, and webhook/outbox processing by `(status, available_at)`. Use keyset pagination, not offsets, for high-volume inboxes, messages, contacts, and events. Partition high-volume append-only event tables by time only after observed growth warrants the operational complexity.

## Data protection and retention

Classify contact PII, message content, provider credentials, and AI prompt content as sensitive. Encrypt secrets with envelope encryption/KMS; restrict PII exports; audit read/export actions; support tenant-scoped erasure and configurable retention. Exact retention periods, legal hold requirements, and backup deletion guarantees require product/legal confirmation.

## Integrity rules

Use foreign keys for internal ownership, check constraints for enumerated states, unique provider IDs scoped to provider/channel, and state-transition validation in the application layer. JSONB is permitted for provider payload preservation and controlled extensible attributes, but core query fields must be normalized columns.
