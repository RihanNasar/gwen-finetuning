# Movie Knowledge Fine-Tuning Challenge

## Phase 1: Dataset Creation

This documentation covers the first phase of the project - creating the training dataset.

## What is this?

This dataset contains question-answer pairs about movies, actors, directors, and their relationships. It's built from IMDb's public datasets and designed for fine-tuning language models on factual movie knowledge retrieval.

The questions range from simple ("How many movies did Tom Hanks act in 1994?") to complex multi-hop queries ("Which movies did Leonardo DiCaprio and Kate Winslet act together in?").

## How we built it

### Data sources

We used five datasets from IMDb's official website (datasets.imdbws.com):

- **title.basics.tsv.gz** - Basic movie information (titles, years, genres)
- **title.principals.tsv.gz** - Cast and crew for each movie
- **name.basics.tsv.gz** - Names of people in the industry
- **title.crew.tsv.gz** - Directors and writers organized by movie
- **title.ratings.tsv.gz** - Movie ratings (not used in current version)

### Filtering decisions

We made several filtering choices to keep the dataset clean and useful:

**Movies only**: We excluded TV shows, episodes, video games, and other non-movie content. Only entries marked as 'movie' in titleType were kept.

**Year range**: Movies from 1920-2024. This excludes silent era films (often missing data) and future releases (incomplete information).

**Person credibility**: We only included people with 3-500 film credits. This removes:
- One-off appearances (likely data errors or uncredited roles)
- Suspiciously prolific entries (usually data quality issues)
- The sweet spot captures working professionals without noise

**Required data**: Movies must have a title, year, and genre. People must have names. Missing data (marked as '\N' in IMDb files) gets filtered out.

### Knowledge base structure

After filtering, we merged everything into a single knowledge base where each row represents one person's involvement in one movie. For example, if Tom Hanks acted in Forrest Gump, that's one row. If Robert Zemeckis directed it, that's another row.

Columns in the knowledge base:
- tconst: IMDb movie ID
- primaryTitle: Movie name
- startYear: Release year
- genres: Comma-separated list
- nconst: IMDb person ID
- primaryName: Person's name
- category: Their role (actor, actress, director, writer, producer)

## Question types and complexity

We generated six types of questions with varying complexity levels:

### Single-hop queries (30% of dataset)

**Type: Temporal queries**

Pattern: "How many movies did [person] [verb] in [year]?"

Example:
```
Q: How many movies did Tom Hanks act in 1994?
A: Tom Hanks appeared in 2 movie(s) in 1994:
1. Forrest Gump
2. The Road to Wellville
```

These test basic lookup: find person → filter by year → count movies.

### Two-hop queries (40% of dataset)

**Type: Year range queries**

Pattern: "Which films did [person] [verb] between [year1]-[year2]?"

Example:
```
Q: Which films did Steven Spielberg direct between 1993-1997?
A: Steven Spielberg worked on 3 film(s) between 1993-1997:
1. Jurassic Park (1993)
2. Schindler's List (1993)
3. The Lost World: Jurassic Park (1997)
```

These require: find person → filter by role → filter by year range → list movies.

### Three-hop queries (20% of dataset)

**Type: Actor-director collaborations (10%)**

Pattern: "Which movies did [actor] act in that were directed by [director]?"

Example:
```
Q: Which movies did Leonardo DiCaprio act in that were directed by Martin Scorsese?
A: Leonardo DiCaprio acted in 5 movie(s) directed by Martin Scorsese:
1. Gangs of New York (2002)
2. The Aviator (2004)
3. The Departed (2006)
4. Shutter Island (2010)
5. The Wolf of Wall Street (2013)
```

**Type: Co-star queries (10%)**

Pattern: "Which movies did [actor1] and [actor2] act together in?"

Example:
```
Q: Which movies did Brad Pitt and George Clooney act together in?
A: Brad Pitt and George Clooney appeared together in 3 movie(s):
1. Ocean's Eleven (2001)
2. Ocean's Twelve (2004)
3. Ocean's Thirteen (2007)
```

These need: find actor1's movies → find actor2's movies → find intersection.

### Aggregation queries (10% of dataset)

**Type: Director comparisons (5%)**

Pattern: "Who directed more movies in the [decade]s: [director1] or [director2]?"

Example:
```
Q: Who directed more movies in the 1990s: Steven Spielberg or Martin Scorsese?
A: Steven Spielberg directed more movies in the 1990s with 9 films compared to Martin Scorsese's 7 films.
```

**Type: Genre counts (5%)**

Pattern: "How many [genre] movies has [person] [verb] in?"

Example:
```
Q: How many Action movies has Tom Cruise appeared in?
A: Tom Cruise has appeared in 23 Action movie(s).
```

These involve counting and sometimes comparing counts across different filters.

## Statistics

After processing, our knowledge base contains:

- ~500,000-800,000 person-movie records (varies by download date)
- Actors/actresses: Usually 60-70% of records
- Directors: Usually 15-20% of records
- Writers: Usually 10-15% of records
- Producers: Usually 5-10% of records

The final dataset has 2,000 Q&A pairs split as:
- train.jsonl: 1,600 examples (80%)
- validation.jsonl: 200 examples (10%)
- test.jsonl: 200 examples (10%)

Distribution by complexity:
- Single-hop: ~600 examples
- Two-hop: ~800 examples
- Three-hop: ~400 examples
- Aggregation: ~200 examples

## Edge cases and limitations

### Things we handle

**Multiple movies in one year**: If someone worked on 15 movies in a year, we list up to 10 and add "... and 5 more" at the end.

**Missing genres**: Some movies have no genre data. Genre queries skip these.

**Name variations**: We use primaryName from IMDb, which is their canonical form. Stage names and variations are already normalized by IMDb.

**Duplicate roles**: If someone is listed as both actor and producer on the same film, they appear in our dataset twice with different category values. This is intentional and accurate.

### Things that might be weird

**Actress vs actor**: IMDb separates these as different categories. We preserve this distinction because it's in the source data, even though it's somewhat outdated.

**Self-collaborations**: You might see "Which movies did Clint Eastwood act in that were directed by Clint Eastwood?" This is valid - he both acted and directed many films.

**Zero collaborations**: During generation, we try to pick people who actually worked together, but the attempt limit (20x for actor-director, 10x for co-stars) means we might occasionally skip valid pairs if we're unlucky.

**Year boundaries**: "Between 1990-1995" is inclusive on both ends, so it includes movies from both 1990 and 1995.

**No TV shows**: Even though some "movies" in IMDb are actually TV movies or direct-to-video releases, if IMDb categorizes them as titleType='movie', we include them.

### Known limitations

**Recent movies incomplete**: For very recent years (2023-2024), cast/crew information might be incomplete as IMDb updates their data continuously.

**Foreign films underrepresented**: IMDb is more comprehensive for English-language films. Other languages are present but less complete.

**No rating/quality filtering**: We don't filter by movie ratings or popularity, so you'll get obscure films mixed with blockbusters.

**Character names not included**: We don't track which character an actor played, just that they appeared in the film.

**Genre ambiguity**: Genres are comma-separated strings like "Action,Thriller,Drama". We do substring matching, which means searching for "Action" will match any movie with Action in its genre list.

## Data format

Each file (train.jsonl, validation.jsonl, test.jsonl) contains one JSON object per line in ChatML format:

```json
{
  "messages": [
    {
      "role": "user",
      "content": "How many movies did Tom Hanks act in 1994?"
    },
    {
      "role": "assistant",
      "content": "Tom Hanks appeared in 2 movie(s) in 1994:\n1. Forrest Gump\n2. The Road to Wellville"
    }
  ]
}
```

The metadata (query type and complexity level) is not included in the output files - it's only used during generation for maintaining the distribution.

## Reproducibility notes

If you regenerate this dataset, the exact questions will differ because:

1. Random sampling of people from the knowledge base
2. Random year/decade selection for temporal queries
3. Random pairing of directors for comparison queries
4. The shuffle before train/val/test split

However, the distribution (30/40/20/10) and general characteristics should remain consistent.

IMDb updates their datasets daily. If you download the source files on different dates, the underlying data will be slightly different (new movies added, data corrections, etc.).

## Usage tips

**For fine-tuning**: Use train.jsonl for training, validation.jsonl for hyperparameter tuning, and test.jsonl for final evaluation. Don't look at test.jsonl during development.

**For evaluation**: The different complexity levels let you measure model performance on different reasoning depths. A model might do well on single-hop queries but struggle with three-hop reasoning.

**For augmentation**: You can easily generate more examples by running the code again with a different random seed or higher total_samples count.

## License and attribution

This dataset is derived from IMDb datasets, which are available for personal and non-commercial use. See IMDb's licensing terms at https://www.imdb.com/interfaces/.

When using this dataset, please acknowledge IMDb as the source of the underlying information.
