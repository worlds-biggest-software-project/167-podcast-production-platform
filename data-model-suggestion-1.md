# Data Model Suggestion 1: Entity-Centric Normalized Relational

> Project: Podcast Production Platform · Created: 2026-05-20

## Philosophy

This model follows classical relational database design: every concept gets its own table, relationships are expressed through foreign keys, and data integrity is enforced at the database level through constraints, unique indexes, and referential integrity rules. The schema is designed to align closely with the RSS 2.0 + iTunes namespace + Podcast 2.0 namespace data structures, so that generating a standards-compliant podcast feed is a straightforward JOIN across a predictable set of tables.

The normalized approach minimises data redundancy and makes it easy to answer ad-hoc queries across the dataset. A question like "which guests have appeared on multiple shows?" or "what is the average episode duration by season?" is a simple SQL query against well-indexed tables. This design is battle-tested in content management systems and media catalogues.

The trade-off is table count. A fully normalized podcast production platform requires 30-40 tables to cover shows, episodes, seasons, guests, transcripts, chapters, clips, analytics, distribution, and access control. Migrations become a regular activity as the domain evolves, and some queries (e.g., assembling a full RSS feed) require multi-table JOINs.

**Best for:** Teams that prioritise data integrity, complex cross-entity reporting, and long-term maintainability over rapid schema evolution.

**Trade-offs:**
- Pro: Strong data integrity via foreign keys and constraints
- Pro: Easy to query ad-hoc with standard SQL
- Pro: Well-understood by most backend engineers
- Pro: Standards-aligned field names make RSS/feed generation straightforward
- Con: High table count increases migration burden
- Con: Multi-table JOINs for feed generation can be slower without caching
- Con: Schema changes require migrations for every new field
- Con: Less flexible for jurisdiction-specific or plugin-specific metadata

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| RSS 2.0 | `shows` table maps to `<channel>`, `episodes` table maps to `<item>`, `episode_enclosures` maps to `<enclosure>` |
| iTunes Namespace | `shows.itunes_category`, `shows.itunes_type`, `episodes.itunes_explicit`, `episodes.itunes_episode_type` columns |
| Podcast 2.0 Namespace | Dedicated tables: `episode_transcripts`, `episode_chapters`, `episode_soundbites`, `episode_persons`, `show_funding`, `show_value_recipients` |
| Schema.org PodcastSeries/PodcastEpisode | `shows` and `episodes` tables carry all Schema.org required properties |
| EBU R128 / ATSC A/85 | `audio_files.lufs_integrated`, `audio_files.lufs_true_peak` columns for loudness metadata |
| ID3v2.4 | `audio_files.id3_tags` stores tag mapping for MP3 metadata embedding |
| ISO 3166-1 | `analytics_events.country_code` uses ISO 3166-1 alpha-2 |
| OAuth 2.0 / JWT | `oauth_clients`, `api_tokens` tables for API authentication |

---

## Core Identity & Multi-Tenancy

```sql
CREATE TABLE organisations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    plan            VARCHAR(50) NOT NULL DEFAULT 'free',  -- free, creator, pro, enterprise
    billing_email   VARCHAR(255),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email           VARCHAR(255) NOT NULL UNIQUE,
    display_name    VARCHAR(255) NOT NULL,
    avatar_url      TEXT,
    password_hash   TEXT,               -- NULL if OAuth-only
    auth_provider   VARCHAR(50),        -- google, github, email
    auth_provider_id VARCHAR(255),
    email_verified  BOOLEAN NOT NULL DEFAULT FALSE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE organisation_members (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    role            VARCHAR(50) NOT NULL DEFAULT 'member',  -- owner, admin, editor, member
    invited_at      TIMESTAMPTZ,
    accepted_at     TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organisation_id, user_id)
);

CREATE INDEX idx_org_members_org ON organisation_members(organisation_id);
CREATE INDEX idx_org_members_user ON organisation_members(user_id);
```

---

## Show Management

```sql
CREATE TABLE shows (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id     UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    title               VARCHAR(500) NOT NULL,
    slug                VARCHAR(200) NOT NULL,
    description         TEXT,
    summary             TEXT,                       -- Short summary for directories
    language            VARCHAR(10) NOT NULL DEFAULT 'en',  -- BCP 47 language tag
    author              VARCHAR(255),
    owner_name          VARCHAR(255),
    owner_email         VARCHAR(255),
    copyright           TEXT,
    website_url         TEXT,
    artwork_url         TEXT,                       -- 3000x3000 minimum per Apple spec
    -- iTunes namespace fields
    itunes_category     VARCHAR(100),               -- Primary category
    itunes_subcategory  VARCHAR(100),
    itunes_type         VARCHAR(20) DEFAULT 'episodic',  -- episodic, serial
    itunes_explicit     BOOLEAN NOT NULL DEFAULT FALSE,
    itunes_complete     BOOLEAN NOT NULL DEFAULT FALSE,
    -- Podcast 2.0 fields
    podcast_guid        UUID,                       -- podcast:guid tag
    podcast_medium      VARCHAR(50) DEFAULT 'podcast',  -- podcast, music, video, film, audiobook
    podcast_locked      BOOLEAN NOT NULL DEFAULT FALSE,
    podcast_lock_owner  VARCHAR(255),
    -- Distribution
    rss_feed_url        TEXT,
    rss_feed_built_at   TIMESTAMPTZ,
    -- Status
    status              VARCHAR(20) NOT NULL DEFAULT 'draft',  -- draft, active, paused, archived
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organisation_id, slug)
);

CREATE INDEX idx_shows_org ON shows(organisation_id);
CREATE INDEX idx_shows_status ON shows(status);

CREATE TABLE show_categories (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    show_id         UUID NOT NULL REFERENCES shows(id) ON DELETE CASCADE,
    category        VARCHAR(100) NOT NULL,
    subcategory     VARCHAR(100),
    is_primary      BOOLEAN NOT NULL DEFAULT FALSE,
    UNIQUE (show_id, category, subcategory)
);

CREATE TABLE show_funding (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    show_id         UUID NOT NULL REFERENCES shows(id) ON DELETE CASCADE,
    url             TEXT NOT NULL,
    message         VARCHAR(500),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE show_value_recipients (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    show_id         UUID NOT NULL REFERENCES shows(id) ON DELETE CASCADE,
    recipient_name  VARCHAR(255) NOT NULL,
    recipient_type  VARCHAR(50) NOT NULL,           -- wallet, node
    address         TEXT NOT NULL,
    split           INTEGER NOT NULL DEFAULT 100,   -- Percentage split (out of total)
    fee             BOOLEAN NOT NULL DEFAULT FALSE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## Seasons & Episodes

```sql
CREATE TABLE seasons (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    show_id         UUID NOT NULL REFERENCES shows(id) ON DELETE CASCADE,
    season_number   INTEGER NOT NULL,
    title           VARCHAR(500),
    description     TEXT,
    artwork_url     TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (show_id, season_number)
);

CREATE TABLE episodes (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    show_id             UUID NOT NULL REFERENCES shows(id) ON DELETE CASCADE,
    season_id           UUID REFERENCES seasons(id) ON DELETE SET NULL,
    title               VARCHAR(500) NOT NULL,
    slug                VARCHAR(200) NOT NULL,
    description         TEXT,                       -- HTML allowed per RSS spec
    summary             TEXT,                       -- Plain text summary
    episode_number      INTEGER,
    -- iTunes namespace fields
    itunes_episode_type VARCHAR(20) DEFAULT 'full',  -- full, trailer, bonus
    itunes_explicit     BOOLEAN NOT NULL DEFAULT FALSE,
    itunes_block        BOOLEAN NOT NULL DEFAULT FALSE,
    -- Timing
    duration_seconds    INTEGER,
    published_at        TIMESTAMPTZ,
    scheduled_at        TIMESTAMPTZ,                -- Future publish date
    -- Artwork
    artwork_url         TEXT,                       -- Episode-specific override
    -- Status
    status              VARCHAR(20) NOT NULL DEFAULT 'draft',  -- draft, scheduled, published, archived
    -- Workflow
    created_by          UUID REFERENCES users(id),
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (show_id, slug)
);

CREATE INDEX idx_episodes_show ON episodes(show_id);
CREATE INDEX idx_episodes_status ON episodes(status);
CREATE INDEX idx_episodes_published ON episodes(published_at DESC);
CREATE INDEX idx_episodes_show_number ON episodes(show_id, episode_number);
```

---

## Audio & Media Files

```sql
CREATE TABLE audio_files (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    episode_id          UUID REFERENCES episodes(id) ON DELETE SET NULL,
    organisation_id     UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    -- File metadata
    filename            VARCHAR(500) NOT NULL,
    mime_type           VARCHAR(100) NOT NULL,       -- audio/mpeg, audio/x-m4a, audio/wav
    file_size_bytes     BIGINT NOT NULL,
    duration_seconds    NUMERIC(10,2),
    sample_rate         INTEGER,                     -- 44100, 48000
    bit_rate            INTEGER,                     -- 128000, 256000, 320000
    channels            SMALLINT,                    -- 1 (mono), 2 (stereo)
    codec               VARCHAR(50),                 -- mp3, aac, wav, flac
    -- Storage
    storage_provider    VARCHAR(50) NOT NULL DEFAULT 's3',
    storage_key         TEXT NOT NULL,
    storage_bucket      VARCHAR(255),
    public_url          TEXT,
    cdn_url             TEXT,
    -- Audio quality metadata (EBU R128)
    lufs_integrated     NUMERIC(5,1),                -- Target: -16.0 LUFS
    lufs_true_peak      NUMERIC(5,1),                -- Should be below -1.0 dBTP
    lufs_range          NUMERIC(5,1),
    -- Processing
    processing_status   VARCHAR(20) NOT NULL DEFAULT 'pending',  -- pending, processing, complete, failed
    is_original         BOOLEAN NOT NULL DEFAULT TRUE,
    source_file_id      UUID REFERENCES audio_files(id),  -- Points to original if this is a processed version
    -- Type
    file_purpose        VARCHAR(50) NOT NULL DEFAULT 'episode',  -- episode, raw_track, clip, trailer
    track_label         VARCHAR(100),                -- "Host", "Guest 1", "Guest 2"
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_audio_files_episode ON audio_files(episode_id);
CREATE INDEX idx_audio_files_org ON audio_files(organisation_id);
CREATE INDEX idx_audio_files_purpose ON audio_files(file_purpose);

CREATE TABLE episode_enclosures (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    episode_id      UUID NOT NULL REFERENCES episodes(id) ON DELETE CASCADE,
    audio_file_id   UUID NOT NULL REFERENCES audio_files(id),
    url             TEXT NOT NULL,                   -- Public URL for RSS <enclosure>
    mime_type       VARCHAR(100) NOT NULL,            -- audio/mpeg
    file_size_bytes BIGINT NOT NULL,
    is_primary      BOOLEAN NOT NULL DEFAULT TRUE,
    -- Podcast 2.0: alternateEnclosure support
    bitrate         INTEGER,
    title           VARCHAR(255),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (episode_id, audio_file_id)
);
```

---

## Transcripts & Chapters

```sql
CREATE TABLE episode_transcripts (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    episode_id      UUID NOT NULL REFERENCES episodes(id) ON DELETE CASCADE,
    format          VARCHAR(20) NOT NULL,            -- vtt, srt, json, html, txt
    language        VARCHAR(10) NOT NULL DEFAULT 'en',
    url             TEXT,                            -- Public URL for podcast:transcript tag
    storage_key     TEXT,
    content         TEXT,                            -- Full transcript text for search
    is_machine_generated BOOLEAN NOT NULL DEFAULT TRUE,
    generation_model VARCHAR(100),                   -- whisper-large-v3, etc.
    confidence_score NUMERIC(4,3),                   -- 0.000 to 1.000
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_transcripts_episode ON episode_transcripts(episode_id);
CREATE INDEX idx_transcripts_content_search ON episode_transcripts USING gin(to_tsvector('english', content));

CREATE TABLE episode_chapters (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    episode_id      UUID NOT NULL REFERENCES episodes(id) ON DELETE CASCADE,
    chapter_number  INTEGER NOT NULL,
    start_time_ms   INTEGER NOT NULL,                -- Milliseconds from start
    end_time_ms     INTEGER,
    title           VARCHAR(500) NOT NULL,
    url             TEXT,                            -- Optional link for chapter
    image_url       TEXT,                            -- Optional chapter artwork
    is_hidden       BOOLEAN NOT NULL DEFAULT FALSE,  -- Per JSON chapters spec
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (episode_id, chapter_number)
);

CREATE INDEX idx_chapters_episode ON episode_chapters(episode_id);

CREATE TABLE episode_soundbites (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    episode_id      UUID NOT NULL REFERENCES episodes(id) ON DELETE CASCADE,
    start_time_ms   INTEGER NOT NULL,
    duration_ms     INTEGER NOT NULL,
    title           VARCHAR(500),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## Guests & People

```sql
CREATE TABLE persons (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    full_name       VARCHAR(255) NOT NULL,
    email           VARCHAR(255),
    bio             TEXT,
    avatar_url      TEXT,
    website_url     TEXT,
    twitter_handle  VARCHAR(100),
    linkedin_url    TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_persons_org ON persons(organisation_id);
CREATE INDEX idx_persons_name ON persons(full_name);

-- Maps to podcast:person tag
CREATE TABLE episode_persons (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    episode_id      UUID NOT NULL REFERENCES episodes(id) ON DELETE CASCADE,
    person_id       UUID NOT NULL REFERENCES persons(id) ON DELETE CASCADE,
    role            VARCHAR(100) NOT NULL DEFAULT 'guest',  -- host, co-host, guest, editor, producer
    role_group      VARCHAR(100) DEFAULT 'cast',             -- cast, crew (per podcast namespace spec)
    display_name    VARCHAR(255),                            -- Override name for this appearance
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (episode_id, person_id, role)
);

CREATE INDEX idx_episode_persons_episode ON episode_persons(episode_id);
CREATE INDEX idx_episode_persons_person ON episode_persons(person_id);
```

---

## AI Content Generation

```sql
CREATE TABLE ai_show_notes (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    episode_id      UUID NOT NULL REFERENCES episodes(id) ON DELETE CASCADE,
    summary         TEXT,
    key_points      TEXT,                            -- Markdown bullet list
    action_items    TEXT,
    keywords        TEXT[],
    generation_model VARCHAR(100),
    generation_prompt TEXT,
    is_approved     BOOLEAN NOT NULL DEFAULT FALSE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE ai_clips (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    episode_id      UUID NOT NULL REFERENCES episodes(id) ON DELETE CASCADE,
    audio_file_id   UUID REFERENCES audio_files(id),
    start_time_ms   INTEGER NOT NULL,
    end_time_ms     INTEGER NOT NULL,
    title           VARCHAR(500),
    description     TEXT,
    shareability_score NUMERIC(4,3),                 -- AI-predicted engagement score
    format          VARCHAR(50) NOT NULL DEFAULT 'landscape',  -- landscape, square, vertical
    platform_target VARCHAR(50),                     -- tiktok, youtube_shorts, instagram_reels
    status          VARCHAR(20) NOT NULL DEFAULT 'suggested',  -- suggested, approved, exported
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_ai_clips_episode ON ai_clips(episode_id);

CREATE TABLE ai_guest_research (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    person_id       UUID NOT NULL REFERENCES persons(id) ON DELETE CASCADE,
    episode_id      UUID REFERENCES episodes(id) ON DELETE SET NULL,
    research_summary TEXT,
    recent_work     TEXT,
    suggested_questions TEXT,
    sources         TEXT[],
    generation_model VARCHAR(100),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## Distribution & Publishing

```sql
CREATE TABLE distribution_targets (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    show_id         UUID NOT NULL REFERENCES shows(id) ON DELETE CASCADE,
    platform        VARCHAR(50) NOT NULL,            -- apple_podcasts, spotify, youtube, amazon_music
    platform_id     VARCHAR(255),                    -- External ID on the platform
    submission_url  TEXT,
    status          VARCHAR(20) NOT NULL DEFAULT 'pending',  -- pending, submitted, approved, rejected
    submitted_at    TIMESTAMPTZ,
    approved_at     TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (show_id, platform)
);

CREATE TABLE episode_publications (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    episode_id      UUID NOT NULL REFERENCES episodes(id) ON DELETE CASCADE,
    distribution_target_id UUID NOT NULL REFERENCES distribution_targets(id),
    external_id     VARCHAR(255),                    -- Episode ID on the platform
    external_url    TEXT,
    status          VARCHAR(20) NOT NULL DEFAULT 'pending',  -- pending, published, failed
    published_at    TIMESTAMPTZ,
    error_message   TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_publications_episode ON episode_publications(episode_id);
```

---

## Analytics

```sql
CREATE TABLE analytics_events (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    episode_id      UUID NOT NULL REFERENCES episodes(id) ON DELETE CASCADE,
    show_id         UUID NOT NULL REFERENCES shows(id),
    event_type      VARCHAR(50) NOT NULL,            -- download, stream, partial_play, complete_play
    -- Listener context
    country_code    CHAR(2),                         -- ISO 3166-1 alpha-2
    region          VARCHAR(100),
    city            VARCHAR(100),
    device_type     VARCHAR(50),                     -- mobile, desktop, smart_speaker
    app_name        VARCHAR(100),                    -- apple_podcasts, spotify, overcast
    -- Engagement
    listen_duration_seconds INTEGER,
    listen_percentage NUMERIC(5,2),
    -- Attribution
    referrer        TEXT,
    user_agent      TEXT,
    ip_hash         VARCHAR(64),                     -- Hashed for privacy (GDPR)
    occurred_at     TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_analytics_episode ON analytics_events(episode_id);
CREATE INDEX idx_analytics_show ON analytics_events(show_id);
CREATE INDEX idx_analytics_occurred ON analytics_events(occurred_at DESC);
CREATE INDEX idx_analytics_type ON analytics_events(event_type);

-- Materialized daily rollup for performance
CREATE TABLE analytics_daily (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    episode_id      UUID NOT NULL REFERENCES episodes(id) ON DELETE CASCADE,
    show_id         UUID NOT NULL REFERENCES shows(id),
    date            DATE NOT NULL,
    downloads       INTEGER NOT NULL DEFAULT 0,
    streams         INTEGER NOT NULL DEFAULT 0,
    unique_listeners INTEGER NOT NULL DEFAULT 0,
    avg_listen_pct  NUMERIC(5,2),
    top_country     CHAR(2),
    top_app         VARCHAR(100),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (episode_id, date)
);

CREATE INDEX idx_analytics_daily_show_date ON analytics_daily(show_id, date DESC);
```

---

## API & Integrations

```sql
CREATE TABLE api_tokens (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    token_hash      VARCHAR(64) NOT NULL UNIQUE,     -- SHA-256 hash of the token
    scopes          TEXT[] NOT NULL DEFAULT '{}',     -- read, write, admin
    last_used_at    TIMESTAMPTZ,
    expires_at      TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE webhooks (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    url             TEXT NOT NULL,
    events          TEXT[] NOT NULL,                  -- episode.published, recording.complete, etc.
    secret          VARCHAR(255) NOT NULL,
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    last_triggered_at TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## Processing Jobs

```sql
CREATE TABLE processing_jobs (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    episode_id      UUID REFERENCES episodes(id) ON DELETE SET NULL,
    audio_file_id   UUID REFERENCES audio_files(id) ON DELETE SET NULL,
    job_type        VARCHAR(50) NOT NULL,            -- transcode, noise_removal, loudness_norm, transcribe, clip_extract, chapter_gen, show_notes_gen
    status          VARCHAR(20) NOT NULL DEFAULT 'queued',  -- queued, running, complete, failed
    priority        INTEGER NOT NULL DEFAULT 5,
    input_params    JSONB,
    output_params   JSONB,
    error_message   TEXT,
    started_at      TIMESTAMPTZ,
    completed_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_jobs_status ON processing_jobs(status, priority);
CREATE INDEX idx_jobs_episode ON processing_jobs(episode_id);
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Identity & Access | 3 | organisations, users, organisation_members |
| Show Management | 4 | shows, show_categories, show_funding, show_value_recipients |
| Episodes & Seasons | 2 | seasons, episodes |
| Audio & Media | 2 | audio_files, episode_enclosures |
| Transcripts & Chapters | 3 | episode_transcripts, episode_chapters, episode_soundbites |
| Guests & People | 2 | persons, episode_persons |
| AI Content | 3 | ai_show_notes, ai_clips, ai_guest_research |
| Distribution | 2 | distribution_targets, episode_publications |
| Analytics | 2 | analytics_events, analytics_daily |
| API & Integrations | 2 | api_tokens, webhooks |
| Processing | 1 | processing_jobs |
| **Total** | **26** | |

---

## Key Design Decisions

1. **UUID primary keys everywhere** — enables distributed ID generation, avoids sequential ID enumeration attacks, and simplifies future sharding.

2. **Organisation-scoped multi-tenancy** — all content belongs to an organisation via foreign keys; Row-Level Security policies can enforce tenant isolation at the database level.

3. **Separate `audio_files` table with self-referencing** — raw tracks, processed versions, and clips all live in one table with `source_file_id` pointing to the original, enabling full provenance tracking.

4. **Standards-aligned column names** — `itunes_*` and `podcast_*` prefixed columns map directly to RSS namespace attributes, making feed generation a straightforward SELECT.

5. **Dedicated tables for Podcast 2.0 entities** — transcripts, chapters, soundbites, persons, funding, and value recipients each get their own table rather than being crammed into JSONB, enabling relational queries across them.

6. **Analytics split into events and daily rollups** — raw events for detailed analysis, pre-aggregated daily table for dashboard performance.

7. **Processing jobs as first-class entities** — every AI or audio processing task is tracked with status, input/output params, and timing for observability and retry logic.

8. **Text search on transcripts** — GIN index on `to_tsvector('english', content)` enables full-text search across episode transcripts without an external search engine for basic use cases.
