# Standards & API Reference

> Project: Podcast Production Platform · Generated: 2026-05-03

## Industry Standards & Specifications

### ISO Standards

- **ISO/IEC 27001:2022** — Information security management; governs access controls and encryption for podcast production platforms handling proprietary audio content, unreleased episodes, and creator account data. URL: https://www.iso.org/standard/82875.html

- **ISO/IEC 14496-3 — MPEG-4 Audio** — ISO standard for AAC audio encoding (Advanced Audio Coding); AAC-LC and AAC-HE are the standard audio codecs for podcast delivery; M4A/MP4 containers use ISO BMFF (ISO 14496-12) packaging. URL: https://www.iso.org/standard/76383.html

### W3C & IETF Standards

- **RSS 2.0 — Really Simple Syndication** — The foundational XML syndication format for podcast feed distribution; all podcast directories (Apple Podcasts, Spotify, Amazon Music, Google Podcasts) consume RSS 2.0 feeds; the `<enclosure>` element carries the audio/video file URL, type (audio/mpeg, audio/x-m4a), and file size. URL: https://www.rssboard.org/rss-specification

- **iTunes Podcast Namespace (`itunes:`)** — Apple's de facto podcast RSS extension namespace (`http://www.itunes.com/dtds/podcast-1.0.dtd`); defines `<itunes:author>`, `<itunes:category>`, `<itunes:image>`, `<itunes:explicit>`, `<itunes:duration>`, `<itunes:episode>`, `<itunes:season>`, and `<itunes:type>`; required by Apple Podcasts and adopted as the universal podcast tagging standard. URL: https://podcasters.apple.com/support/823-podcast-requirements

- **Podcast Namespace (Podcasting 2.0)** — Open-source (`podcast:`) namespace collaboratively developed via the Podcast Standards Project; adds `<podcast:transcript>`, `<podcast:chapters>`, `<podcast:soundbite>`, `<podcast:person>`, `<podcast:location>`, `<podcast:value>` (Value4Value/Lightning), `<podcast:liveItem>`, and 40+ additional tags; used by Podcast Index ecosystem apps. URL: https://podcasting2.org/docs/podcast-namespace

- **Podcast Standards Project PSP-1** — Community-maintained specification for podcast RSS feeds, standardising the combination of RSS 2.0, iTunes namespace, and Podcast namespace requirements for interoperability across all major podcast directories. URL: https://github.com/Podcast-Standards-Project/PSP-1-Podcast-RSS-Specification

- **Atom Namespace** — RSS feeds must include the `atom:link` self-referencing tag (`http://www.w3.org/2005/Atom`) for feed URL discovery and canonical URL declaration; required by Apple Podcasts. URL: https://www.w3.org/2005/Atom

- **RFC 8216 — HLS: HTTP Live Streaming** — Apple adopted HLS (iOS 26.4, March 2026) as the primary delivery mechanism for video podcasts in Apple Podcasts, replacing the traditional RSS `<enclosure>` method for video content; audio podcasts continue using direct MP3/AAC enclosures but video podcasts must now provide HLS streams. URL: https://datatracker.ietf.org/doc/html/rfc8216

- **RFC 6749 — OAuth 2.0** — Authorization framework used by podcast hosting platforms (Transistor, Buzzsprout) for third-party app authorization and API integrations with podcast directories and analytics services. URL: https://datatracker.ietf.org/doc/html/rfc6749

- **RFC 7519 — JSON Web Token (JWT)** — Used in podcast platform API authentication tokens and private podcast subscriber access token generation. URL: https://datatracker.ietf.org/doc/html/rfc7519

### Data Model & API Specifications

- **OpenAPI 3.1** — Used by Podcast Index, Transistor, and Descript to describe their REST APIs; enables SDK code generation and automated integration testing. URL: https://spec.openapis.org/oas/latest.html

- **Podcast Index API** — Free, open podcast search and directory API (documented using OpenAPI syntax) providing podcast search, episode lookup, feed data, chapters data, and live item information; the open alternative to Spotify and Apple podcast data APIs. URL: https://podcastindex-org.github.io/docs-api/

- **MP3 / MPEG-1 Audio Layer III** — The universal podcast audio format; codec standard (MPEG-1 Layer 3, ISO 11172-3); virtually all podcast apps and directories accept MP3 as the primary audio format for maximum compatibility.

- **ID3 Tags (ID3v2.4)** — De facto standard for embedding metadata in MP3 files; used to store episode title, podcast name, artist, album art, episode number, and description directly in the audio file; the Podcast Standards Project recommends specific ID3 tag usage for podcast episodes.

- **WebVTT / SRT Transcripts** — W3C WebVTT and SRT formats are the standard for podcast transcript files referenced by `<podcast:transcript>` tags; Apple Podcasts and major podcast apps render interactive transcripts from these files.

- **JSON Chapters Format** — The `<podcast:chapters>` tag references a JSON chapters file (podcast chapter format specification); defines chapter start time, title, image, and URL for interactive chapter navigation in supporting apps. URL: https://github.com/Podcastindex-org/podcast-namespace/blob/main/chapters/jsonChapters.md

### Security & Authentication Standards

- **GDPR Article 6 & 32 — Lawful Processing and Security of Processing** — Podcast production platforms processing creator and listener personal data must have lawful basis; podcast analytics (listener geolocation, device data, consumption behaviour) require appropriate data handling and privacy notices. URL: https://gdpr-info.eu/art-32-gdpr/

- **COPPA — Children's Online Privacy Protection Act** — Podcasts marked `<itunes:explicit>clean</itunes:explicit>` targeted at children must comply with COPPA requirements for any listener analytics or interactive features. URL: https://www.ftc.gov/legal-library/browse/rules/childrens-online-privacy-protection-rule-coppa

- **SOC 2 Type II** — Required enterprise compliance certification for SaaS podcast production platforms; Buzzsprout, Transistor, and RSS.com maintain compliance certifications for enterprise customers. URL: https://www.aicpa-cima.com/topic/audit-assurance/audit-and-assurance-greater-than-soc-2

- **OWASP API Security Top 10 (2023)** — Governs REST API security for podcast management APIs; API1 (Broken Object Level Authorization) is critical for ensuring creators can only access their own episodes and analytics. URL: https://owasp.org/API-Security/

- **Value4Value / Lightning Network** — Bitcoin Lightning Network micropayment standard used by `<podcast:value>` tag for listener-to-creator direct value streaming; not a traditional security standard but a payment protocol enabling novel creator monetisation. URL: https://podcasting2.org/docs/podcast-namespace

### MCP Server Specifications

Podcast production platforms are integrating with the MCP ecosystem for AI-assisted workflow automation:

- **Transistor.fm MCP Server** — Community MCP server enabling AI agents to automate podcast workflows via the Transistor API; create episodes, update show notes, manage subscribers, and retrieve analytics through AI-driven interactions. URL: https://skywork.ai/skypage/en/transistor-mcp-ai-podcast-workflows/1981609287047286784

- **AI Podcast Production Pattern (2025-2026)** — Emerging architecture: AI agents access recording transcripts (via Whisper or platform-native ASR), generate episode summaries and show notes, create chapter markers, and publish episodes via podcast hosting API — with MCP servers providing the integration layer between AI tools and podcast platforms.

---

## Similar Products — Developer Documentation & APIs

### Transistor

- **Description:** Professional podcast hosting platform with unlimited shows per account; REST API v1 for managing shows, episodes, subscribers, and analytics; popular with agencies and multi-show podcasters; Transistor MCP server available.
- **API Documentation:** https://developers.transistor.fm/
- **Developer Guide:** https://transistor.fm/changelog/api-v1/
- **SDKs/Libraries:** REST API (JSON); JavaScript community SDK; MCP server (community)
- **Standards:** REST/JSON, OpenAPI, OAuth 2.0 (Personal Access Token), RSS 2.0 + iTunes namespace feed generation
- **Authentication:** Personal Access Token (x-api-key header)

### Buzzsprout

- **Description:** Leading beginner-friendly podcast hosting platform; one-click distribution to Apple Podcasts, Spotify, YouTube, and all major directories; REST API for episode and analytics management; strong WordPress and Squarespace integrations.
- **API Documentation:** https://www.buzzsprout.com/api (requires account)
- **SDKs/Libraries:** REST API (JSON); Ruby gem (buzzsprout-api); Zapier integration
- **Developer Guide:** Buzzsprout developer portal (account required)
- **Standards:** REST/JSON, OpenAPI, API token auth, RSS 2.0 + iTunes namespace feed generation
- **Authentication:** API token (per podcast)

### Spotify for Creators (formerly Anchor)

- **Description:** Free podcast hosting from Spotify with built-in distribution to Spotify and other directories; Partner Program (January 2026: 1,000 engaged listeners + 2,000 consumption hours for ad revenue sharing); video podcast support; limited API access for creators.
- **API Documentation:** No public REST API (closed platform); Spotify Web API provides podcast data for listeners
- **SDKs/Libraries:** Spotify Web API (for podcast search/playback, not creation); no creator API
- **Developer Guide:** https://developer.spotify.com/documentation/web-api
- **Standards:** Spotify Web API (REST/JSON), OAuth 2.0, RSS 2.0 feed (internal)
- **Authentication:** Spotify OAuth 2.0 (Web API for listener data only)

### Podcast Index API (Open Source)

- **Description:** Free, open podcast search and discovery API maintained by the Podcast Index Foundation; indexes millions of podcasts from public RSS feeds; provides search, episode data, chapters, transcripts, live items, and feed validation; the open alternative to commercial podcast data APIs.
- **API Documentation:** https://podcastindex-org.github.io/docs-api/
- **Developer Docs:** https://api.podcastindex.org/developer_docs
- **SDKs/Libraries:** podindexr (R package); python-podcastindex; podcastindex-node; multiple community clients
- **Developer Guide:** https://github.com/Podcastindex-org/docs-api
- **Standards:** REST/JSON, OpenAPI (Swagger), HMAC-SHA1 authentication, RSS 2.0 parsing
- **Authentication:** API key + secret + HMAC-SHA1 request signing

### Descript

- **Description:** AI-powered podcast and video production platform with text-based audio/video editing (edit transcript to edit media); multitrack recording, automated transcription, filler word removal, and Studio Sound AI noise reduction; public API for partner platform integrations.
- **API Documentation:** https://www.descript.com/developers (requires account)
- **SDKs/Libraries:** REST API; "Edit in Descript" partner integration API
- **Developer Guide:** Descript developer portal
- **Standards:** REST/JSON, OpenAPI, OAuth 2.0
- **Authentication:** OAuth 2.0; API token for partner integrations

### Riverside.fm

- **Description:** Professional remote podcast and video recording platform with studio-quality separate track recording (local recording, no packet loss); Business API for programmatic access to recordings and studio workflow data; strong for interview-format podcasts.
- **API Documentation:** https://riverside.fm/developers (requires account)
- **SDKs/Libraries:** Business API (REST/JSON); Webhook events for recording completion
- **Developer Guide:** Riverside developer portal
- **Standards:** REST/JSON, OpenAPI, OAuth 2.0, Webhooks
- **Authentication:** API key; OAuth 2.0 for partner integrations

### Auphonic (Open Source / API)

- **Description:** AI-powered audio post-production service; automatic loudness normalisation (to EBU R128 / -16 LUFS), noise reduction, chapter marks, and transcript generation; REST API for batch audio processing; used as a post-production step in podcast workflow automation.
- **API Documentation:** https://auphonic.com/api
- **SDKs/Libraries:** auphonic-api (Python); REST API (JSON); command-line client
- **Developer Guide:** https://auphonic.com/api
- **Standards:** REST/JSON, OpenAPI, OAuth 2.0, EBU R128 loudness normalisation, MP3/AAC/FLAC/OGG input formats
- **Authentication:** HTTP Basic Auth (username/password); OAuth 2.0

### RSS.com

- **Description:** Podcast hosting platform with a focus on distribution and analytics; REST API for podcast management; supports all major directories; compliance certifications for enterprise; integrated podcast website builder.
- **API Documentation:** https://rss.com/api/ (requires account)
- **SDKs/Libraries:** REST API (JSON); Zapier integration; Webhook events
- **Developer Guide:** RSS.com developer portal
- **Standards:** REST/JSON, OpenAPI, OAuth 2.0, RSS 2.0 + iTunes + Podcast namespace feed generation
- **Authentication:** API key; OAuth 2.0 for integrations

---

## Notes

- **Apple Podcasts HLS video standard (iOS 26.4, March 2026)**: Apple adopted HLS as the primary delivery mechanism for video podcasts in Apple Podcasts with iOS 26.4 (March 2026); video podcast producers must now provide HLS streams rather than direct MP4 enclosures for optimal Apple Podcasts delivery; audio-only podcasts continue using MP3/AAC enclosures.

- **Podcasting 2.0 namespace adoption**: The `podcast:` namespace (podcasting2.org) has seen growing adoption in 2025-2026 with Podcast Index ecosystem apps; key tags include `<podcast:transcript>` (searchable transcripts), `<podcast:chapters>` (interactive chapters), `<podcast:person>` (guest tagging), and `<podcast:value>` (Value4Value micropayments); adoption varies across hosting platforms.

- **EBU R128 loudness standard**: Professional podcast audio should target -16 LUFS integrated loudness (based on EBU R128 / ITU-R BS.1770); Apple Podcasts and Spotify normalise audio to approximately -16 LUFS; Auphonic automates this normalisation in post-production.

- **Podcast Standards Project (PSP-1)**: The community-maintained PSP-1 specification provides the definitive combined standard for podcast RSS feeds, bringing together RSS 2.0, iTunes namespace, and Podcast namespace requirements; new podcast platforms should validate feeds against PSP-1.

- **Value4Value / Lightning Network**: The `<podcast:value>` tag enables listener-to-creator micropayments via Bitcoin Lightning Network (streaming sats); while niche, it represents a novel monetisation model gaining traction in Podcasting 2.0 apps (Fountain, Breez, Castamatic).

- **Open-source landscape**: There is no dominant open-source full-stack podcast production platform; open-source components include ffmpeg (LGPL, audio/video processing), Whisper (MIT, transcription), Auphonic (commercial API with free tier), and the Podcast Index API (open/free) for podcast discovery.
