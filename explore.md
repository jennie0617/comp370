# Exploring the My Little Pony Dialog Dataset

Dataset: `clean_dialog.csv` from https://www.kaggle.com/liury123/my-little-pony-transcript, stored in this repo at `data/clean_dialog.csv`. All commands below were run from the root of the repo on my EC2 instance.

## How big is the dataset?

```
ls -lh data/clean_dialog.csv
wc -l data/clean_dialog.csv
csvtool height data/clean_dialog.csv
csvtool width data/clean_dialog.csv
```

- The file is **4.7 MB**.
- `wc -l` reports **36,860** lines and `csvtool height` reports **36,860** rows. Since the two numbers match, no dialog field contains an embedded newline, so one line in the file is one row of data.
- The first row is a header, so the dataset contains **36,859 lines of dialog**.
- Each row has **4 columns**.

## What's the structure of the data?

```
head -n 5 data/clean_dialog.csv
```

The header row is `"title","writer","pony","dialog"`, and every field is wrapped in double quotes. For example:

```
"Friendship is Magic, part 1","Lauren Faust","Twilight Sparkle","...and harmony has been maintained in Equestria for generations since. Hmm... Elements of Harmony. I know I've heard of those before... but where?"
```

| Field | What it contains | Example value | Distinct values |
|---|---|---|---|
| `title` | Name of the episode the line comes from | `Friendship is Magic, part 1` | 197 |
| `writer` | Credited writer of the episode | `Lauren Faust` | 66 |
| `pony` | The character (or characters) speaking the line | `Twilight Sparkle`, `Narrator` | 842 |
| `dialog` | The text of the line itself | `...sun and moon...` | — |

Distinct values per column were counted with:

```
csvtool col 1 data/clean_dialog.csv | tail -n +2 | sort -u | wc -l
csvtool col 2 data/clean_dialog.csv | tail -n +2 | sort -u | wc -l
csvtool col 3 data/clean_dialog.csv | tail -n +2 | sort -u | wc -l
```

(`tail -n +2` drops the header row before counting.)

The most frequent speakers were found with:

```
csvtool col 3 data/clean_dialog.csv | tail -n +2 | sort | uniq -c | sort -rn | head -n 20
```

The top ten are Twilight Sparkle (4,745), Rainbow Dash (3,072), Pinkie Pie (2,833), Applejack (2,748), Rarity (2,660), Spike (2,268), Fluttershy (2,109), Apple Bloom (1,362), Starlight Glimmer (1,161) and Others (979).

## How many episodes does it cover?

```
csvtool col 1 data/clean_dialog.csv | tail -n +2 | sort -u | wc -l
```

The dataset covers **197 episodes**, counting each distinct title as one episode. Multi-part episodes are listed as separate titles (for example `Friendship is Magic, part 1` and `Friendship is Magic, part 2`), so they count as separate episodes here.

## Unexpected aspects of the dataset

I looked at every speaker label containing "Twilight":

```
csvtool col 3 data/clean_dialog.csv | grep "Twilight" | sort | uniq -c | sort -rn | head -n 20
```

This turned up several things that would cause problems for later analysis:

1. **Some lines have more than one speaker.** The `pony` field can name several characters at once, such as `Narrator and Twilight Sparkle`, `Twilight Sparkle and Spike` (7 lines) or `Pinkie Pie and Twilight Sparkle` (3 lines). It's not obvious whose line count these should go toward.
2. **The same character appears under different names.** Besides `Twilight Sparkle`, there are labels like `Twilight` (as in `Twilight and Fluttershy`), `Young Twilight Sparkle` (12), `Future Twilight Sparkle` (8), `Past Twilight Sparkle` (6), `Mean Twilight Sparkle` (21) and `Illusion Twilight Sparkle` (5). Counting a character's lines depends on deciding which of these "count".
3. **A label can contain a name while meaning that character is *not* speaking.** `All sans Twilight Sparkle` (4) and `Main cast sans Twilight Sparkle` (2) are lines spoken by everyone *except* Twilight. A simple search for her name would wrongly credit these to her.
4. **Different characters share part of a name.** `Twilight Velvet` (16) is a different character (Twilight Sparkle's mother), so searching for just "Twilight" would mix two characters together.
5. **There's a catch-all `Others` speaker label** with 979 lines, so some dialog can't be attributed to any specific character.

Together these explain why there are 842 distinct speaker labels: the `pony` field isn't a clean list of character names.

## Speaker frequency of the main ponies

To count how often each main pony speaks, I extracted the `pony` column and used `grep` with `-c` (count matching lines) and `-x` (the whole line must match exactly):

```
csvtool col 3 data/clean_dialog.csv | grep -c -x "Twilight Sparkle"
csvtool col 3 data/clean_dialog.csv | grep -c -x "Rarity"
csvtool col 3 data/clean_dialog.csv | grep -c -x "Pinkie Pie"
csvtool col 3 data/clean_dialog.csv | grep -c -x "Rainbow Dash"
csvtool col 3 data/clean_dialog.csv | grep -c -x "Fluttershy"
```

Using `-x` means only lines spoken by that pony alone are counted. Lines shared with another character (e.g. `Twilight Sparkle and Spike`), variant labels (e.g. `Young Twilight Sparkle`), `sans` labels and other characters such as `Twilight Velvet` are all excluded. Shared lines are only a handful per pony, so this choice has very little effect on the totals.

The total number of lines across all characters (the denominator) was:

```
csvtool col 3 data/clean_dialog.csv | tail -n +2 | wc -l
```

which gives **36,859**. Each percentage was then computed with `awk`, for example:

```
awk 'BEGIN {printf "%.2f\n", 4745*100/36859}'
```

| Pony | Lines | % of all lines |
|---|---|---|
| Twilight Sparkle | 4,745 | 12.87 |
| Rainbow Dash | 3,072 | 8.33 |
| Pinkie Pie | 2,833 | 7.69 |
| Rarity | 2,660 | 7.22 |
| Fluttershy | 2,109 | 5.72 |

These results are also saved in `Line_percentages.csv`.

### Why not just grep the whole file?

For comparison, a naive search over the raw file:

```
grep -c "Twilight Sparkle" data/clean_dialog.csv
```

returns **5,120**, which is 375 more than the exact count. This is because it matches the name anywhere in the row, including lines where *other* characters say "Twilight Sparkle" in their dialog and the variant or `sans` speaker labels described above. Restricting the search to the `pony` column with an exact match avoids these false positives.
