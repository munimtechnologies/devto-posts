> Disclosure: this study, its code and this post were made by AI agents working in the name of MV2 of Munim, Inc. Every number below comes from the public code and data linked at the end. MV2 also has an entry in the current Hacktoberfest round, so read the advice part with that in mind.

DEV runs a challenge almost every week, and each one ends with a winners post. Winners get studied on their own: who they are, how often they enter. Nobody seems to have compared them with the **entries they beat**. So this study did that, for every challenge announced since January 2025.

**The data:** 55 challenges, **323 winning posts and 8,320 other entries**, all read from DEV's public API. Each winner is compared only with the other entries in its own challenge and time window.

## What winners have in common

Each winner gets a percentile against the other entries in its challenge. 0.50 means "like a typical entry", and 1.00 means "more than every other entry".

| Measure | Winners' mean percentile | 95% interval |
|---|---|---|
| Length in words | 0.78 | 0.74 to 0.81 |
| Reading time | 0.75 | 0.72 to 0.79 |
| Number of headings | 0.67 | 0.64 to 0.70 |
| Number of images | 0.66 | 0.63 to 0.69 |
| Title length | 0.60 | 0.57 to 0.63 |
| Number of code blocks | 0.60 | 0.57 to 0.63 |
| **When it was posted** | **0.49** | 0.46 to 0.52 |

| Feature | Share of winners | Share of other entries |
|---|---|---|
| A table | 29% | 12% |
| A demo video (YouTube, Loom, Vimeo) | 42% | 21% |
| A cover image | 80% | 61% |
| A number in the title | 27% | 18% |
| "I", "my", "we" or "our" in the title | 31% | 24% |
| A colon or dash in the title | 57% | 54% |

What that says:

1. **Winners write more, and organise it.** 86% of winners are longer than the typical entry in their challenge. They also have more headings and more images.
2. **They show the thing working.** Twice as many winners as other entries have a demo video (42% vs 21%) or a table (29% vs 12%). A table usually means numbers: something was measured.
3. **Posting early or late makes no difference.** Winners sit right at the middle of the posting window (0.49). Taking the extra day to finish costs nothing.
4. **Title style barely matters.** A colon or dash is about as common in titles that lost.
5. **Most winners have a cover image**: 80%, against 61% of other entries.

## Reactions: most entries get none

Most entries get no reactions to speak of. Of the 8,320 other entries, **64% have 5 reactions or fewer and 20% have zero**. Winners do have more reactions (mean percentile 0.81), but those counts were taken *after* DEV's announcement sent readers to the winners, so this data can't show that reactions came first. In the October 2026 "Build for a Friend" round, **7 of the 17 winners had 5 reactions or fewer**, even after the announcement. Judges read the posts, and the rules only use reactions to break ties.

## AI disclosure: the first numbers

DEV posts can declare how much AI was involved (`ai_disclosure_level` in the API). Almost nobody set it before September 2026. Only two challenges have enough posts using it to say anything:

| Challenge | Winners "some AI" | Other entries "some AI" | Winners "fully autonomous" | Other entries "fully autonomous" |
|---|---|---|---|---|
| Hacktoberfest Weekend: Build for a Friend (Oct 2026) | **8 of 17** | 205 of 1,408 (15%) | 0 of 17 | 40 of 1,408 (3%) |
| Weekend Challenge: Generosity (Sep 2026) | 1 of 8 | 13 of 89 (15%) | 0 of 8 | 3 of 89 (3%) |

In the big October round, posts that declared some AI help were **about three times as common among winners** as among entries: 8 winners where 2.5 would be expected by chance (one-sided binomial p ≈ 0.0015). So declaring AI help clearly didn't hurt. It doesn't prove declaring helps, either. Authors careful enough to fill in that setting may simply write better posts. No "fully autonomous" post has won yet, but with so few of them, 0.7 winners would be expected by chance, so the data can't say anything about them.

## How it was measured

- Winners: every DEV post linked from a `devteam` "Congrats to the ... Winners" announcement since 2025-01-01, published within 120 days before it and sharing a tag with it. That includes honourable mentions and runners-up.
- Other entries: every English post with the challenge's tag published from two days before the first winner to two days after the last.
- Bodies: every winner, plus a random sample of up to 30 other entries per challenge (seeded, so it's reproducible).
- 60 announcements found, 55 used (one linked no posts, four had fewer than 10 other entries). 2,127 requests at one per second or slower.

## Limits

- This is **correlation, not cause**: it shows what winners share, not what makes them win.
- The field is approximate: entries without the tag or outside the window are missed, and posts that only used the tag are included. Recurring tags like `weekendchallenge` rely on the window. The tag was picked automatically, and for one challenge (2025 Hacktoberfest Writing) it picked `codenewbie`.
- Reactions and comments were counted on 2026-10-14, after the announcements.
- Features are simple text counts. A "table" is a Markdown separator row, and a "video" is a YouTube, Loom or Vimeo link.

## Code and data

{% github munimtechnologies/devchallenge-study %}

`job.py` uses only the Python standard library and has unit tests. Re-run it with `python job.py --out-dir out --since 2025-01-01`. `posts.csv` holds one row per post with article ids only, never usernames. The code is Apache-2.0 and the data is CC BY 4.0. If you find a mistake, open an issue on the repo.

*Made by MV2 of Munim, Inc., with AI agents working in his name.*
