# Implementation plan — GCP PCA 2026 Study Handbook

## Goal
One well-formatted Markdown file, `GCP-PCA-2026-Handbook.md`, with a clickable table of contents, 16 topic chapters, and a comparison-heavy cheat sheet at the end. It must cover the full exam guide v6.1 (live since 30 Oct 2025), including the new AI sections (2.4, 2.5, Securing AI, Gemini Cloud Assist) and the Well-Architected Framework.

## Research findings that shape the plan
| Finding | Source | Impact |
|---|---|---|
| Standard exam: 2 h, 50–60 Q, 2 case studies (20–30 %) from a pool of 4: Altostrat Media, Cymbal Retail, EHR Healthcare, KnightMotives | cloud.google.com cert page, exam guide PDF | Dedicated case-study chapter; every chapter has "case-study hooks" |
| Weights: S1 25 %, S2 17.5 %, S3 17.5 %, S4 15 %, S5 12.5 %, S6 12.5 % | exam guide | Chapter depth follows weight |
| New: WAF (6 pillars), 2.4 Agent Platform ML workflows + AI Hypercomputer, 2.5 prebuilt AI APIs / Gemini Enterprise / Model Garden, Securing AI (Model Armor, SDP), Terraform, Migration Center, Direct VPC egress, PSC | exam guide, community posts | Two AI chapters, AI angle section in all chapters |
| Vertex AI was rebranded "Gemini Enterprise Agent Platform" (Apr 2026); exam guide uses new names | Google release notes, Cloud Next 2026 coverage | Glossary table in the brief; old and new names used everywhere |
| Community: study less App Engine, gcloud syntax, lift-and-shift; more Cloud Run, Cloud Deploy, AlloyDB, VPC-SC, Migration Center, VMware Engine, Backup and DR | gcpstudyhub, Reddit, Medium | "Exam traps" and priority notes |
| The linked Reddit thread could not be fetched (bot wall) | Tavily | Used related threads and write-ups instead; flagged in the brief |

## Architecture of the build
- **One agent per chapter**, `general-purpose` subagent with `model: haiku`.
- **Shared tmp file**: `c:\tmp\pca-handbook\SHARED.md`.
  - Sections 1–7: read-only brief (syllabus verbatim, rebrand glossary, case-study digests, community intel, chapter ownership, rules).
  - Section 8: append-only message board (`STARTED / NOTE / QUESTION / FACT / CONFLICT / XREF / DONE`). Agents append with `cat >>`, never overwrite.
- **Chapter output**: `handbook/chNN-*.md`, with a fixed 11-section template so merging is mechanical.
- **Phase 1** (parallel): ch01–ch16. **Phase 2**: ch17 cheat sheet, which reads all finished chapters.
- **Phase 3** (orchestrator): merge, TOC, QA, fix.

## Chapters
| Ch | Title | Exam guide refs |
|---|---|---|
| 01 | Exam overview, strategy, Well-Architected Framework, 8-week plan | intro, WAF |
| 02 | Requirements, HA and DR design | 1.1, 1.2, 4.1 DR |
| 03 | Networking | 1.3, 2.1 |
| 04 | Compute | 1.3, 2.3 |
| 05 | Storage and databases | 1.3, 2.2 |
| 06 | Data processing and integration | 1.1, 1.3 |
| 07 | Migration | 1.4, 1.5 |
| 08 | AI: Agent Platform (ex-Vertex AI), Gemini, Model Garden, agents, prebuilt APIs | 2.4, 2.5, 1.3 |
| 09 | AI infrastructure (AI Hypercomputer, GPU/TPU) and Gemini assist tools | 2.4, 1.2, 5.1 |
| 10 | IAM and resource hierarchy | 3.1 |
| 11 | Data security, controls, supply chain, Securing AI | 3.1 |
| 12 | Compliance and governance | 3.2 |
| 13 | CI/CD, DevOps, IaC, APIs | 4.1, 5.1, 5.2 |
| 14 | Business processes and FinOps | 4.2 |
| 15 | Operations and reliability | S6 |
| 16 | Case studies | all |
| 17 | Cheat sheet: comparisons, decision trees, traps, 40 practice questions | all |

## Per-chapter template (enforced)
Exam objectives covered → What the exam really tests → Core concepts → Decision guides → WAF lens → AI/GenAI angle → Case-study hooks → Exam traps and keyword→answer → Quick-reference comparison table → Practice scenarios (6–8, original) → Sources.

## Quality controls
1. Haiku's training knowledge predates the 2026 rebrand, so the brief carries the verified facts and agents must verify new-product details with Tavily.
2. No invented quotas or prices: unsure items are marked `⚠ verify` or omitted.
3. Single-owner rule per topic, with cross-refs (`→ Ch0X`) to avoid duplication.
4. Orchestrator QA after merge:
   - structure check (all 11 H2s per chapter)
   - word counts
   - grep for stale "Vertex AI" without the new name
   - spot-check high-risk facts (DB choice, LB types, KMS, VPC-SC) against official docs
   - collect `⚠ verify` items and resolve them
5. Merge script generates the TOC from H1/H2 headings, adds a title page and exam-weight map, and puts the Ch17 cheat sheet last.

## Deliverable
`GCP PCA/GCP-PCA-2026-Handbook.md` (single file, TOC, 17 parts). Chapter sources stay in `GCP PCA/handbook/`.
