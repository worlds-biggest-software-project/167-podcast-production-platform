# Data Model Suggestion 3: Hybrid Relational + JSONB

> Project: Podcast Production Platform · Created: 2026-05-20

## Philosophy

This model uses a smaller set of relational tables for the core entities (shows, episodes, audio files, users) but delegates variable, domain-specific, and evolving metadata to JSONB columns. Instead of separate tables for chapters, soundbites, funding links, value recipients, and Podcast 2.0 tags, these are stored as structured JSON within the parent entity's row.

This pattern is widely used in modern SaaS platforms (Shopify's metafields, Stripe's metadata, Notion's property system) where the core data structure is stable but the edges evolve rapidly. For a podcast platform, this is compelling because: (a) the Podcast 2.0 namespace is still evolving with new tags being added regularly, (b) different shows may use completely different subsets of features (some use chapters, some use value4value, some use neither), (c) AI-generated content metadata varies per model and version, and (d) rapid MVP development benefits from fewer migrations.

The key discipline is: relational columns for anything you need to query, filter, sort, or JOIN on; JSONB for everything else. PostgreSQL's JSONB operators, GIN indexes, and containment queries (`@>`) make this practical without sacrificing query performance for the common access patterns.

**Best for:** Teams prioritising rapid iteration, MVP speed, tolerance for schema evolution, and multi-tenant platforms where different tenants use different feature sets.

**Trade-offs:**
- Pro: Fewer tables (roughly half the normalized model) — faster to develop and migrate
- Pro: New Podcast 2.0 tags can be supported without schema migration
- Pro: AI metadata evolves freely without blocking deployments
- Pro: JSONB containment queries are fast with GIN indexes
- Pro: Simpler ORM mappings — less JOIN complexity
- Con: No foreign key constraints inside JSONB — data integrity relies on application logic
- Con: JSONB fields are harder to document and validate — schema drift is a real risk
- Con: Reporting queries on JSONB fields are more verbose (->>, @>, jsonb_array_elements)
- Con: Cannot enforce NOT NULL or UNIQUE constraints within JSONB structures
- Con: Larger row sizes can affect table scan performance

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| RSS 2.0 | Core `shows` and `episodes` relational columns cover all RSS 2.0 channel/item fields |
| iTunes Namespace | `shows.itunes_meta` and `episodes.itunes_meta` JSONB columns store all iTunes-specific tags |
| Podcast 2.0 Namespace | `shows.podcast2_meta` and `episodes.podcast2_meta` JSONB columns — new tags added without migration |
| EBU R128 | `audio_files.audio_analysis` JSONB stores LUFS measurements alongside other analysis data |
| Schema.org | `shows.seo_meta` JSONB contains Schema.org PodcastSeries structured data |
| JSON Chapters Format | `episodes.podcast2_meta.chapters` stores the JSON chapters array directly |
| WebVTT/SRT | Transcript files stored in object storage; reference in `episodes.transcripts` JSONB array |
| ID3v2.4 | `audio_files.id3_tags` JSONB for flexible tag mapping |

---

## Core Tables

```sql
CREATE TABLE organisations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    plan            VARCHAR(50) NOT NULL DEFAULT 'free',
    settings        JSONB NOT NULL DEFAULT '{}',
    -- settings example: {"default_language": "en", "default_lufs_target": -16,
    --   "branding": {"primary_color": "#1a1a2e", "logo_url": "https://..."},
    --   "integrations": {"zapier_key": "...", "slack_webhook": "..."}}
    billing_email   VARCHAR(255),
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
    auth_provider_id VARCHAR(255),
    preferences     JSONB NOT NULL DEFAULT '{}',
    -- preferences example: {"theme": "dark", "notifications": {"email": true, "slack": false},
    --   "default_editor": "timeline", "timezone": "America/New_York"}
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE organisation_members (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    role            VARCHAR(50) NOT NULL DEFAULT 'member',
    permissions     JSONB NOT NULL DEFAULT '{}',
    -- permissions example: {"shows": ["read", "write"], "analytics": ["read"],
    --   "billing": [], "api_keys": ["read"]}
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organisation_id, user_id)
);

CREATE INDEX idx_org_members_org ON organisation_members(organisation_id);
CREATE INDEX idx_org_members_user ON organisation_members(user_id);
```

---

## Shows

```sql
CREATE TABLE shows (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id     UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    -- Core relational fields (queryable, sortable)
    title               VARCHAR(500) NOT NULL,
    slug                VARCHAR(200) NOT NULL,
    description         TEXT,
    language            VARCHAR(10) NOT NULL DEFAULT 'en',
    author              VARCHAR(255),
    artwork_url         TEXT,
    status              VARCHAR(20) NOT NULL DEFAULT 'draft',
    episode_count       INTEGER NOT NULL DEFAULT 0,
    latest_episode_at   TIMESTAMPTZ,
    -- iTunes namespace (JSONB — all iTunes-specific fields)
    itunes_meta         JSONB NOT NULL DEFAULT '{}',
    -- itunes_meta example:
    -- {"category": "Technology", "subcategory": "Tech News",
    --  "type": "episodic", "explicit": false, "complete": false,
    --  "owner_name": "Jane Doe", "owner_email": "jane@example.com",
    --  "additional_categories": [{"category": "Business", "subcategory": "Entrepreneurship"}]}
    -- Podcast 2.0 namespace (JSONB — evolving spec)
    podcast2_meta       JSONB NOT NULL DEFAULT '{}',
    -- podcast2_meta example:
    -- {"guid": "uuid-here", "medium": "podcast", "locked": false, "lock_owner": "jane@example.com",
    --  "funding": [{"url": "https://patreon.com/show", "message": "Support us!"}],
    --  "value": {"type": "lightning", "method": "keysend",
    --    "recipients": [{"name": "Host", "type": "wallet", "address": "abc123", "split": 90},
    --                   {"name": "Editor", "type": "wallet", "address": "def456", "split": 10}]},
    --  "podroll": [{"feed_url": "https://other-podcast.com/rss"}],
    --  "update_frequency": "weekly"}
    -- SEO / Schema.org
    seo_meta            JSONB NOT NULL DEFAULT '{}',
    -- Distribution
    distribution        JSONB NOT NULL DEFAULT '{}',
    -- distribution example:
    -- {"rss_feed_url": "https://feed.example.com/show.xml",
    --  "targets": [
    --    {"platform": "apple_podcasts", "id": "123456", "status": "approved", "url": "https://..."},
    --    {"platform": "spotify", "id": "abc", "status": "approved", "url": "https://..."},
    --    {"platform": "youtube", "id": "xyz", "status": "pending"}
    --  ]}
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organisation_id, slug)
);

CREATE INDEX idx_shows_org ON shows(organisation_id);
CREATE INDEX idx_shows_status ON shows(status);
CREATE INDEX idx_shows_itunes ON shows USING gin(itunes_meta);
CREATE INDEX idx_shows_podcast2 ON shows USING gin(podcast2_meta);
```

---

## Episodes

```sql
CREATE TABLE episodes (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    show_id             UUID NOT NULL REFERENCES shows(id) ON DELETE CASCADE,
    organisation_id     UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    -- Core relational fields (queryable, sortable, filterable)
    title               VARCHAR(500) NOT NULL,
    slug                VARCHAR(200) NOT NULL,
    description         TEXT,
    episode_number      INTEGER,
    season_number       INTEGER,
    duration_seconds    INTEGER,
    published_at        TIMESTAMPTZ,
    scheduled_at        TIMESTAMPTZ,
    status              VARCHAR(20) NOT NULL DEFAULT 'draft',
    artwork_url         TEXT,
    created_by          UUID REFERENCES users(id),
    -- iTunes namespace (JSONB)
    itunes_meta         JSONB NOT NULL DEFAULT '{}',
    -- itunes_meta example:
    -- {"episode_type": "full", "explicit": false, "block": false,
    --  "title": "Override Title for iTunes"}
    -- Podcast 2.0 namespace (JSONB — rich structured data)
    podcast2_meta       JSONB NOT NULL DEFAULT '{}',
    -- podcast2_meta example:
    -- {"transcript": {"url": "https://cdn.example.com/ep42/transcript.vtt",
    --                 "type": "text/vtt", "language": "en"},
    --  "chapters": {"url": "https://cdn.example.com/ep42/chapters.json", "type": "application/json+chapters"},
    --  "soundbites": [{"start": 1200.5, "duration": 60, "title": "Key insight on AI"}],
    --  "persons": [
    --    {"name": "Jane Doe", "role": "host", "group": "cast", "img": "https://...", "href": "https://..."},
    --    {"name": "John Smith", "role": "guest", "group": "cast", "img": "https://..."}
    --  ],
    --  "location": {"geo": "geo:37.7749,-122.4194", "osm": "R123456", "name": "San Francisco, CA"},
    --  "alternate_enclosures": [
    --    {"type": "audio/opus", "length": 12000000, "bitrate": 96000, "url": "https://..."}
    --  ]}
    -- Enclosure (primary audio for RSS)
    enclosure           JSONB NOT NULL DEFAULT '{}',
    -- enclosure example:
    -- {"url": "https://cdn.example.com/ep42/episode.mp3", "type": "audio/mpeg",
    --  "length": 48000000, "audio_file_id": "uuid"}
    -- AI-generated content (JSONB — varies by model and version)
    ai_content          JSONB NOT NULL DEFAULT '{}',
    -- ai_content example:
    -- {"show_notes": {"summary": "...", "key_points": ["...", "..."],
    --                 "keywords": ["AI", "production"], "model": "gpt-4o", "approved": true},
    --  "clips": [
    --    {"start_ms": 1200000, "end_ms": 1260000, "title": "Hot take",
    --     "shareability_score": 0.87, "format": "vertical", "platform": "tiktok",
    --     "audio_file_id": "uuid", "status": "approved"}
    --  ],
    --  "guest_research": {"person_name": "John Smith", "summary": "...",
    --                     "suggested_questions": ["...", "..."], "model": "gpt-4o"}}
    -- Workflow state
    workflow            JSONB NOT NULL DEFAULT '{}',
    -- workflow example:
    -- {"stages_completed": ["uploaded", "noise_removed", "transcribed", "chapters_generated"],
    --  "current_stage": "review", "processing_jobs": ["uuid1", "uuid2"],
    --  "last_processed_at": "2026-05-20T10:00:00Z"}
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (show_id, slug)
);

CREATE INDEX idx_episodes_show ON episodes(show_id);
CREATE INDEX idx_episodes_org ON episodes(organisation_id);
CREATE INDEX idx_episodes_status ON episodes(status);
CREATE INDEX idx_episodes_published ON episodes(published_at DESC);
CREATE INDEX idx_episodes_podcast2 ON episodes USING gin(podcast2_meta);
CREATE INDEX idx_episodes_ai ON episodes USING gin(ai_content);

-- Full-text search on episode descriptions
CREATE INDEX idx_episodes_fts ON episodes USING gin(to_tsvector('english', title || ' ' || COALESCE(description, '')));
```

---

## Audio Files

```sql
CREATE TABLE audio_files (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    episode_id          UUID REFERENCES episodes(id) ON DELETE SET NULL,
    organisation_id     UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    -- Core relational fields
    filename            VARCHAR(500) NOT NULL,
    mime_type           VARCHAR(100) NOT NULL,
    file_size_bytes     BIGINT NOT NULL,
    duration_seconds    NUMERIC(10,2),
    codec               VARCHAR(50),
    file_purpose        VARCHAR(50) NOT NULL DEFAULT 'episode',
    track_label         VARCHAR(100),
    is_original         BOOLEAN NOT NULL DEFAULT TRUE,
    source_file_id      UUID REFERENCES audio_files(id),
    processing_status   VARCHAR(20) NOT NULL DEFAULT 'pending',
    -- Storage (JSONB — provider-agnostic)
    storage             JSONB NOT NULL DEFAULT '{}',
    -- storage example:
    -- {"provider": "s3", "bucket": "podcast-assets", "key": "org/ep42/host-track.wav",
    --  "region": "us-east-1", "public_url": "https://cdn.example.com/...",
    --  "cdn_url": "https://d123.cloudfront.net/..."}
    -- Audio analysis (JSONB — varies by analysis tool)
    audio_analysis      JSONB NOT NULL DEFAULT '{}',
    -- audio_analysis example:
    -- {"lufs_integrated": -16.1, "lufs_true_peak": -1.5, "lufs_range": 8.2,
    --  "sample_rate": 48000, "bit_rate": 320000, "channels": 2,
    --  "noise_floor_db": -60, "clipping_detected": false,
    --  "silence_regions": [{"start_ms": 0, "end_ms": 1500}, {"start_ms": 3600000, "end_ms": 3602000}]}
    -- ID3 tags for MP3 embedding
    id3_tags            JSONB NOT NULL DEFAULT '{}',
    -- id3_tags example:
    -- {"TIT2": "Episode 42: AI in Production", "TPE1": "The Dev Podcast",
    --  "TALB": "The Dev Podcast", "TRCK": "42", "TYER": "2026",
    --  "APIC": "https://cdn.example.com/artwork.jpg"}
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_audio_files_episode ON audio_files(episode_id);
CREATE INDEX idx_audio_files_org ON audio_files(organisation_id);
CREATE INDEX idx_audio_files_purpose ON audio_files(file_purpose);
CREATE INDEX idx_audio_files_status ON audio_files(processing_status);
```

---

## Transcripts

```sql
CREATE TABLE transcripts (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    episode_id          UUID NOT NULL REFERENCES episodes(id) ON DELETE CASCADE,
    format              VARCHAR(20) NOT NULL,         -- vtt, srt, json, txt
    language            VARCHAR(10) NOT NULL DEFAULT 'en',
    content             TEXT,                          -- Full transcript text for search
    storage_key         TEXT,                          -- Object storage key for file
    public_url          TEXT,
    -- Generation metadata (JSONB)
    generation_meta     JSONB NOT NULL DEFAULT '{}',
    -- generation_meta example:
    -- {"model": "whisper-large-v3", "confidence_score": 0.94,
    --  "word_count": 8500, "processing_time_ms": 45000,
    --  "is_machine_generated": true, "edited_by": null}
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_transcripts_episode ON transcripts(episode_id);
CREATE INDEX idx_transcripts_fts ON transcripts USING gin(to_tsvector('english', content));
```

---

## Persons / Guests

```sql
CREATE TABLE persons (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    full_name       VARCHAR(255) NOT NULL,
    email           VARCHAR(255),
    bio             TEXT,
    avatar_url      TEXT,
    -- Contact and social (JSONB — variable per person)
    contact_info    JSONB NOT NULL DEFAULT '{}',
    -- contact_info example:
    -- {"website": "https://johndoe.com", "twitter": "@johndoe",
    --  "linkedin": "https://linkedin.com/in/johndoe",
    --  "mastodon": "@john@mastodon.social", "bluesky": "@john.bsky.social"}
    -- Appearance history (denormalised for quick access)
    appearance_count INTEGER NOT NULL DEFAULT 0,
    last_appeared_at TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_persons_org ON persons(organisation_id);
CREATE INDEX idx_persons_name ON persons(full_name);
```

---

## Analytics

```sql
CREATE TABLE analytics_events (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    episode_id      UUID NOT NULL REFERENCES episodes(id) ON DELETE CASCADE,
    show_id         UUID NOT NULL,
    event_type      VARCHAR(50) NOT NULL,
    -- Structured event data (JSONB — varies by event type)
    event_data      JSONB NOT NULL DEFAULT '{}',
    -- download event_data example:
    -- {"country_code": "US", "region": "California", "city": "San Francisco",
    --  "device_type": "mobile", "app_name": "apple_podcasts",
    --  "listen_duration_seconds": 1800, "listen_percentage": 50.0,
    --  "referrer": "https://twitter.com/...", "ip_hash": "abc123"}
    occurred_at     TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (occurred_at);

-- Create monthly partitions
CREATE TABLE analytics_events_2026_01 PARTITION OF analytics_events
    FOR VALUES FROM ('2026-01-01') TO ('2026-02-01');
CREATE TABLE analytics_events_2026_02 PARTITION OF analytics_events
    FOR VALUES FROM ('2026-02-01') TO ('2026-03-01');
CREATE TABLE analytics_events_2026_03 PARTITION OF analytics_events
    FOR VALUES FROM ('2026-03-01') TO ('2026-04-01');
CREATE TABLE analytics_events_2026_04 PARTITION OF analytics_events
    FOR VALUES FROM ('2026-04-01') TO ('2026-05-01');
CREATE TABLE analytics_events_2026_05 PARTITION OF analytics_events
    FOR VALUES FROM ('2026-05-01') TO ('2026-06-01');
CREATE TABLE analytics_events_2026_06 PARTITION OF analytics_events
    FOR VALUES FROM ('2026-06-01') TO ('2026-07-01');

CREATE INDEX idx_analytics_episode ON analytics_events(episode_id, occurred_at);
CREATE INDEX idx_analytics_show ON analytics_events(show_id, occurred_at);
CREATE INDEX idx_analytics_type ON analytics_events(event_type, occurred_at);
CREATE INDEX idx_analytics_data ON analytics_events USING gin(event_data);

-- Pre-aggregated daily rollup
CREATE TABLE analytics_daily (
    show_id         UUID NOT NULL,
    episode_id      UUID NOT NULL,
    date            DATE NOT NULL,
    stats           JSONB NOT NULL DEFAULT '{}',
    -- stats example:
    -- {"downloads": 450, "streams": 200, "unique_listeners": 380,
    --  "avg_listen_pct": 62.5,
    --  "countries": {"US": 250, "GB": 80, "DE": 45, "CA": 30},
    --  "apps": {"apple_podcasts": 200, "spotify": 150, "overcast": 50},
    --  "devices": {"mobile": 300, "desktop": 100, "smart_speaker": 50}}
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (episode_id, date)
);

CREATE INDEX idx_daily_show ON analytics_daily(show_id, date DESC);
```

---

## Processing Jobs

```sql
CREATE TABLE processing_jobs (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    episode_id      UUID REFERENCES episodes(id) ON DELETE SET NULL,
    audio_file_id   UUID REFERENCES audio_files(id) ON DELETE SET NULL,
    job_type        VARCHAR(50) NOT NULL,
    status          VARCHAR(20) NOT NULL DEFAULT 'queued',
    priority        INTEGER NOT NULL DEFAULT 5,
    -- Input/output as JSONB (varies by job type)
    params          JSONB NOT NULL DEFAULT '{}',
    -- noise_removal params example:
    -- {"input": {"file_id": "uuid", "storage_key": "..."}, 
    --  "config": {"noise_threshold": -40, "reduction_db": 12},
    --  "output": {"file_id": "uuid", "storage_key": "...", "duration_ms": 45000}}
    error_message   TEXT,
    started_at      TIMESTAMPTZ,
    completed_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_jobs_status ON processing_jobs(status, priority);
CREATE INDEX idx_jobs_episode ON processing_jobs(episode_id);
```

---

## API & Webhooks

```sql
CREATE TABLE api_tokens (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    token_hash      VARCHAR(64) NOT NULL UNIQUE,
    scopes          TEXT[] NOT NULL DEFAULT '{}',
    expires_at      TIMESTAMPTZ,
    last_used_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE webhooks (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    url             TEXT NOT NULL,
    events          TEXT[] NOT NULL,
    secret          VARCHAR(255) NOT NULL,
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    config          JSONB NOT NULL DEFAULT '{}',
    -- config example: {"retry_count": 3, "timeout_ms": 5000, "headers": {"X-Custom": "value"}}
    last_triggered_at TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## Example Queries

### Find episodes with a specific guest (querying JSONB)

```sql
SELECT e.id, e.title, e.published_at
FROM episodes e
WHERE e.show_id = '{{show_uuid}}'
  AND e.podcast2_meta @> '{"persons": [{"name": "John Smith"}]}'
ORDER BY e.published_at DESC;
```

### Get all chapters for an episode

```sql
SELECT jsonb_array_elements(podcast2_meta->'chapters'->'data') as chapter
FROM episodes
WHERE id = '{{episode_uuid}}';
```

### Find episodes with AI clips scoring above threshold

```sql
SELECT e.id, e.title, clip->>'title' as clip_title, clip->>'shareability_score' as score
FROM episodes e,
     jsonb_array_elements(e.ai_content->'clips') as clip
WHERE e.show_id = '{{show_uuid}}'
  AND (clip->>'shareability_score')::numeric > 0.8
  AND clip->>'status' = 'approved';
```

### Full-text search across transcripts

```sql
SELECT t.episode_id, e.title, ts_headline('english', t.content, q) as headline
FROM transcripts t
JOIN episodes e ON e.id = t.episode_id
CROSS JOIN plainto_tsquery('english', 'artificial intelligence deployment') q
WHERE t.content @@ q
  AND e.show_id = '{{show_uuid}}'
ORDER BY ts_rank(to_tsvector('english', t.content), q) DESC
LIMIT 20;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Identity & Access | 3 | organisations, users, organisation_members |
| Shows | 1 | shows (with JSONB for iTunes, Podcast 2.0, distribution) |
| Episodes | 1 | episodes (with JSONB for iTunes, Podcast 2.0, AI content, workflow) |
| Audio & Media | 1 | audio_files (with JSONB for storage, analysis, ID3) |
| Transcripts | 1 | transcripts (separate for full-text search) |
| Persons | 1 | persons (with JSONB for contact info) |
| Analytics | 2 | analytics_events (partitioned), analytics_daily |
| Processing | 1 | processing_jobs (with JSONB params) |
| API & Integrations | 2 | api_tokens, webhooks |
| **Total** | **13** | Half the table count of the normalized model |

---

## Key Design Decisions

1. **JSONB for namespace-specific metadata** — iTunes and Podcast 2.0 fields live in dedicated JSONB columns. When a new `<podcast:*>` tag is adopted, it is supported by adding a key to the JSON structure rather than running a migration.

2. **Transcripts as a separate table** — despite the JSONB-heavy design, transcripts get their own table because full-text search with GIN indexes on large text content performs best on a dedicated column, and transcripts can be large (100KB+ each).

3. **AI content embedded in episodes** — show notes, clips, and guest research are stored in the episode's `ai_content` JSONB column because they are always accessed in the context of their episode and change frequently as models improve.

4. **Partitioned analytics** — `analytics_events` is range-partitioned by month for time-bounded query performance and efficient data retention (drop old partitions).

5. **GIN indexes on JSONB columns** — enables fast containment queries (`@>`) for filtering episodes by guest name, tag, or AI metadata without full table scans.

6. **Storage abstraction in JSONB** — `audio_files.storage` holds provider-specific fields (S3 bucket/key, GCS bucket/object, local path) without needing a separate table per provider.

7. **Workflow state in JSONB** — episode workflow tracking (stages completed, current stage, processing jobs) is embedded in the episode row for single-query access to the full production state.

8. **Distribution targets in show JSONB** — the list of platforms a show is distributed to (and their approval status) is stored as a JSONB array in the show row, avoiding a join table for what is typically 3-6 entries per show.
