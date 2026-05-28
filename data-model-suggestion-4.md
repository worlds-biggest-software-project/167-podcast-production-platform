# Data Model Suggestion 4: Document-Oriented with Relational Anchors

> Project: Podcast Production Platform · Created: 2026-05-20

## Philosophy

This model inverts the traditional relational approach: the primary data objects are rich, self-contained documents (shows, episodes) stored as JSONB in PostgreSQL, while a thin relational layer provides identity, access control, and cross-document indexing. Each episode document contains everything needed to render that episode's page, generate its RSS item, and serve its API response — in a single row, with a single query.

This pattern draws inspiration from document databases (MongoDB, CouchDB) and headless CMS architectures (Contentful, Sanity), but implemented within PostgreSQL to retain transactional consistency, ACID guarantees, and the ability to JOIN when needed. It is particularly well-suited to a podcast platform because: (a) episodes are the natural "document" — a self-contained unit of content with nested metadata, (b) API responses and RSS feed items map directly to the document structure, (c) content varies significantly between shows (some have video, chapters, transcripts; others have just audio), and (d) the platform serves both a REST API and an RSS feed, both of which benefit from pre-assembled documents.

The key insight is that podcast data is read-heavy and episode-centric. A single episode page or API response needs: the episode metadata, its audio URLs, transcript reference, chapter list, guest list, show notes, and clip data. In a normalized model, this requires 8-10 JOINs. In a document model, it is one row.

**Best for:** Teams building API-first or headless architectures, read-heavy workloads, platforms where episodes are the primary unit of access, and projects that want MongoDB-like flexibility with PostgreSQL's reliability.

**Trade-offs:**
- Pro: Single-query episode retrieval — no JOINs for the most common access pattern
- Pro: API/RSS responses map directly to the document structure
- Pro: Flexible schema accommodates any podcast type (audio, video, live, audiobook)
- Pro: Natural fit for GraphQL or REST API response shaping
- Pro: Simpler caching — cache the document row and serve it directly
- Con: Cross-document queries are expensive (e.g., "find all episodes featuring guest X across all shows")
- Con: Data duplication — guest info repeated in every episode they appear on
- Con: Document size can grow large (10-50KB per episode) affecting table scan performance
- Con: Updates to shared data (guest bio change) must propagate to all referencing documents
- Con: Harder to enforce relational consistency — no FK constraints within documents

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| RSS 2.0 | Episode documents contain all fields needed to render an RSS `<item>` without transformation |
| iTunes Namespace | iTunes-specific fields embedded directly in show and episode documents |
| Podcast 2.0 Namespace | All Podcast 2.0 tags stored as nested objects within episode documents |
| Schema.org PodcastSeries/Episode | Show and episode documents include `schema_org` subobjects ready for JSON-LD rendering |
| EBU R128 | Audio analysis data embedded in audio track objects within episode documents |
| JSON Chapters Format | Chapters stored in the exact JSON Chapters format within the episode document |
| PSP-1 | Feed validation checks run against the document structure at publish time |

---

## Relational Anchors (Thin Layer)

```sql
-- Minimal relational tables for identity, access control, and cross-document indexing

CREATE TABLE organisations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    plan            VARCHAR(50) NOT NULL DEFAULT 'free',
    billing_email   VARCHAR(255),
    settings        JSONB NOT NULL DEFAULT '{}',
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
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE organisation_members (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    role            VARCHAR(50) NOT NULL DEFAULT 'member',
    permissions     JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organisation_id, user_id)
);

CREATE INDEX idx_org_members_org ON organisation_members(organisation_id);
CREATE INDEX idx_org_members_user ON organisation_members(user_id);
```

---

## Show Documents

```sql
CREATE TABLE shows (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id     UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    -- Indexed relational fields for listing/filtering
    title               VARCHAR(500) NOT NULL,
    slug                VARCHAR(200) NOT NULL,
    status              VARCHAR(20) NOT NULL DEFAULT 'draft',
    language            VARCHAR(10) NOT NULL DEFAULT 'en',
    episode_count       INTEGER NOT NULL DEFAULT 0,
    latest_episode_at   TIMESTAMPTZ,
    -- The full show document
    document            JSONB NOT NULL DEFAULT '{}',
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organisation_id, slug)
);

CREATE INDEX idx_shows_org ON shows(organisation_id);
CREATE INDEX idx_shows_status ON shows(status);
CREATE INDEX idx_shows_doc ON shows USING gin(document);
```

### Show Document Structure

```jsonc
// shows.document example:
{
  "description": "A weekly show about building software with AI...",
  "summary": "Short summary for podcast directories",
  "author": "Jane Doe",
  "copyright": "Copyright 2026 Jane Doe",
  "artwork": {
    "url": "https://cdn.example.com/shows/dev-podcast/artwork-3000.jpg",
    "width": 3000,
    "height": 3000
  },
  "website_url": "https://thedevpodcast.com",

  // iTunes namespace
  "itunes": {
    "owner": {"name": "Jane Doe", "email": "jane@example.com"},
    "categories": [
      {"category": "Technology", "subcategory": "Tech News"},
      {"category": "Business", "subcategory": "Entrepreneurship"}
    ],
    "type": "episodic",
    "explicit": false,
    "complete": false
  },

  // Podcast 2.0 namespace
  "podcast2": {
    "guid": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
    "medium": "podcast",
    "locked": false,
    "lock_owner": "jane@example.com",
    "funding": [
      {"url": "https://patreon.com/thedevpodcast", "message": "Support us on Patreon!"}
    ],
    "value": {
      "type": "lightning",
      "method": "keysend",
      "recipients": [
        {"name": "Host", "type": "wallet", "address": "abc123", "split": 90, "fee": false},
        {"name": "Editor", "type": "wallet", "address": "def456", "split": 10, "fee": false}
      ]
    },
    "podroll": [
      {"feed_url": "https://feeds.example.com/related-show.xml", "title": "Related Show"}
    ],
    "update_frequency": "weekly",
    "publisher": {"name": "Dev Media LLC"}
  },

  // Distribution targets
  "distribution": {
    "rss_feed_url": "https://feeds.example.com/the-dev-podcast.xml",
    "rss_built_at": "2026-05-20T10:00:00Z",
    "targets": [
      {"platform": "apple_podcasts", "id": "1234567890", "status": "approved",
       "url": "https://podcasts.apple.com/podcast/id1234567890", "submitted_at": "2026-01-15T00:00:00Z"},
      {"platform": "spotify", "id": "abc123", "status": "approved",
       "url": "https://open.spotify.com/show/abc123"},
      {"platform": "youtube", "id": "UC_xyz", "status": "pending"}
    ]
  },

  // Schema.org structured data (ready for JSON-LD)
  "schema_org": {
    "@type": "PodcastSeries",
    "name": "The Dev Podcast",
    "description": "A weekly show about building software with AI...",
    "webFeed": "https://feeds.example.com/the-dev-podcast.xml",
    "numberOfEpisodes": 42,
    "genre": "Technology"
  },

  // Team (denormalised from organisation_members for display)
  "team": [
    {"user_id": "uuid", "name": "Jane Doe", "role": "owner", "avatar_url": "https://..."},
    {"user_id": "uuid", "name": "Bob Editor", "role": "editor", "avatar_url": "https://..."}
  ]
}
```

---

## Episode Documents

```sql
CREATE TABLE episodes (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    show_id             UUID NOT NULL REFERENCES shows(id) ON DELETE CASCADE,
    organisation_id     UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    -- Indexed relational fields for listing/filtering/sorting
    title               VARCHAR(500) NOT NULL,
    slug                VARCHAR(200) NOT NULL,
    episode_number      INTEGER,
    season_number       INTEGER,
    duration_seconds    INTEGER,
    published_at        TIMESTAMPTZ,
    status              VARCHAR(20) NOT NULL DEFAULT 'draft',
    created_by          UUID REFERENCES users(id),
    -- The full episode document
    document            JSONB NOT NULL DEFAULT '{}',
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (show_id, slug)
);

CREATE INDEX idx_episodes_show ON episodes(show_id);
CREATE INDEX idx_episodes_org ON episodes(organisation_id);
CREATE INDEX idx_episodes_status ON episodes(status);
CREATE INDEX idx_episodes_published ON episodes(published_at DESC);
CREATE INDEX idx_episodes_doc ON episodes USING gin(document);
-- Partial index for published episodes (most common query)
CREATE INDEX idx_episodes_published_only ON episodes(show_id, published_at DESC) WHERE status = 'published';
```

### Episode Document Structure

```jsonc
// episodes.document example:
{
  "description": "<p>In this episode, we explore how AI is transforming...</p>",
  "summary": "Plain text summary for podcast apps",
  "artwork_url": "https://cdn.example.com/episodes/ep42/artwork.jpg",

  // iTunes metadata
  "itunes": {
    "episode_type": "full",
    "explicit": false,
    "block": false
  },

  // Primary enclosure (for RSS <enclosure> tag)
  "enclosure": {
    "url": "https://cdn.example.com/episodes/ep42/episode.mp3",
    "type": "audio/mpeg",
    "length": 48000000
  },

  // All audio tracks (raw and processed)
  "audio_tracks": [
    {
      "id": "uuid-raw-host",
      "filename": "host-track.wav",
      "mime_type": "audio/wav",
      "file_size_bytes": 145000000,
      "duration_seconds": 3600.5,
      "codec": "pcm",
      "track_label": "Host",
      "is_original": true,
      "storage": {"provider": "s3", "bucket": "podcast-raw", "key": "org/ep42/host.wav"},
      "audio_analysis": {
        "sample_rate": 48000, "channels": 1, "bit_rate": 768000,
        "lufs_integrated": -18.5, "lufs_true_peak": -3.2
      }
    },
    {
      "id": "uuid-raw-guest",
      "filename": "guest-track.wav",
      "mime_type": "audio/wav",
      "file_size_bytes": 140000000,
      "duration_seconds": 3600.5,
      "codec": "pcm",
      "track_label": "Guest",
      "is_original": true,
      "storage": {"provider": "s3", "bucket": "podcast-raw", "key": "org/ep42/guest.wav"},
      "audio_analysis": {
        "sample_rate": 48000, "channels": 1, "bit_rate": 768000,
        "lufs_integrated": -20.1, "lufs_true_peak": -2.8
      }
    },
    {
      "id": "uuid-final",
      "filename": "episode-42-final.mp3",
      "mime_type": "audio/mpeg",
      "file_size_bytes": 48000000,
      "duration_seconds": 3601,
      "codec": "mp3",
      "track_label": "Final Mix",
      "is_original": false,
      "source_track_ids": ["uuid-raw-host", "uuid-raw-guest"],
      "storage": {"provider": "s3", "bucket": "podcast-cdn", "key": "org/ep42/episode.mp3",
                  "cdn_url": "https://cdn.example.com/episodes/ep42/episode.mp3"},
      "audio_analysis": {
        "sample_rate": 44100, "channels": 2, "bit_rate": 192000,
        "lufs_integrated": -16.0, "lufs_true_peak": -1.1, "lufs_range": 7.5
      }
    }
  ],

  // Podcast 2.0 metadata
  "podcast2": {
    "transcript": {
      "url": "https://cdn.example.com/episodes/ep42/transcript.vtt",
      "type": "text/vtt",
      "language": "en"
    },
    "chapters": {
      "url": "https://cdn.example.com/episodes/ep42/chapters.json",
      "type": "application/json+chapters",
      "data": [
        {"startTime": 0, "title": "Intro"},
        {"startTime": 60, "title": "Guest Introduction", "img": "https://..."},
        {"startTime": 180, "title": "AI in Production", "url": "https://link-to-resource.com"},
        {"startTime": 1200, "title": "Lightning Round"},
        {"startTime": 3480, "title": "Outro"}
      ]
    },
    "soundbites": [
      {"startTime": 1200.5, "duration": 60, "title": "Key insight on AI deployment"}
    ],
    "persons": [
      {"name": "Jane Doe", "role": "host", "group": "cast",
       "img": "https://cdn.example.com/people/jane.jpg", "href": "https://janedoe.com"},
      {"name": "John Smith", "role": "guest", "group": "cast",
       "img": "https://cdn.example.com/people/john.jpg", "href": "https://johnsmith.dev"}
    ],
    "location": {"geo": "geo:37.7749,-122.4194", "osm": "R123456", "name": "San Francisco, CA"},
    "alternate_enclosures": [
      {"type": "audio/opus", "length": 12000000, "bitrate": 96000,
       "source": [{"uri": "https://cdn.example.com/episodes/ep42/episode.opus"}]},
      {"type": "video/mp4", "length": 500000000, "height": 1080,
       "source": [{"uri": "https://cdn.example.com/episodes/ep42/episode.mp4"}]}
    ]
  },

  // AI-generated content
  "ai": {
    "show_notes": {
      "summary": "This episode explores how AI is changing production workflows...",
      "key_points": [
        "AI reduces post-production time by 80%",
        "Loudness normalization is now fully automated",
        "Transcript-based editing is the future"
      ],
      "keywords": ["AI", "podcast production", "automation", "transcription"],
      "model": "gpt-4o",
      "generated_at": "2026-05-20T09:00:00Z",
      "approved": true,
      "approved_by": "uuid"
    },
    "clips": [
      {
        "id": "uuid-clip-1",
        "start_ms": 1200000,
        "end_ms": 1260000,
        "title": "Hot take on AI deployment",
        "description": "Jane shares her controversial view...",
        "shareability_score": 0.87,
        "format": "vertical",
        "platform_target": "tiktok",
        "audio_url": "https://cdn.example.com/episodes/ep42/clips/clip-1.mp4",
        "status": "approved"
      },
      {
        "id": "uuid-clip-2",
        "start_ms": 2400000,
        "end_ms": 2480000,
        "title": "The future of podcast editing",
        "shareability_score": 0.72,
        "format": "square",
        "platform_target": "instagram_reels",
        "status": "suggested"
      }
    ],
    "guest_research": {
      "person_name": "John Smith",
      "summary": "John is a senior ML engineer at...",
      "recent_work": ["Published paper on...", "Gave talk at..."],
      "suggested_questions": [
        "How has your approach to model deployment changed since...",
        "What surprised you most about..."
      ],
      "sources": ["https://arxiv.org/...", "https://johnsmith.dev/blog/..."],
      "model": "gpt-4o",
      "generated_at": "2026-05-19T14:00:00Z"
    }
  },

  // Workflow tracking
  "workflow": {
    "current_stage": "published",
    "stages": [
      {"stage": "drafted", "at": "2026-05-18T10:00:00Z", "by": "uuid"},
      {"stage": "tracks_uploaded", "at": "2026-05-18T11:00:00Z", "by": "uuid"},
      {"stage": "processing", "at": "2026-05-18T11:01:00Z", "by": "system"},
      {"stage": "processed", "at": "2026-05-18T11:15:00Z", "by": "system"},
      {"stage": "transcribed", "at": "2026-05-18T11:20:00Z", "by": "system"},
      {"stage": "chapters_generated", "at": "2026-05-18T11:21:00Z", "by": "system"},
      {"stage": "show_notes_generated", "at": "2026-05-18T11:22:00Z", "by": "system"},
      {"stage": "reviewed", "at": "2026-05-19T09:00:00Z", "by": "uuid"},
      {"stage": "published", "at": "2026-05-20T10:00:00Z", "by": "uuid"}
    ],
    "processing_jobs": [
      {"id": "uuid-job1", "type": "noise_removal", "status": "complete", "completed_at": "2026-05-18T11:05:00Z"},
      {"id": "uuid-job2", "type": "loudness_norm", "status": "complete", "completed_at": "2026-05-18T11:10:00Z"},
      {"id": "uuid-job3", "type": "transcode_mp3", "status": "complete", "completed_at": "2026-05-18T11:15:00Z"},
      {"id": "uuid-job4", "type": "transcribe", "status": "complete", "completed_at": "2026-05-18T11:20:00Z"}
    ]
  }
}
```

---

## Cross-Document Indexes

```sql
-- Search index for finding episodes by guest across all shows
CREATE TABLE person_episode_index (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    person_name     VARCHAR(255) NOT NULL,
    person_role     VARCHAR(100) NOT NULL,
    episode_id      UUID NOT NULL REFERENCES episodes(id) ON DELETE CASCADE,
    show_id         UUID NOT NULL,
    organisation_id UUID NOT NULL,
    published_at    TIMESTAMPTZ
);

CREATE INDEX idx_person_ep_name ON person_episode_index(person_name);
CREATE INDEX idx_person_ep_org ON person_episode_index(organisation_id, person_name);

-- Transcript storage for full-text search (extracted from documents)
CREATE TABLE transcript_search (
    episode_id      UUID PRIMARY KEY REFERENCES episodes(id) ON DELETE CASCADE,
    show_id         UUID NOT NULL,
    organisation_id UUID NOT NULL,
    language        VARCHAR(10) NOT NULL DEFAULT 'en',
    content         TEXT NOT NULL,
    content_tsv     TSVECTOR GENERATED ALWAYS AS (to_tsvector('english', content)) STORED
);

CREATE INDEX idx_transcript_search_fts ON transcript_search USING gin(content_tsv);
CREATE INDEX idx_transcript_search_org ON transcript_search(organisation_id);
```

---

## Analytics (Separate from Documents)

```sql
-- Analytics stay relational — they're write-heavy and time-series in nature
CREATE TABLE analytics_events (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    episode_id      UUID NOT NULL,
    show_id         UUID NOT NULL,
    event_type      VARCHAR(50) NOT NULL,
    event_data      JSONB NOT NULL DEFAULT '{}',
    occurred_at     TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (occurred_at);

CREATE TABLE analytics_events_2026_05 PARTITION OF analytics_events
    FOR VALUES FROM ('2026-05-01') TO ('2026-06-01');
CREATE TABLE analytics_events_2026_06 PARTITION OF analytics_events
    FOR VALUES FROM ('2026-06-01') TO ('2026-07-01');

CREATE INDEX idx_analytics_ep ON analytics_events(episode_id, occurred_at);
CREATE INDEX idx_analytics_show ON analytics_events(show_id, occurred_at);

CREATE TABLE analytics_daily (
    show_id         UUID NOT NULL,
    episode_id      UUID NOT NULL,
    date            DATE NOT NULL,
    stats           JSONB NOT NULL DEFAULT '{}',
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
    job_type        VARCHAR(50) NOT NULL,
    status          VARCHAR(20) NOT NULL DEFAULT 'queued',
    priority        INTEGER NOT NULL DEFAULT 5,
    params          JSONB NOT NULL DEFAULT '{}',
    result          JSONB,
    error_message   TEXT,
    started_at      TIMESTAMPTZ,
    completed_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_jobs_status ON processing_jobs(status, priority);
CREATE INDEX idx_jobs_episode ON processing_jobs(episode_id);
```

---

## API Tokens

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
```

---

## Example Queries

### Get a full episode for API response (single query)

```sql
SELECT id, title, slug, episode_number, season_number,
       duration_seconds, published_at, status, document
FROM episodes
WHERE id = '{{episode_uuid}}'
  AND organisation_id = '{{org_uuid}}';
-- The document column contains everything: audio tracks, chapters,
-- transcript reference, persons, AI content, workflow state.
-- The API handler returns document directly with minimal transformation.
```

### Generate RSS feed (single query per show)

```sql
SELECT e.title, e.episode_number, e.season_number, e.published_at,
       e.duration_seconds, e.document
FROM episodes e
WHERE e.show_id = '{{show_uuid}}'
  AND e.status = 'published'
ORDER BY e.published_at DESC;
-- Each row's document contains enclosure, itunes, podcast2 fields
-- ready for XML template rendering.
```

### Cross-show guest search (uses index table)

```sql
SELECT pei.person_name, pei.person_role, e.title, e.published_at, s.title as show_title
FROM person_episode_index pei
JOIN episodes e ON e.id = pei.episode_id
JOIN shows s ON s.id = pei.show_id
WHERE pei.organisation_id = '{{org_uuid}}'
  AND pei.person_name ILIKE '%John Smith%'
ORDER BY e.published_at DESC;
```

### Semantic search across transcripts

```sql
SELECT ts.episode_id, e.title, e.published_at,
       ts_headline('english', ts.content, q, 'MaxWords=30') as snippet
FROM transcript_search ts
JOIN episodes e ON e.id = ts.episode_id
CROSS JOIN websearch_to_tsquery('english', 'deploying AI models production') q
WHERE ts.organisation_id = '{{org_uuid}}'
  AND ts.content_tsv @@ q
ORDER BY ts_rank(ts.content_tsv, q) DESC
LIMIT 20;
```

### Find episodes with unapproved AI clips

```sql
SELECT id, title, published_at,
       jsonb_path_query_array(document, '$.ai.clips[*] ? (@.status == "suggested")') as pending_clips
FROM episodes
WHERE organisation_id = '{{org_uuid}}'
  AND document @> '{"ai": {"clips": [{"status": "suggested"}]}}';
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Identity & Access | 3 | organisations, users, organisation_members |
| Show Documents | 1 | shows (with rich JSONB document) |
| Episode Documents | 1 | episodes (with rich JSONB document) |
| Cross-Document Indexes | 2 | person_episode_index, transcript_search |
| Analytics | 2 | analytics_events (partitioned), analytics_daily |
| Processing | 1 | processing_jobs |
| API | 1 | api_tokens |
| **Total** | **11** | Lowest table count; complexity is in document structure |

---

## Key Design Decisions

1. **Episodes as self-contained documents** — the `document` JSONB column contains all audio tracks, chapters, transcripts, persons, AI content, and workflow state. This eliminates multi-table JOINs for the most common access pattern (viewing/serving an episode).

2. **Relational columns for queryable attributes** — `title`, `status`, `published_at`, `episode_number`, `season_number` are relational columns because they are filtered, sorted, and indexed. Everything else lives in the document.

3. **Explicit cross-document indexes** — since JSONB containment queries across thousands of episodes are slow for cross-cutting concerns (find all episodes with guest X), dedicated index tables (`person_episode_index`, `transcript_search`) are maintained by application triggers or async workers.

4. **Audio tracks embedded in episode documents** — raw tracks, processed versions, and final mixes are all nested objects within the episode document. This means the full audio provenance chain is visible in a single query. A separate `audio_files` table is NOT used.

5. **Transcript content in a separate search table** — while the transcript reference lives in the episode document, the full transcript text is stored in `transcript_search` with a generated TSVECTOR column for efficient full-text search.

6. **Analytics remain relational and partitioned** — analytics events are high-volume, time-series, and write-heavy. They do not belong in documents and benefit from range partitioning.

7. **Workflow history in the document** — the complete production timeline (drafted, uploaded, processed, transcribed, reviewed, published) is stored as an array in the episode document, providing a built-in audit trail visible in every API response.

8. **Document update patterns** — PostgreSQL's `jsonb_set()` and `||` operators enable targeted updates to specific parts of the document without rewriting the entire JSONB value, though bulk document rewrites are needed for structural changes.
