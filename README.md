# Podcast Production Platform

> Part of the [worlds-biggest-software-project](https://github.com/worlds-biggest-software-project) initiative.
>
> An open, AI-native end-to-end platform for recording, editing, transcribing, hosting, and distributing podcasts.

Podcast Production Platform is a candidate open-source alternative to the consolidating SaaS landscape of podcast tools. It targets independent creators, branded podcast producers, internal communications teams, and media companies who need a single workflow that spans recording, editing, show notes, transcription, and distribution — without lock-in to a single vendor.

---

## Why Podcast Production Platform?

- The leading all-in-one tools (Riverside, Descript, Zencastr) are proprietary SaaS at $19–$50/month per seat, and recent acquisitions (Descript and Zencastr by Spotify) signal accelerating platform consolidation.
- Open-source incumbents like Audacity and Ardour are powerful editors but offer no recording-to-distribution workflow, no hosting, no transcription, and no AI automation.
- Specialised tools fragment the workflow: Auphonic handles mastering, Podsqueeze handles repurposing, Buzzsprout handles hosting — creators must stitch them together themselves.
- AI features (show notes, chapters, social clips, filler-word removal) are now table stakes in 2026 but are gated behind paid tiers in every commercial offering.
- No strong open-source option exists for privacy-first, self-hosted podcast production, leaving organisations that avoid SaaS underserved.

---

## Key Features

### Recording and Capture

- Multi-track local recording with separate audio and video per participant
- Lossless capture during recording with upload after the session to preserve fidelity
- Support for video podcasts alongside audio
- Cloud storage with persistent episode library and version control

### Editing and Post-Production

- Timeline editing for trim, cut, mix, and arrange operations
- AI audio enhancement: noise removal, filler-word cleanup, volume balancing
- Loudness normalisation targeting LUFS standards for Apple, Spotify, and YouTube
- Spoken-word-optimised defaults for EQ and levelling
- Transcript-based editing as a longer-term goal

### AI Content Generation

- Automatic transcription with searchable indexing
- AI-generated show notes, chapters, and summaries
- Social clip generation in vertical, square, and landscape formats for TikTok, YouTube Shorts, and Instagram Reels
- Automated metadata: titles, descriptions, episode artwork prompts

### Distribution and Hosting

- RSS feed generation and management
- One-click distribution to Apple Podcasts, Spotify, YouTube, and other major directories
- Podcast 2.0 namespace support for chapters, transcripts, and value4value
- Metadata and tagging for episode title, description, artwork, and guest info

### Analytics and Audience

- Episode downloads, listener geography, and basic engagement tracking
- Guest management with contact details and appearance history
- Backlog: longitudinal theme tracking and engagement-pattern analysis

---

## AI-Native Advantage

The platform treats AI as a core part of the production pipeline rather than a paid add-on. One-click post-production ingests raw audio and applies noise removal, loudness normalisation, silence trimming, and filler-word removal without a human touching a timeline. Show notes and chapters are generated automatically from transcripts in the creator's established voice. AI clip selection identifies the most shareable 30–90 second moments and exports them per-platform. A guest research assistant briefs the host before recording, and cross-episode semantic search lets producers and listeners find any statement across an entire back-catalogue by meaning rather than keyword.

---

## Tech Stack and Deployment

The project is intended to support self-hosted deployment for organisations that need full data ownership, alongside a managed cloud option for creators who want zero-ops. Distribution builds on open standards: RSS 2.0 with the Podcast Namespace, Podcast 2.0 extensions, MP3 and AAC encoding, ID3 tagging, and EBU R128 / ATSC A/85 loudness targets. Integration points include direct publishing to Apple Podcasts, Spotify, and YouTube, plus webhooks and a REST API for custom workflows.

---

## Market Context

The global podcasting market is valued between $32–62 billion in 2026 depending on scope, with the podcast hosting software sub-segment alone estimated at roughly $420 million in 2024 and projected to reach $1.2 billion by 2033 at 12.5% CAGR (Verified Market Reports, 2026). Incumbent pricing for all-in-one tools sits in the $19–$45/month range, with Descript Pro at $35/month and Enterprise at $50/month. Primary buyers are independent creators monetising audiences, branded podcast producers at marketing agencies, internal learning and communications teams, enterprise thought leadership teams, and media companies managing large show catalogues.

---

## Project Status

> This project is in the **research and specification phase**.  
> Contributions, feedback, and domain expertise are welcome.

---

## Contributing

We welcome contributions from developers, domain experts, and potential users.
See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.

**Important:** All contributions must be your own original work or clearly attributed
open-source material with a compatible licence. Copyright infringement and licence
violations will not be tolerated and will result in immediate removal of the offending
contribution. If you are unsure whether a piece of code, text, or other material is
safe to contribute, open an issue and ask before submitting.

---

## Licence

Licence to be determined. See [discussion](#) for context.
