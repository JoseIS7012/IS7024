# IS7024
Repository for IS7024 Group Project


# LinguaNews

## 📰 Introduction
LinguaNews teaches reading in a target language using real news articles. Imagine the ability to build a workable list of vocabulary using real-world news articles that are updated in real time and rank various articles and vocabulary based on difficulty and frequency.

---

## 📡 Data Sources

- NewsAPI  
- Dictionary / Translation API (DeepL)  
- CEFR Word Difficulty Database (Cathoven API)  
- [1000 Most Common Words API](https://rapidapi.com/vicscodes/api/1000-most-common-words) (used for difficulty heuristics)  
- Quiz building API (e.g., quizapi.io)

---

## Team Meeting Structure

- Ad-hoc meetings to complement Sunday weekly touchpoint via MS Teams

---

## Requirements (User Stories)

1. As a language learner, I want to read real news articles in my target language so that I can improve my vocabulary and comprehension.  
2. As a user, I want to save words and review them later so that I can retain new vocabulary.  
3. As a user, I want to filter articles by topic and difficulty so that I can find content that matches my level.

---
## 1.1 Article ingestion and feed

### Article Feed Behavior
**Given** a logged-in user with `TargetLanguage = Spanish`  
**When** they open `/Articles/Feed`  
**Then** NewsAPI is configured with language selection `"ES"`  
And populates a list of Spanish articles sorted by recency and filtered by the user’s difficulty and topic preferences.

### Normalization of Article Data
**Given** the NewsAPI is configured  
**When** the ingest job runs for the next 7 days of content  
**Then** new normalized Article records are created with:
- `Title`
- `SourceUrl`
- `ContentSnapshot`
- `Language`
- `Publisher`
- `DifficultyEstimate`
- creation metadata

### No Articles Available (Edge Case)
**Given** the user has selected `"Italian"` as target language  
And NewsAPI has no recent articles in Italian  
**When** the article list attempts to load  
**Then** the application should:
- Display message `"No articles available in Italian at this time"`
- Suggest trying a different language or category
- Not display an error message or crash

### API Rate Limit Exceeded (Error Case)
**Given** the user has made 95 API requests today (NewsAPI free tier limit: 100/day)  
And the user changes language 6 more times  
**When** the 101st API request is made  
**Then** the application should:
- Display message `"API rate limit reached. Using cached articles."`
- Load articles from local cache if available
- Not crash or hang

---

## 1.2 Article normalization and snapshot

### Snapshot Creation
**Given** the ingest worker retrieves article HTML from a source  
**When** normalization runs  
**Then** the system stores:
- Cleaned `ContentSnapshot`
- Source attribution
- `ArticleVocabList`
- Difficulty estimate  
And preserves the original source URL for attribution.

### Change of Article Data
**Given** a previously ingested Article has changed at the source  
**When** an admin triggers a manual re-ingest  
**Then**:
- The snapshot and `ArticleVocabList` are updated
- A new ingest log entry is created
- Quizzes / review lists that used the older snapshot remain linked to the old version until manually updated

---

## 2.1 Search and filtering

### Successful Search
**Given** the user is viewing Spanish articles  
And the search bar is empty  
**When** the user types `"tecnología"` and presses Enter  
**Then** the application should:
- Send search query to NewsAPI with keyword `"tecnología"` and language `"es"`
- Display only articles matching the search term
- Highlight the search term in article titles/descriptions

### No Search Results
**Given** the user is viewing French articles  
**When** the user searches for `"xyz123nonsenseword"`  
**Then** the application should:
- Display `"No results found for 'xyz123nonsenseword'"`
- Suggest checking spelling or trying different keywords
- Provide a `"Clear Search"` button to return to all articles

### Clear Search
**Given** the user has searched for `"football"`  
And 12 sports articles are currently displayed  
**When** the user clicks the `"X"` button in the search bar  
**Then**:
- The search bar should clear
- All available articles should be displayed again (not just sports)

---

## 2.2 Inline reader and word lookup

### Word Lookup
**Given** a user opens `/Articles/Read/{ArticleID}` for an article in their target language  
**When** they click or hover a word  
**Then**:
- DeepL fetches translation data
- A popover displays the word’s lemma, translation, part of speech, and one example sentence from the article or dictionary cache

### Deduplication / Caching
**Given** the DeepL dictionary/translation API result is cached  
**When** the user requests the lookup  
**Then** the cached response is returned and the external API is not called

---

## 2.3 Difficulty Filtering

### Vocabulary Ranking
**Given** NewsAPI is configured and fetches articles in the User’s target language  
**When** they are ingested and the `ContentSnapshot` normalizes the data  
**Then** words added to the `ArticleVocabulary` are ranked:
- Beginner
- Intermediate
- Advanced  
Based off of their frequency using the 1000 most common words dataset

### Filtered Feed
**Given** a user applies difficulty filters on `/Articles/Feed` (beginner, intermediate, advanced)  
**When** they submit the filter  
**Then** the feed returns only URLs to Articles matching the filters

---

## 3.1 Save word and vocabulary tracking

### Save Word
**Given** a user clicks Save Word in the reader  
**When** the save action completes  
**Then** a `UserWord` record is created or updated with:
- `AddedAt`
- `NextReviewAt`
- `ReviewInterval`
- `WordDifficulty`  
And the word appears on `/Vocabulary`

### Review Session
**Given** a saved `UserWord` is due for review today  
**When** the user opens `/Vocabulary` and starts a review session  
**Then**:
- The word appears in the review queue with article context
- The user can mark Easy / OK / Hard, which updates `NextReviewAt`, `WordDifficulty`, and `ReviewInterval`

---

## 3.2 Difficulty estimation and scaffolding (CEFR Word Difficulty Database/API?)

### Difficulty Estimation
**Given** an article is normalized  
**When** difficulty heuristics run (sentence length, uncommon-word ratio, named-entity density)  
**Then** the Article is assigned a `DifficultyEstimate` and a CEFR-style label (e.g., A2, B1)

### Scaffolding
**Given** an article’s `DifficultyEstimate` is above a user’s `ProficiencyLevel`  
**When** the user opens the article  
**Then** the UI suggests scaffolding options:
- Simplified glossary
- Pre-reading list of key words
- Shorter excerpt

