# Pharma Drug Reviews ETL Pipeline

In my [NLP project](https://github.com/Aritra-20/pharma-nlp-sentiment) I worked on the Drugs.com review data as one flat CSV. That's fine for analysis, but real data rarely shows up that way. Product catalogues and review systems usually live in separate places, and someone has to reconcile them before anyone can run a query.

So I rebuilt the same dataset as a small data warehouse. I pulled the raw data out, cleaned and modelled it into a star schema, loaded it into SQLite, and then answered business questions in SQL instead of pandas.

Notebook: [pharma-etl-pipeline-drug-reviews.ipynb](pharma-etl-pipeline-drug-reviews.ipynb)

## The pipeline

```
drugsComTrain_raw.csv (161,297 reviews)
        │
   EXTRACT  → raw_drug_master (8,490 drug–condition pairs)
            → raw_reviews     (161,297 review records)
        │
 TRANSFORM  → clean text, standardise drug names, fill missing conditions
            → add sentiment_score, review_length, usefulness_ratio
            → model as a star schema
        │
      LOAD  → SQLite: dim_drug + fact_reviews
        │
     QUERY  → SQL answers to business questions
```

**Extract.** I split the flat file into two sources the way it would arrive in practice. One is a drug master table with one row per drug–condition pair, like a product catalogue. The other is a review transaction table.

**Transform.** I decoded HTML entities and removed stray symbols from the review text, then standardised drug-name casing so one drug doesn't end up as two records. Missing conditions are labelled instead of dropped. I added three features:

| Feature | What it is |
|---|---|
| `sentiment_score` | VADER compound score, from -1 to +1 |
| `review_length` | Word count of the cleaned review |
| `usefulness_ratio` | `usefulCount` divided by review length, a rough "high-value review" signal |

**Load.** The modelled tables go into a SQLite database:

| Table | Grain | Rows |
|---|---|---|
| `dim_drug` | One row per drug–condition, with a surrogate `drug_id` | 8,490 |
| `fact_reviews` | One row per review, linked by `drug_id` | 160,398 |

About 900 reviews didn't match a drug record, and I dropped them as a data-quality check instead of letting them through with missing keys.

## What the queries showed

**Sentiment tracks the star rating in the right direction.** Average sentiment rises steadily from -0.47 at 1 star to +0.15 at 10 stars. Even 7-star reviews are slightly negative on average, because people describe their symptoms before saying the drug helped. That's the same effect I dug into in the NLP project.

**Some conditions sound far worse than their ratings suggest.** These have the lowest average sentiment (minimum 50 reviews):

| Condition | Reviews | Avg sentiment | Avg rating |
|---|---|---|---|
| Dental Abscess | 90 | -0.470 | 6.50 |
| Neuropathic Pain | 253 | -0.434 | 6.42 |
| Postherpetic Neuralgia | 71 | -0.422 | 7.30 |
| Diverticulitis | 142 | -0.388 | 6.68 |
| Osteoporosis | 372 | -0.374 | 4.83 |

Postherpetic Neuralgia gets a 7.3 rating while its reviews read as strongly negative. That gap is what a star rating alone would miss.

**Birth control dominates review volume.** Six of the ten most-reviewed drugs are contraceptives, led by Levonorgestrel with 3,631 reviews. Phentermine stands out in that top ten, with an 8.78 rating and clearly positive sentiment (+0.24). The contraceptives all sit slightly below zero.

## Tech stack

Python · Pandas · NumPy · NLTK (VADER) · SQLite (`sqlite3`) · Matplotlib · Seaborn

## Data

[UCI ML Drug Review dataset](https://www.kaggle.com/datasets/jessicali9530/kuc-hackathon-winter-2018) (Drugs.com), training split `drugsComTrain_raw.csv`.

## What I'd do next

- Move the same schema to a cloud warehouse such as BigQuery or Snowflake.
- Add a `dim_date` table to track month-over-month sentiment trends.
- Schedule the pipeline with Airflow so new reviews load incrementally.
