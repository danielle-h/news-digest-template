# AI news digest template

A free daily email digest that reads news sites from across the political spectrum, asks Gemini
to summarise the day and compare how outlets of different leanings frame each story, and emails
you the result. It runs every morning on GitHub Actions, keeps a short memory of past stories so
it can follow them over time, and sends a weekly review on Fridays.

The example sources and wording follow Israeli news in Hebrew and English plus international
outlets. Change them to any topic: see [Adapting it to your topic](#adapting-it-to-your-topic).

Everything here runs on free tiers: Gemini's free API tier, GitHub Actions minutes, and Gmail.

## What each email contains

- **Daily digest:** top stories, with a "How it's framed" note where left and right outlets
  differ; follow-ups on older stories that resurfaced; stories only one side is covering;
  international news; and "And finally", one good, funny or whimsical story. Every claim links to
  the source article.
- **Weekly review (Fridays):** how the multi-day stories moved, what went quiet, how the coverage
  shifted, and what to watch next week. It's written from the daily records, so it doesn't repeat
  the dailies.

## Setup

### 1. Create your copy

Click **Use this template** at the top of this page and create a **private** repository. Private
matters: the workflow commits your story history to the repo every day.

### 2. Get the credentials

- **Gemini API key:** create one at [Google AI Studio](https://aistudio.google.com/apikey).
- **A Gmail account just for sending:** create a new account rather than using your main one.
  Turn on 2-step verification, then create an app password at
  <https://myaccount.google.com/apppasswords>. If the password ever leaks, it only exposes an
  empty mailbox.

### 3. Add them to your repository

In your repository, go to **Settings > Secrets and variables > Actions** and add:

| Name | Type | Value |
|---|---|---|
| `GEMINI_API_KEY` | Secret | Your Gemini key |
| `GMAIL_ADDRESS` | Secret | The sending Gmail address |
| `GMAIL_APP_PASSWORD` | Secret | The app password |
| `MAIL_TO` | Variable | Where to send it; several addresses separated by commas |

Optional variables:

| Name | Purpose |
|---|---|
| `DIGEST_LANGUAGE` | `English` (default) or `Hebrew`, which gets a right-to-left layout and Hebrew outlet names. Both have been tested for the daily digest; the weekly review has only been tested in English. Other languages are accepted but untested |
| `ELECTION_SECTION` | `on` adds an Elections section (the wording is written for Israel's election; edit `ELECTION_SECTION` in `digest.py` for yours) |
| `GEMINI_MODEL` | Main model; defaults to `gemini-flash-latest` |
| `GEMINI_FALLBACK_MODEL` | Used when the main model is busy; defaults to `gemini-flash-lite-latest`. Can list several models, comma-separated, tried in turn |

Never paste keys or passwords into an AI chat or commit them to the repository.

### 4. Run it

Go to **Actions > Daily news digest > Run workflow**:

- `feeds` checks every source from GitHub's servers and sends no email. Run this first.
- `preview` writes the daily digest to a file you can download from the run's page, and sends no
  email. Use it to try changes.
- `test` sends a daily digest by email without saving anything, so the next scheduled digest is
  unaffected.
- `daily` sends the digest.
- `weekly` sends the weekly review (it needs a few days of history first).

The optional **language** field sets the language for that run only, for example `Hebrew`.

After that it runs by itself: the daily digest at 03:17 UTC and the weekly review on Fridays at
04:23 UTC. Change the times in `.github/workflows/digest.yml`.

## Adapting it to your topic

1. **Sources:** edit `feeds.json`. Each entry has:
   - `name`, `url` (the RSS feed), `lang` (`en`, `he`, …).
   - `lean`: the outlet's political leaning or angle, shown to the model so it can compare framing.
   - `region`: which group of sources it belongs to. The example uses `israel`, `world`, and
     `light` (light news, used only for "And finally").
   - `site` (optional): the site's domain. When the site's own feed fails, or there is none, the
     script reads a Google News search of that domain, then a Bing News search.
   - `also_search` (optional, needs `site`): `true` for sites whose own feed holds only its latest
     10-20 items. The rest of the day is filled in from the Google News search.
   - `window_hours` (optional): read further back than usual, for feeds that publish rarely.
   - `display_names` (optional): the outlet's name in other languages, used for the source links
     when the digest is in that language, for example `{"hebrew": "הארץ"}`.

   To find a site's feed, try adding `/feed` or `/rss` to its address, or look for an RSS link on
   the page.
2. **Wording and sections:** edit `DAILY_PROMPT` and `WEEKLY_PROMPT` at the top of `digest.py`.
   The example is written for a reader who wants "mostly Israeli news"; change the first
   paragraph, the section names, and the region names to match your `feeds.json`. The region
   headings the model sees are set in `build_daily_prompt`.

If you use an AI coding assistant, it can make these changes for you. Describe your topic and
your sources and ask it to adapt `feeds.json` and the prompts.

## Running it locally

```
pip install -r requirements.txt
cp .env.example .env            # then fill it in
python digest.py --fetch-only   # check the feeds, no Gemini or email
python digest.py --dry-run      # write out/digest.html, no email, history untouched
python digest.py --no-save      # send the email without updating the history
```

## How it works

1. Read every feed, keeping only articles published since the last digest (at most 48 hours
   back), and number each headline.
2. Send all headlines to Gemini in one request, with each outlet's leaning and the last three
   weeks of story records.
3. Gemini writes the digest and cites item numbers. The code turns those numbers into links, so
   the model never writes URLs itself and can't invent one.
4. Email the digest, then save the day's story records to `history.json` and the send time to
   `state.json`. The workflow commits both back to the repository.

It's a plain pipeline with one AI call rather than an agent: the code decides every step, which
keeps it predictable and cheap.

## Things to know

- **Some sites block GitHub's servers.** That's why feeds can fall back to Google News and Bing
  News searches. The footer of each email lists any feed that failed.
- **Gemini's free tier is sometimes busy.** The script retries, alternating between two models.
  If every attempt fails, the run fails and GitHub emails you.
- **GitHub starts scheduled runs late,** sometimes by hours. That's normal for free scheduled
  workflows. The schedule uses odd minutes, which helps; if you need an exact time, trigger the
  workflow from an external scheduler.
- **First emails may land in spam** until you mark the sender as safe.
- **History:** records older than 45 days are deleted automatically.

## License

MIT. See [LICENSE](LICENSE).
