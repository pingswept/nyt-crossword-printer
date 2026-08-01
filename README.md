Tampermonkey script for downloading NYT crosswords as a PDF, requires NYT subscription

I built this because printing out crosswords is tedious. I am a programmer, but most of this was written with AI, so who the hell knows if it works. Good luck.

## Setup

1. Install the **Tampermonkey** browser extension
2. Click the Tampermonkey icon → **Create a new script**, delete the
   placeholder content, and paste in the contents of
   `nyt-crossword-fetcher.user.js`. Save (Ctrl+S).

## Usage

1. Go to `https://www.nytimes.com/crosswords/archive/daily`while logged in.
2. A "Crossword Batch Fetcher" panel should appear in the top-right.
4. Click **Fetch & Download** — the file downloads through your browser like any other download.
