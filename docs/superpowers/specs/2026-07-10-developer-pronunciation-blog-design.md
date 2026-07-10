# Developer Pronunciation Blog Design

## Goal

Create a Chinese-language blog post for Java, Python, and Vue developers who often mispronounce English technical terms. The post should be readable as a learning article and useful as a searchable pronunciation reference.

The core experience is an interactive pronunciation table where users can click either the English word or its IPA phonetic transcription to hear the word pronounced. Each row must include a Chinese pronunciation approximation so Chinese-speaking developers can learn quickly before using the audio and IPA as calibration.

## Audience

- Chinese-speaking Java, Python, and Vue developers.
- Developers preparing for meetings, interviews, code reviews, technical sharing, or English documentation discussions.
- Readers who want practical pronunciation help rather than general English study.

## Non-Goals

- Do not build a full English learning platform.
- Do not try to include every possible programming word in the first version.
- Do not ship custom audio files or external pronunciation service dependencies.
- Do not make the page a standalone marketing or landing page.

## Recommended Approach

Use a blog article plus an interactive MDX table component.

The article explains pronunciation principles and common traps, then embeds a reusable pronunciation table. The table is driven by structured data so new words can be added later without editing large chunks of markup.

This balances readability, maintainability, and usefulness:

- It still feels like a blog post.
- The table can support click-to-speak, search, and category filtering.
- The word list remains easy to expand.

## Content Structure

The post title should be close to:

`Java / Python / Vue 开发者常见英文发音词表`

Suggested sections:

1. Why developer pronunciation matters
   - State that the goal is clear technical communication, not perfect accent.
   - Mention common situations: standups, interviews, technical sharing, and reading docs aloud.

2. How to use the table
   - Click the English word, IPA, or speaker button to hear pronunciation.
   - Use the Chinese approximation as a memory aid.
   - Use IPA and audio as the final source of truth.

3. Common pronunciation traps
   - Letter names versus word pronunciation, such as SQL and Vue.
   - Long and short vowels, such as `ship` versus `sheep`.
   - Special clusters, such as `sch`, `que`, `th`, and `cache`.
   - Technology-specific conventions, such as `nginx` as "engine-x".

4. Interactive pronunciation table
   - Group words by Java, Python, Vue/frontend, database/cache, DevOps, and general engineering.
   - Include search and category filtering so a long table remains usable.

5. Practice sentences
   - Use realistic software engineering sentences.
   - Examples: "The async task is queued." and "The schema migration failed."

## Table Data Model

Each pronunciation row should include:

- `term`: English word or phrase, such as `cache`.
- `ipa`: IPA transcription, such as `/kæʃ/`.
- `zhPronunciation`: Chinese-friendly approximation, such as `凯什 / 开什`.
- `meaningZh`: Chinese meaning, such as `缓存`.
- `commonMistake`: Common Chinese-speaker mispronunciation, such as `卡切、卡其`.
- `note`: Short explanation or usage note.
- `example`: Optional realistic sentence.
- `category`: One of `Java`, `Python`, `Vue`, `Frontend`, `Database`, `DevOps`, or `General`.
- `speakText`: Optional override for text-to-speech when the visible term needs special handling.

The Chinese approximation is required for every row. It should be practical and easy to read, while the article clearly says it is only a learning aid. Do not force every sound into Chinese characters when that would mislead the reader. Mixed notation is allowed and preferred for sounds that Chinese characters represent poorly, such as `th`, `ki`, `kee`, and unstressed `uh`.

Example: `schema` should use `/ˈskiː.mə/` with a Chinese-friendly approximation like `斯 kee-ma`, not `斯基-ma` or `斯给-ma`. The note should explain that `sch` is pronounced `/sk/` here and `ee` is the long `/iː/` sound.

## Interaction Design

The pronunciation table should support:

- Click the English word to pronounce the word.
- Click the IPA phonetic transcription to pronounce the same word.
- Click a speaker button to pronounce the same word.
- Search by English term, Chinese meaning, Chinese pronunciation approximation, and common mistake.
- Filter by category.
- Keyboard access for all clickable pronunciation controls.
- Accessible labels that identify what will be pronounced.
- Graceful fallback when browser speech synthesis is unavailable.

The audio should use the browser's built-in `SpeechSynthesis` API. This avoids audio files and external services. The browser voice may vary across systems, which is acceptable for the first version.

The IPA click target does not read IPA symbols literally. It triggers pronunciation of the corresponding English term or `speakText` override.

## Component Design

Create a small Astro-compatible MDX component for the interactive table.

Suggested unit boundary:

- `PronunciationTable`: renders search, filters, table, and client-side speech behavior.
- `pronunciationTerms`: structured data for all table rows.

The component should keep presentation and data separate. The blog post imports the component and data, then uses them in the article.

## Initial Word List Scope

The first version should include about 80-120 high-value words.

Priority categories:

- Java and JVM: `Java`, `JVM`, `Spring`, `Maven`, `Gradle`, `daemon`, `enum`, `generic`, `hibernate`, `deprecated`, `synchronized`.
- Python and backend: `Python`, `Django`, `FastAPI`, `async`, `await`, `tuple`, `schema`, `decorator`, `generator`, `virtualenv`.
- Vue and frontend: `Vue`, `Vite`, `Nuxt`, `Pinia`, `props`, `emit`, `component`, `template`, `reactive`, `router`.
- General engineering: `cache`, `queue`, `suite`, `issue`, `repo`, `deploy`, `merge`, `branch`, `commit`, `release`.
- Database and DevOps: `SQL`, `Redis`, `Postgres`, `MySQL`, `nginx`, `Docker`, `Kubernetes`, `daemon`, `cron`, `proxy`.

Avoid low-frequency words in the first version unless they are especially easy to misread.

## Styling Requirements

The feature should fit the current AstroPaper styling:

- Use the existing article width and typography.
- Keep the table readable on mobile with horizontal scrolling if needed.
- Make clickable words and IPA visually obvious but not distracting.
- Keep controls compact; this is a reference table inside a blog, not a dashboard.

## Error Handling

- If `speechSynthesis` is unavailable, disable speech controls or show a short inline fallback message.
- If a voice is unavailable, use the browser default English-capable voice.
- If the browser blocks playback until user interaction, normal click-based playback should still work.

## Testing And Verification

Before completion:

- Run the project type/build check used by the repo.
- Verify the MDX article renders.
- Verify the table works on desktop and mobile widths.
- Verify clicking term, IPA, and speaker button all invoke speech behavior.
- Verify the page still works when speech synthesis is unavailable.

## Open Decisions

None. The approved direction is a readable Chinese blog post with an interactive pronunciation table. Chinese pronunciation approximations are required, and both English terms and IPA cells should be clickable.
