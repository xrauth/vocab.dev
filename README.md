# Palavras

A small single-page vocabulary game for German, English, Spanish and Portuguese. It shows you a word with its translation, brings it back a few cards later with the word hidden, and asks you to spell it one letter at a time.

**This is not a language-learning platform.** There are no lessons, no grammar explanations, no sentences, no progression path. It shows words and assumes you already know what to do with them: you have a textbook, a teacher, a book or a habit, and you want a place to keep seeing and spelling vocabulary. Everything below should be read with that in mind.

It is a personal project, built iteratively, and many of its numbers are educated guesses. I try to be explicit about which ones.

> Developed with the help of an AI assistant (Claude); the mechanics were decided and revised through play and conversation.

## How it works

There are two kinds of cards, and a fixed first card.

- **Start card.** Always the same word (*start / iniciar / iniciar / starten*). It is only there to be skipped, so the screen is never empty while data loads.
- **Exposure card.** A new word appears with its part of speech (and gender, when it exists) and its translation. Tap to continue. Tapping schedules the word to return.
- **Writing card.** When a word comes due, the main word is replaced by blank spaces in a monospace font and only the translation is visible. You fill it in one letter at a time. The card has a fixed size, so nothing reflows while you type.

Cards turn over horizontally, as if the next word were on the back.

### Typing

Each letter is chosen from a row of **three** keys: the correct one plus two distractors. Distractors never include the *next* letter of the word (so for "dog", typing `d` will not offer `o`), and when the language has accented variants they are used as distractors (`u` / `ü`, `c` / `ç`).

If you press a wrong key, it fades out and the correct key is highlighted. You continue letter by letter. The penalty is applied **once** per word, no matter how many times you slip. Only a perfect run counts as a success.

After the last letter the game advances by itself after 1.5 seconds. A soft green or red glow on the card signals success or error.

### Scheduling

Words come back according to a simple interval, measured **in cards, not in time**:

- A newly seen word returns after 3 cards.
- Perfect writing: the interval is multiplied by 2.5.
- Any error: the interval is divided by 3 (minimum 2).
- When nothing is due, a new word is introduced.

### Score

The score is loosely inspired by the Elo rating in chess. Each word has an estimated difficulty derived from its frequency and length; your score and the word's difficulty give an expected chance of success.

- A perfect word raises the score by `K · (1 − expected)`: rare, hard words are worth more.
- The first error lowers it by `1.5 · K · expected`: missing an obvious word costs more.
- `K` shrinks as you play, so the score grows more slowly over time.

The score starts at 1000. It is a rough feedback signal, not a measurement of anything.

### Word counter

The counter at the top counts words you have *demonstrated* you know. A word is counted the first time you write it perfectly, and uncounted if you later fail it. Repeated successes do not add up. The intent is a number that stays roughly honest.

### Frequency level

Each word carries a frequency value from 10 (rare) to 100 (very common). The game keeps a hidden "level" that starts at 100. Perfect writing moves it toward rarer words (faster on streaks), and errors move it back. New words are picked from the batch closest to that level, so someone who already knows the common words is not kept there for long.

## Design decisions, and what I'm unsure about

**Writing from the translation, not from the sound.** The writing card shows the translation and asks for the word. That is production, not recognition, and it is the harder direction. It also means you practise only one direction: translation → word. There is no reverse card.

**Letter by letter, with options.** Typing on a phone keyboard invited odd behaviour (autocorrect, swipe, layout shifts), so letters are chosen from on-screen keys. This is *not* real typing. With three options, a blind guess has a one-in-three chance per letter, and recognising the right letter is easier than recalling it. A "perfect" run is therefore weaker evidence than it looks. Earlier versions showed only the letters of the word and turned the game into unscrambling; later ones showed five options. Three, with the "next letter" rule, is where I stopped. It is a compromise, not a proven choice.

**Punish imprecision once.** Errors cost points a single time but a word is only a success if flawless. I wanted spelling mistakes to matter without making one slip snowball. The 1.5× error multiplier and other constants were picked by feel.

**Scheduling in cards, not hours.** It keeps the system simple and avoids clock and timezone issues, but it means there is **no forgetting curve**: if you stop for a month, nothing has decayed. The 2.5× growth and ÷3 shrink are not taken from SM-2, FSRS or any study; they are plausible numbers I have not tuned or validated. A proper spaced-repetition algorithm would almost certainly do better.

**Difficulty from frequency and length only.** The word's difficulty ignores things that clearly matter: cognates, spelling regularity (Spanish and Portuguese are far more phonetic than English), your native language, your goals. The score is therefore not comparable across language pairs, and the frequency level is a crude one-dimensional guess at "what you probably know".

**Single words only.** Each row is one word with one sense and one equivalent per language, with no articles, inflections, phrases or example sentences. Polysemy is handled by adding separate rows. A real learner would want context; this deliberately does not provide it.

**Things I tried and removed.**
- *Browser text-to-speech*, on cards and after typing. Voice quality varies a lot between devices, it gave away the spelling, and it cluttered the interface.
- *Typing the translation instead of the word.* It broke the rhythm.
- *Five letter options and a scrambled word bank.* It turned into an unscrambling puzzle instead of a translation-to-word exercise.
- *Auto-skipping on error.* Users could not look at the correct word, so errors reveal the correct key and let you continue instead.

## Architecture

One static HTML file, no build step, plus a [Supabase](https://supabase.com) project.

- **Progress.** Guests are stored in `localStorage`. Signing in with Google (optional) syncs progress to the server: per-word schedule and "known" flag, plus score, level and turn counter per language pair. Changes are queued locally and sent when the connection allows, and guest progress is merged on first login. Conflicts are resolved crudely (the server wins, except for words it has not seen).
- **Words.** The word bank lives in a table with no direct read access. The client only gets small batches (5 new words at a time) through two server functions, and can fetch specific words by id for reviews. A short list of 20 words is embedded in the page as an instant fallback when the server is unreachable.
- **Daily quota.** The server hands out at most 60 *new* words per day, per user if signed in, per IP address otherwise.
- **Interface language.** Labels (words counter, score, confirmations) follow the selected main language.

### Limitations of the architecture

- **The word bank is not secret.** The anonymous key is public by design and anything shown on screen can be copied. Batching and the daily quota slow bulk downloading; they do not prevent it. Quotas per IP also penalise people sharing a network and are easy to bypass.
- **Not stress-tested.** It has not been tried with a large bank or many users. Fetching new words sorts the whole table by distance to your level, which should be fine for tens of thousands of rows and not much beyond.
- **Local progress is fragile.** Guest data lives in one browser; clearing it loses everything.
- **Mobile audio, accessibility and screen readers, right-to-left scripts, and offline use beyond the fallback list** have not been considered or tested.

## Setup

1. Create a Supabase project and run `supabase.sql` in the SQL Editor. It creates the tables, security rules, server functions, quota table, and a small example word list.
2. In *Authentication → Providers*, enable Google (this needs an OAuth client from Google Cloud) and add your site URL to the redirect URLs.
3. Open `index.html` and replace `SUPA_URL` and `SUPA_KEY` with your project URL and public (anon) key. Never put the `service_role` key in the page.
4. Host the HTML file anywhere static (Netlify, Cloudflare Pages, GitHub Pages).

## Word data

Each row in `words` is one word across the four languages: an `id` (the German word, unique), `f` (frequency, 10–100), and a JSON object `w` with `{ w: word, p: part of speech, g: gender }` for each language. Languages a word does not exist in are omitted, and such a word is simply never offered for pairs that include them.

The simplest way to build the data is a flat CSV:

```csv
id,f,de_w,de_p,de_g,en_w,en_p,en_g,es_w,es_p,es_g,pt_w,pt_p,pt_g
Haus,98,Haus,substantiv,n,house,noun,,casa,sustantivo,f,casa,substantivo,f
```

Words may contain only letters (accents, `ß` and `ñ` are fine); the on-screen keys cannot produce spaces, hyphens or apostrophes. Import the CSV into a staging table and convert it with SQL:

```sql
create table words_stage(id text,f int,
 de_w text,de_p text,de_g text, en_w text,en_p text,en_g text,
 es_w text,es_p text,es_g text, pt_w text,pt_p text,pt_g text);
alter table words_stage enable row level security;

-- import the CSV (Table Editor → Import, or psql \copy), then:
insert into words(id,f,w)
select id,f,jsonb_strip_nulls(jsonb_build_object(
  'de',case when nullif(de_w,'') is not null then jsonb_build_object('w',de_w,'p',initcap(de_p),'g',coalesce(de_g,'')) end,
  'en',case when nullif(en_w,'') is not null then jsonb_build_object('w',en_w,'p',initcap(en_p),'g',coalesce(en_g,'')) end,
  'es',case when nullif(es_w,'') is not null then jsonb_build_object('w',es_w,'p',initcap(es_p),'g',coalesce(es_g,'')) end,
  'pt',case when nullif(pt_w,'') is not null then jsonb_build_object('w',pt_w,'p',initcap(pt_p),'g',coalesce(pt_g,'')) end))
from words_stage
on conflict(id) do update set f=excluded.f, w=excluded.w;

drop table words_stage;
```

For frequency, any ranking works as long as you map it onto 10–100, for example `f = round(100 - 90 * ln(rank) / ln(N))`. The quality of the progression depends almost entirely on how good that ranking is.

## Status

A personal tool that works for my own use. Expect rough edges. The constants above are the first thing I would revisit if the game feels too easy, too hard, or too forgiving.
