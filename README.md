# Hidden Reminders – Content Setup

This package keeps your quotes and letters in one Excel workbook, publishes them as three small files on a free host, and lets the game update itself without an app update.

## What's inside

| Path | What it is |
|---|---|
| `Tools/Quotes_for_games_all.xlsx` | The master workbook. Edit **All Quotes**, **Letters** and **Categories** only. |
| `Tools/export_content.py` | Turns the workbook into the files the game downloads. |
| `Tools/export/` | The exported `manifest.json`, `quotes.json` and `letters.json` (version 1). You never need to open or edit these. |
| `Unity/Resources/Content/` | Built-in fallback copies of `quotes.json` and `letters.json`. |
| `Unity/Scripts/Content/` | `ContentSync.cs`, `ContentModels.cs` and `LetterReplyRules.cs`. |
| `Unity/Scripts/Replace/` | New versions of your `QuoteProgress.cs` and `QuoteFiles.cs`. Replace the old ones. |

## 1. Put the files in Unity

1. Copy `Unity/Scripts/Content/` into `Assets/Scripts/`.
2. Copy `Unity/Resources/Content/` into `Assets/Resources/` so you have `Assets/Resources/Content/quotes.json` and `letters.json`.
3. In your first scene (Opening), create an empty GameObject called **ContentSync** and add the `ContentSync` component.
4. Set **Base Url** once you've done step 2 below. Until then, the game simply runs on the built-in files.

## 1b. Connect it to JourneyMenu

1. Replace your `QuoteProgress.cs` and `QuoteFiles.cs` with the ones in `Unity/Scripts/Replace/`. Their methods are unchanged, so JourneyMenu, ReadPanel and gameplay keep working.
2. On the **Categories** sheet, set **Unity Folder** for each category to the exact **Folder** name you typed for that category in JourneyMenu. Then run the export script.
3. A category whose folder matches now reads its quotes from `quotes.json`. A folder that doesn't match keeps using the old StreamingAssets files, so you can switch over one category at a time.
4. Progress is now saved against quote IDs. The first time a player opens the menu after the update, their old progress is moved over automatically by matching quote wording. Keep the old StreamingAssets folders in the build for at least one release so this can happen.

## 2. Host the files (GitHub Pages – free)

1. Create a free GitHub account and a new **public** repository, e.g. `hidden-reminders-content`.
2. Upload `manifest.json`, `quotes.json` and `letters.json` from `Tools/export/` (Add file → Upload files → Commit).
3. Go to **Settings → Pages**, set Source to *Deploy from a branch*, branch `main`, folder `/ (root)`, and save.
4. After a minute your files are live at `https://YOUR-NAME.github.io/hidden-reminders-content/`. Open `.../manifest.json` in a browser to check.
5. Paste that folder URL (ending in `/`) into **Base Url** on the ContentSync component.

Firebase Hosting or Cloudflare Pages work the same way if you prefer them.

## 3. Updating content later

1. Edit the workbook: add, amend, or set **Active = No** to remove. Set **Pick** to *In Game* or *Reserve* (any number of quotes can be In Game). Never delete rows or change IDs; new rows can leave the ID blank.
2. Close Excel, then run (first time only: `pip install openpyxl`):
   ```
   python export_content.py --unity "C:/YourGame/Assets/Resources/Content"
   ```
3. Upload the three files from `Tools/export/` to GitHub, replacing the old ones.

Players get the new content the next time they open the game online. The script only raises a version number when that file actually changed, so fixing one letter won't make anyone re-download the quotes.

The `--unity` part refreshes the built-in fallback too. It's worth doing before each app release, so new players start with the latest content even offline.

The script also rebuilds the category tabs (Memento, Qalb…) so they always match All Quotes.

## Adding a category or theme

- **Category:** add a row on the **Categories** sheet (ID in lowercase, Title, Subtitle, Order). It appears in the Category dropdown on All Quotes straight away, and the script creates its tab and exports it.
- **Theme:** add it to the themes list in column F of the **Categories** sheet. It appears in the theme dropdowns, and the script accepts it.
- **In Unity:** if your menu builds its buttons from `ContentSync.Instance.Categories`, new categories appear on their own. Give unknown categories a default look (cover, colour) so a new one never shows up blank.
- The Summary table lists the original five categories. Add a row there for a new category if you want it counted.

## 4. Using it in your code

```csharp
var content = ContentSync.Instance;

// Categories in order, with titles and subtitles
foreach (var cat in content.Categories)
    Debug.Log($"{cat.title}: {cat.subtitle}");

// The In Game quotes of one category
List<ContentQuote> memento = content.GetQuotes("memento");

// Look up a quote saved in the player's progress (null if it was removed)
ContentQuote q = content.GetQuote("Q0042");

// Refresh your UI when new content arrives mid-session
content.OnContentUpdated += RefreshBook;

// Letters
var ready   = LetterReplyRules.LettersReadyToArrive(content.AllLetters, completedCount, receivedIds);
var options = LetterReplyRules.AvailableReplies(content.AllQuotes, completedIds, usedReplyIds);
LetterReplyQuality result = LetterReplyRules.Evaluate(letter, chosenQuote); // Okay / Good / Best
```

## Rules that keep players' saves safe

- **Save IDs, not text or list positions.** Store `"Q0042"` for completed quotes and `"L007"` for letters, so edits and reordering never break progress.
- **Expect missing IDs.** If you deactivate a quote, `GetQuote` returns `null` for it, so skip it when loading a save.
- **Never reuse an ID.** The export script always continues from the highest existing number.
