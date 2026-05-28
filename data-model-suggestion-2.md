# Data Model Suggestion 2: Event-Sourced / Audit-First (CQRS)

> Project: Podcast Production Platform · Created: 2026-05-20

## Philosophy

This model treats every state change as an immutable event appended to an event store. The current state of any entity (show, episode, audio file) is derived by replaying its event stream. Read-optimised projections (materialised views) serve the application's query needs, while the event store provides a complete audit trail and enables temporal queries ("what was the episode title on March 15th?").

This pattern is used in financial systems, compliance-heavy platforms, and collaborative editing tools where understanding *how* data reached its current state is as important as the state itself. For a podcast production platform, this is particularly powerful because: (a) multiple collaborators edit episodes through a workflow, (b) regulatory compliance may require knowing who changed what and when, (c) AI processing pipelines produce intermediate states that may need debugging or rollback, and (d) analytics on production patterns (how long episodes spend in each workflow stage) become trivial.

The write side accepts commands (CreateEpisode, PublishEpisode, ApplyNoiseReduction) and emits events. The read side subscribes to events and maintains denormalised projections optimised for specific query patterns (episode listing, RSS feed generation, analytics dashboards). This separation (CQRS) allows independent scaling and evolution of reads and writes.

**Best for:** Teams that need full audit trails, temporal queries, workflow analytics, AI pipeline debugging, and the ability to rebuild any read model from history.

**Trade-offs:**
- Pro: Complete audit trail — every change to every entity is preserved forever
- Pro: Temporal queries — "show me the episode as it was on date X"
- Pro: Workflow analytics — measure time in each stage, identify bottlenecks
- Pro: AI pipeline debugging — replay processing steps to understand failures
- Pro: Read model flexibility — add new projections without touching write logic
- Con: Higher complexity — eventual consistency between write and read sides
- Con: Event schema evolution — changing event schemas requires versioning strategy
- Con: Storage growth — events accumulate indefinitely (though they compress well)
- Con: Debugging read issues requires understanding event replay
- Con: Steeper learning curve for developers unfamiliar with event sourcing

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| RSS 2.0 + iTunes + Podcast 2.0 | RSS feed is a read projection rebuilt from episode lifecycle events |
| EBU R128 | Loudness values stored as `AudioAnalysisCompleted` events with LUFS measurements |
| Schema.org | Web page projections built from show/episode events carry Schema.org markup |
| JSON Chapters Format | `ChaptersGenerated` event payload contains the JSON chapters structure |
| WebVTT/SRT | `TranscriptGenerated` event stores the transcript in standard format |
| OAuth 2.0 | API access events tracked for security audit trail |

---

## Event Store

```sql
-- The single source of truth: an append-only event log
CREATE TABLE events (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    stream_id       UUID NOT NULL,                   -- Aggregate ID (show, episode, audio_file, etc.)
    stream_type     VARCHAR(50) NOT NULL,             -- show, episode, audio_file, person, organisation
    event_type      VARCHAR(100) NOT NULL,            -- ShowCreated, EpisodePublished, AudioProcessed, etc.
    event_version   INTEGER NOT NULL,                 -- Schema version for this event type
    sequence_number BIGINT NOT NULL,                  -- Position within this stream (for ordering)
    payload         JSONB NOT NULL,                   -- Event data
    metadata        JSONB NOT NULL DEFAULT '{}',      -- Correlation ID, causation ID, user_id, ip
    occurred_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Ensure ordering within a stream
CREATE UNIQUE INDEX idx_events_stream_seq ON events(stream_id, sequence_number);
-- Query by event type across all streams
CREATE INDEX idx_events_type ON events(event_type, occurred_at);
-- Query all events after a position (for projection rebuilds)
CREATE INDEX idx_events_created ON events(created_at);
-- JSONB index for querying event payloads
CREATE INDEX idx_events_payload ON events USING gin(payload);

-- Optimistic concurrency: ensure no two writers append to same stream simultaneously
-- Application checks expected_sequence before INSERT
```

### Example Events

```sql
-- ShowCreated event
-- payload: {"title": "The Dev Podcast", "slug": "the-dev-podcast", "language": "en",
--           "itunes_category": "Technology", "organisation_id": "uuid-here"}

-- EpisodeDrafted event
-- payload: {"title": "Episode 42: AI in Production", "show_id": "uuid",
--           "episode_number": 42, "season_number": 2}

-- AudioTrackUploaded event
-- payload: {"filename": "host-track.wav", "mime_type": "audio/wav",
--           "file_size_bytes": 145000000, "duration_seconds": 3600.5,
--           "storage_key": "s3://bucket/org/ep42/host-track.wav",
--           "track_label": "Host", "sample_rate": 48000}

-- AudioProcessingRequested event
-- payload: {"job_type": "noise_removal", "source_file_id": "uuid",
--           "params": {"noise_threshold": -40, "reduction_db": 12}}

-- AudioProcessingCompleted event
-- payload: {"job_type": "noise_removal", "output_file_id": "uuid",
--           "lufs_integrated": -16.1, "lufs_true_peak": -1.5, "duration_ms": 45000}

-- TranscriptGenerated event
-- payload: {"format": "vtt", "language": "en", "model": "whisper-large-v3",
--           "confidence_score": 0.94, "word_count": 8500,
--           "storage_key": "s3://bucket/org/ep42/transcript.vtt"}

-- ChaptersGenerated event
-- payload: {"chapters": [{"start_ms": 0, "title": "Intro", "end_ms": 60000},
--           {"start_ms": 60000, "title": "Guest Introduction", "end_ms": 180000}],
--           "model": "gpt-4o", "source": "transcript"}

-- EpisodePublished event
-- payload: {"published_at": "2026-05-20T10:00:00Z", "enclosure_url": "https://...",
--           "file_size_bytes": 48000000, "duration_seconds": 3601}

-- ClipExtracted event
-- payload: {"start_ms": 1200000, "end_ms": 1260000, "title": "Hot take on AI",
--           "shareability_score": 0.87, "format": "vertical",
--           "platform_target": "tiktok"}
```

---

## Command Handlers (Write Side)

```sql
-- Commands table for async command processing (optional — can be in-memory)
CREATE TABLE commands (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    command_type    VARCHAR(100) NOT NULL,            -- CreateShow, PublishEpisode, RequestProcessing
    aggregate_id    UUID,                             -- Target stream
    payload         JSONB NOT NULL,
    status          VARCHAR(20) NOT NULL DEFAULT 'pending',  -- pending, processing, completed, failed
    error_message   TEXT,
    issued_by       UUID NOT NULL,                    -- user_id
    issued_at       TIMESTAMPTZ NOT NULL DEFAULT now(),
    processed_at    TIMESTAMPTZ
);

CREATE INDEX idx_commands_status ON commands(status, issued_at);
```

---

## Read Projections

### Shows Projection

```sql
-- Materialised from ShowCreated, ShowUpdated, ShowArchived events
CREATE TABLE proj_shows (
    id                  UUID PRIMARY KEY,
    organisation_id     UUID NOT NULL,
    title               VARCHAR(500) NOT NULL,
    slug                VARCHAR(200) NOT NULL,
    description         TEXT,
    language            VARCHAR(10),
    author              VARCHAR(255),
    artwork_url         TEXT,
    itunes_category     VARCHAR(100),
    itunes_type         VARCHAR(20),
    itunes_explicit     BOOLEAN,
    podcast_guid        UUID,
    podcast_medium      VARCHAR(50),
    status              VARCHAR(20) NOT NULL,
    episode_count       INTEGER NOT NULL DEFAULT 0,
    latest_episode_at   TIMESTAMPTZ,
    rss_feed_url        TEXT,
    last_event_seq      BIGINT NOT NULL,             -- Track which event was last applied
    projected_at        TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_proj_shows_org ON proj_shows(organisation_id);
```

### Episodes Projection

```sql
-- Materialised from EpisodeDrafted, EpisodeUpdated, EpisodePublished, EpisodeArchived events
CREATE TABLE proj_episodes (
    id                  UUID PRIMARY KEY,
    show_id             UUID NOT NULL,
    organisation_id     UUID NOT NULL,
    title               VARCHAR(500) NOT NULL,
    slug                VARCHAR(200),
    description         TEXT,
    episode_number      INTEGER,
    season_number       INTEGER,
    itunes_episode_type VARCHAR(20),
    duration_seconds    INTEGER,
    published_at        TIMESTAMPTZ,
    status              VARCHAR(20) NOT NULL,
    -- Denormalised for fast access
    enclosure_url       TEXT,
    enclosure_mime_type VARCHAR(100),
    enclosure_size      BIGINT,
    artwork_url         TEXT,
    transcript_url      TEXT,
    chapters_url        TEXT,
    -- Workflow state
    has_transcript      BOOLEAN NOT NULL DEFAULT FALSE,
    has_chapters        BOOLEAN NOT NULL DEFAULT FALSE,
    has_show_notes      BOOLEAN NOT NULL DEFAULT FALSE,
    has_clips           BOOLEAN NOT NULL DEFAULT FALSE,
    processing_status   VARCHAR(20) DEFAULT 'idle',   -- idle, processing, complete
    -- People (denormalised array)
    host_names          TEXT[],
    guest_names         TEXT[],
    last_event_seq      BIGINT NOT NULL,
    projected_at        TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_proj_episodes_show ON proj_episodes(show_id);
CREATE INDEX idx_proj_episodes_published ON proj_episodes(published_at DESC);
CREATE INDEX idx_proj_episodes_status ON proj_episodes(status);
```

### RSS Feed Projection

```sql
-- Optimised for RSS feed generation — contains everything needed for a feed in one row per episode
CREATE TABLE proj_rss_items (
    id                  UUID PRIMARY KEY,
    show_id             UUID NOT NULL,
    episode_id          UUID NOT NULL,
    -- RSS <item> fields pre-rendered
    title               VARCHAR(500) NOT NULL,
    description         TEXT,
    pub_date            TIMESTAMPTZ NOT NULL,
    guid                UUID NOT NULL,
    enclosure_url       TEXT NOT NULL,
    enclosure_type      VARCHAR(100) NOT NULL,
    enclosure_length    BIGINT NOT NULL,
    duration_formatted  VARCHAR(20),                  -- HH:MM:SS
    itunes_episode_type VARCHAR(20),
    itunes_explicit     BOOLEAN,
    episode_number      INTEGER,
    season_number       INTEGER,
    artwork_url         TEXT,
    -- Podcast 2.0 tags
    transcript_url      TEXT,
    transcript_type     VARCHAR(20),
    chapters_url        TEXT,
    soundbites          JSONB,                        -- Array of {start, duration, title}
    persons             JSONB,                        -- Array of {name, role, img, href}
    last_event_seq      BIGINT NOT NULL,
    projected_at        TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_proj_rss_show ON proj_rss_items(show_id, pub_date DESC);
```

### Analytics Projection

```sql
-- Materialised from AnalyticsEventReceived events
CREATE TABLE proj_analytics_daily (
    show_id             UUID NOT NULL,
    episode_id          UUID NOT NULL,
    date                DATE NOT NULL,
    downloads           INTEGER NOT NULL DEFAULT 0,
    streams             INTEGER NOT NULL DEFAULT 0,
    unique_listeners    INTEGER NOT NULL DEFAULT 0,
    avg_listen_pct      NUMERIC(5,2),
    top_countries       JSONB,                        -- {"US": 450, "GB": 120, ...}
    top_apps            JSONB,                        -- {"apple_podcasts": 300, "spotify": 200}
    projected_at        TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (episode_id, date)
);

CREATE INDEX idx_proj_analytics_show ON proj_analytics_daily(show_id, date DESC);
```

### Workflow Timeline Projection

```sql
-- Tracks production workflow stages for each episode
CREATE TABLE proj_workflow_timeline (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    episode_id          UUID NOT NULL,
    show_id             UUID NOT NULL,
    stage               VARCHAR(50) NOT NULL,         -- drafted, recording_uploaded, processing, transcribed, chapters_added, show_notes_generated, reviewed, published
    entered_at          TIMESTAMPTZ NOT NULL,
    exited_at           TIMESTAMPTZ,
    duration_seconds    INTEGER,                      -- Time spent in this stage
    triggered_by        UUID,                         -- user_id or system
    event_id            UUID NOT NULL                 -- Source event that caused this transition
);

CREATE INDEX idx_workflow_episode ON proj_workflow_timeline(episode_id, entered_at);
CREATE INDEX idx_workflow_stage ON proj_workflow_timeline(stage, entered_at);
```

---

## Subscriptions & Projection Tracking

```sql
-- Tracks each projection's position in the event stream
CREATE TABLE projection_checkpoints (
    projection_name VARCHAR(100) PRIMARY KEY,
    last_event_id   UUID,
    last_event_at   TIMESTAMPTZ,
    last_rebuilt_at TIMESTAMPTZ,
    event_count     BIGINT NOT NULL DEFAULT 0,
    status          VARCHAR(20) NOT NULL DEFAULT 'running',  -- running, paused, rebuilding
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Dead letter queue for events that fail projection processing
CREATE TABLE projection_errors (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    projection_name VARCHAR(100) NOT NULL,
    event_id        UUID NOT NULL REFERENCES events(id),
    error_message   TEXT NOT NULL,
    retry_count     INTEGER NOT NULL DEFAULT 0,
    next_retry_at   TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## Identity (Minimal — Not Event-Sourced)

```sql
-- Identity tables are NOT event-sourced (CRUD is fine for auth)
CREATE TABLE organisations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    plan            VARCHAR(50) NOT NULL DEFAULT 'free',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email           VARCHAR(255) NOT NULL UNIQUE,
    display_name    VARCHAR(255) NOT NULL,
    avatar_url      TEXT,
    password_hash   TEXT,
    auth_provider   VARCHAR(50),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE organisation_members (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    role            VARCHAR(50) NOT NULL DEFAULT 'member',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organisation_id, user_id)
);
```

---

## Example Queries

### Rebuild episode state from events

```sql
-- Get current state of an episode by replaying its event stream
SELECT event_type, payload, occurred_at
FROM events
WHERE stream_id = '{{episode_uuid}}'
  AND stream_type = 'episode'
ORDER BY sequence_number ASC;
```

### Temporal query — episode state at a point in time

```sql
-- What was this episode's title on March 15, 2026?
SELECT payload->>'title' as title
FROM events
WHERE stream_id = '{{episode_uuid}}'
  AND stream_type = 'episode'
  AND event_type IN ('EpisodeDrafted', 'EpisodeUpdated')
  AND occurred_at <= '2026-03-15T23:59:59Z'
ORDER BY sequence_number DESC
LIMIT 1;
```

### Workflow analytics — average time from upload to publish

```sql
SELECT
    show_id,
    AVG(duration_seconds) as avg_seconds_in_stage,
    stage
FROM proj_workflow_timeline
WHERE stage IN ('processing', 'reviewed')
  AND exited_at IS NOT NULL
GROUP BY show_id, stage;
```

### Find all episodes where AI processing failed

```sql
SELECT stream_id as episode_id, payload, occurred_at
FROM events
WHERE event_type = 'AudioProcessingFailed'
  AND occurred_at > now() - interval '7 days'
ORDER BY occurred_at DESC;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Event Store | 1 | events (the single source of truth) |
| Commands | 1 | commands (async command processing) |
| Projections — Content | 3 | proj_shows, proj_episodes, proj_rss_items |
| Projections — Analytics | 1 | proj_analytics_daily |
| Projections — Workflow | 1 | proj_workflow_timeline |
| Projection Infrastructure | 2 | projection_checkpoints, projection_errors |
| Identity (CRUD) | 3 | organisations, users, organisation_members |
| **Total** | **12** | Plus additional projections added as needed |

---

## Key Design Decisions

1. **Single events table as source of truth** — all domain state changes are immutable events; no UPDATE or DELETE on domain data. This provides a complete, tamper-evident audit trail.

2. **Stream-based event organisation** — events are grouped by `stream_id` (the aggregate ID) with a `stream_type` discriminator. Each entity's history is its event stream.

3. **CQRS with multiple read projections** — different projections serve different access patterns (episode listing, RSS feed generation, analytics). New projections can be added without touching the write side.

4. **Identity is NOT event-sourced** — authentication and RBAC use simple CRUD tables because the audit trail for auth changes has minimal value compared to the complexity cost.

5. **Projection checkpoints for reliability** — each projection tracks its position in the event stream, enabling restart after failure and full rebuilds from scratch.

6. **Workflow timeline as a projection** — production workflow analytics (time-in-stage, bottleneck identification) are automatically derived from the event stream without additional instrumentation.

7. **Event versioning** — `event_version` field enables schema evolution of event payloads without breaking existing projections.

8. **Metadata for correlation** — every event carries `metadata` with `correlation_id` (original request), `causation_id` (event that caused this event), and `user_id` for complete traceability through async processing chains.
