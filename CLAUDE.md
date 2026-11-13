# TME 6015 Assignment 2: food image classifier

A 6-class food image classifier built from recipe blog photos. The album packaging final project lives in its own
repo (`~/Developer/tme6015-album-project`). Nothing here reads or writes that repo.

## Environment
- macOS on Apple Silicon, PyCharm, `uv`-managed `.venv` on Python 3.12. `pyproject.toml` and `uv.lock` at the root.
- **Keras / TensorFlow.** CPU is fine at this data size. `tensorflow-metal` optional. Colab is an option.
- Load splits with `image_dataset_from_directory`, passing `class_names` explicitly so label indices match across splits.
- Every script sets `ROOT = Path(__file__).resolve().parents[1]`, which is the repo root. Data paths are relative to
  it. Scripts run from anywhere: `python scripts/<name>.py`.

## Working style
- Direct and concise. Push back when something has a problem.
- Don't write code unless I ask for it.
- Explain the reasoning behind choices so I can write it up. The interpretation must be my own, and I verify anything generated.
- Report prose: active voice, plain concrete words, no filler or AI phrases, no dashes or semicolons.
- Comment code clearly and mark each model's section in notebooks.
- Keep a running note of where gen AI helped, for item 5.

## Modelling rules
- Augment only the training set, on the fly as Keras layers.
- Tune on validation only. Touch the test set once per model at the end. Report test size honestly.
- Metrics: accuracy plus macro F1, per-class precision and recall, confusion matrix.

## Assignment

Deadline: extended (date not recorded here).

### Requirements (20 marks)
1. Own multi-class dataset (5)
2. CNN from scratch: accuracy, explain performance, reasoning for architecture and hyperparameters (5)
3. Transfer learning with a pre-trained model on the same data (5)
4. Compare the two models using architecture, hyperparameters and training plots (3)
5. Did I use gen AI, based on my Assignment 1 experience? Why or why not? (2)

Marks reward analysis and explanation, not novelty or raw accuracy. Deliverable: notebook with code, comments,
plots and written interpretation.

### Dataset
- Classes: pasta, soup, salad, cake, cookies, stir_fry.
- Sources: recipetineats.com, budgetbytes.com, cafedelites.com, loveandlemons.com, omnivorescookbook.com,
  sugarspunrun.com, natashaskitchen.com. All have an open WPRM API. Serious Eats and Sally's were dropped (API blocked
  or missing).
- `scripts/01_download_recipes.py` downloads the main photo of each recipe into `data/raw/<class>/` and writes
  `data/metadata.csv` and `data/processed.csv`. Download and label only. Cleaning, dedup and splitting are later steps.
- Collection: WP Recipe Maker REST API with batched parent post and media requests, falling back to schema.org Recipe
  JSON-LD from sitemaps. 1 to 2 s between requests, descriptive user agent, robots.txt per host (RFC 9309). Never
  bypass bot protection.
- Labelling: categories first, title as fallback, exactly one class or skip. Keyword, exclude and veto lists live in
  `CLASS_PATTERNS` at the top of the script and define the classes, so report them. `veto` drops a label only when the
  title doesn't name the class. `hard_veto` always drops it. `--strict` keeps a category label only when the title
  also names the class.
- Caps: `--max-per-class` (150 per site) and `--max-total-per-class` (300 across sites) keep classes balanced. The
  JSON-LD fallback only fetches slugs that name a class unless `--jsonld-all`. WordPress "large" image size (about
  1024 px). Repeated image URLs are skipped.
- Resumable through `processed.csv`. Delete it, or use a fresh `--out`, after changing keyword lists or switching modes.
- `scripts/02_clean.py`: drops title-rule matches (`EXCLUDE_RULES`: roundup posts, spaghetti squash, ice cream as
  cookies, pineapple cake), files in optional `data/exclude.csv`, and near-duplicates (pHash <= 4, keep first, drop all
  on a class conflict). Converts to RGB JPEG q95, shorter side <= 512. Writes `data/clean/`, `clean.csv`, `removed.csv`.
  No manual review pass (my choice), so the mislabel rate isn't measured.
- `scripts/03_split.py`: 70/15/15, seed 42, stratified by class and source site (class-site groups under 10 merge into
  "other"). Writes `data/split/{train,val,test}/<class>/` and `split.csv`. Both 02 and 03 rebuild their folders and are
  deterministic.
- Current dataset: 1,694 downloaded, 1,679 kept (pasta 291, soup 300, salad 300, cake 299, cookies 288, stir_fry 201).
  Split 1,175 / 252 / 252.
- Imbalance: keep everything, use class weights computed from the train folder, report macro F1. No undersampling.

### Things to report
- Site and class are tangled (sugarspunrun: half of cake and cookies, omnivorescookbook: most of stir_fry). Shortcut
  risk. Report accuracy per site from `split.csv`.
- Most photos are portrait (aspect about 0.7). Squashing to 224x224 distorts them, `crop_to_aspect_ratio=True` crops.
- Horizontal flips are fine for food photos (unlike album covers).
