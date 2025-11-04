# IS7024 Group Project
### Clay Caddell, Jose Esquivel, Karthik Raturi
![Logo](/LingaNews.png)
# LinguaNews

## 📰 Introduction
LinguaNews teaches reading in a target language using real news articles. Imagine the ability to build a workable list of vocabulary using real-world news articles that are updated in real-time and rank/sort various articles based on difficulty and word frequency.

---

## 📡 Data Sources

- [NewsAPI](https://newsapi.org/)  
- [DeepL Dictionary / Translation API](https://www.deepl.com/en/products/api)  
- [Cathoven CEFR Word Difficulty Database](https://www.cathoven.com/)  
- [1000 Most Common Words API](https://rapidapi.com/vicscodes/api/1000-most-common-words) (used for difficulty heuristics)  

---

## Team Meeting Structure

- Ad-hoc meetings to complement Sunday weekly touchpoint via MS Teams

---

## Requirements (User Stories)

1. As a language learner, I want to read real news articles in my target language so that I can improve my vocabulary and comprehension.  
2. As a user, I want to quickly view translations, save words and review them later so that I can retain new vocabulary.  
3. As a user, I want to filter articles by vocabulary, topic and difficulty so that I can find content that matches my level and interests.


## 0.1 Onboarding and Target Language Assignment

**Given** a user has created an account and associated their email (used as primary key)  
**When** the user selects `TargetLanguage = Spanish` during onboarding  
**Then** the system should:
- Update the user profile with the selected target language
- Persist the `TargetLanguage` value for future sessions
- Use the assigned language to filter articles in `/Articles/Feed`


### Edge Case: No Language Selected
**Given** a user completes onboarding without selecting a target language  
**Then** the system should:
- Prompt the user to choose a language before accessing `/Articles/Feed`
- Default to a fallback language (e.g., English) if allowed by business rules
- Log the missing preference for analytics or follow-up


### Error Case: Invalid Language Code
**Given** a user selects a language not supported by NewsAPI or the application (e.g., `"Klingon"`)  
**Then** the system should:
- Display a validation error or fallback message
- Prevent submission until a valid language is selected
- Avoid storing unsupported values in the user profile

---

## 1.1 Article ingestion and feed

### Article Feed Behavior
**Given** a logged-in user with `TargetLanguage = Spanish`  
**When** they open `/Articles/Feed`  
**Then** NewsAPI is configured with language selection `"ES"`  
And populates a list of Spanish articles sorted by recency and filtered by the user’s difficulty and topic preferences.


### Normalization of Article Data
**Given** the NewsAPI is configured with the User's 'TargetLanguage' 
**When** the ingest job runs for the next 7 days of content  
**Then** new normalized Article records are created with:
- `Title`
- `SourceUrl`
- `ContentSnapshot`
- `Language`
- `Publisher`
- `DifficultyEstimate`
- creation metadata


### Edge Case: No Articles Available 
**Given** the user has selected `"Italian"` as target language  
And NewsAPI has no recent articles in Italian  
**When** the article list attempts to load  
**Then** the application should:
- Display message `"No articles available in Italian at this time"`
- Suggest trying a different language or category
- Not display an error message or crash

### Error Case: API Rate Limit Exceeded
**Given** the user has made 95 API requests today (NewsAPI free tier limit: 100/day)  
And the user changes language 6 more times  
**When** the 101st API request is made  
**Then** the application should:
- Display message `"API rate limit reached. Using cached articles."`
- Load articles from local cache if available
- Not crash or hang


---

## 1.2 Article Scraping and Content Snapshot

### Snapshot Creation
**Given** the ingest worker retrieves article HTML from a source  
**When** normalization runs  
**Then** the system stores:
- `ContentSnapshot` - a paginated version of the article's text data
- Source attribution
- `ArticleVocabList` - a list of all words in the Article to feed to DeepL and/or cache in 'UserWord'


### Change of Article Data 
**Given** a previously ingested Article has changed at the source  
**When** NewsAPI triggers a weekly re-ingest  
**Then**:
- The snapshot and `ArticleVocabList` are updated
- A new ingest log entry is created
- Quizzes / review lists that used the older snapshot remain linked to the old version until manually updated


### Edge Case: Duplicate Article from Source

**Given** the ingest worker retrieves article HTML from a source  
And the same article has already been normalized and stored  
**When** normalization runs again on the duplicate content  
**Then** the system should:
- Detect the duplicate based on `SourceUrl` and content hash
- Skip re-creating the `Article` record unless content has changed
- Log the duplicate detection in the ingest log for audit purposes


### Edge Case: Article with Missing Body Content

**Given** NewsAPI returns an article with a valid title and metadata  
But the body content is empty or malformed  
**When** normalization runs  
**Then** the system should:
- Flag the article as incomplete
- Store available metadata (`Title`, `SourceUrl`, `Publisher`) with a warning
- Exclude the article from feed and vocabulary extraction
- Log the issue for review without crashing the ingest job


### Error Case: HTML Parsing Failure

**Given** the ingest worker retrieves article HTML from a source  
And the HTML structure is invalid or causes a parsing exception  
**When** normalization attempts to extract content  
**Then** the system should:
- Log the parsing error with source reference
- Skip normalization for that article
- Continue processing other articles without halting the job


### Error Case: Vocabulary Extraction Timeout

**Given** an article is successfully normalized  
And the vocabulary extraction step calls DeepL or internal NLP services  
**When** the extraction process exceeds the timeout threshold  
**Then** the system should:
- Save the article with partial metadata and `ContentSnapshot`
- Mark `ArticleVocabList` as incomplete or pending
- Retry extraction in the next scheduled ingest cycle
- Avoid blocking or retrying indefinitely during the current 


---

## 2.1 Save word and vocabulary tracking

### Save Word
**Given** a user clicks Save Word in the reader  
**When** the save action completes  
**Then** a `UserWord` record is created or updated with:
- `AddedAt`
- `WordDifficulty`
- DeepL Translation Data  
And the 'UserWord' appears on `/Vocabulary`

### Edge Case: Save Word Already Exists

**Given** a user clicks Save Word in the reader  
And the word has already been saved previously to '/Vocabulary'
**When** the save action completes  
**Then** the existing `UserWord` record should be updated with:
- New `AddedAt` timestamp (if applicable)
- Refreshed `WordDifficulty` based on latest context
- Updated DeepL Translation Data (if changed)  
And the word remains visible on `/Vocabulary` without duplication

### Error Case: DeepL API Failure During Save

**Given** a user clicks Save Word in the reader  
And the DeepL API is temporarily unavailable (e.g., 500 error or timeout)  
**When** the save action attempts to fetch translation data  
**Then** the application should:
- Display message `"Translation temporarily unavailable"`
- Save the word with partial data (e.g., `AddedAt` and placeholder for translation)
- Retry translation later or allow manual update  
And not crash, hang, or block the save action


---

## 2.2 Inline reader and word lookup

### Word Lookup
**Given** a user opens `/Articles/Read/{ArticleID}` for an article in their target language  
**When** they click or hover a word  
**Then**:
- DeepL fetches translation data
- A popover displays the word’s lemma, translation, part of speech, and one example sentence from the article or dictionary cache


### Deduplication / Caching
**Given** the DeepL dictionary/translation API result is cached in 'UserWord' 
**When** the user requests the lookup  
**Then** the cached response is returned and the external API is not called


### Example: Click Common/Beginner Word
**Given** the user is reading a Spanish article  
And the word `"casa"` (house) appears in the text  
**When** the user clicks on the word `"casa"`  
**Then** the vocabulary panel should open on the right side  
And:
- Display `"casa"` as the headword
- Show definition: `"Edificio para habitar"` (in Spanish)
- Show translation: `"house"` (in English, user's native language)
- Indicate high frequency rank and/or `"Beginner"` Difficulty


### Example: Click Rare/Advanced Word
**Given** the user is reading a French technology article  
And the word `"blockchain"` appears in the text  
**When** the user clicks on `"blockchain"`  
**Then** the vocabulary panel should open  
And:
- Display `"blockchain"` as a technical term
- Show definition from DeepL
- Indicate low frequency rank and/or `"Advanced"` Difficulty


### Edge Case: Click Non-Dictionary Word 
**Given** the user is reading an article  
And the text contains a proper noun `"García"` (person's name)  
**When** the user clicks on `"García"`  
**Then** the vocabulary panel should open  
And:
- Display message `"Proper noun detected"`
- Offer no translation but note that it's a name
- Not provide a traditional definition

### Error Case: API Failure During Definition Lookup
**Given** the user clicks a word to view definition  
And the DeepL API is temporarily unavailable (500 error)  
**When** the vocabulary panel attempts to load  
**Then** the application should:
- Display `"Definition temporarily unavailable"`
- Show cached definition if available from previous lookup
- Not crash or hang indefinitely

---

## 3.1 Article Search and Filtering

### Vocabulary Search
**Given** the user is viewing Spanish articles  
And the search bar is empty  
**When** the user types `"tecnología"` and presses Enter  
**Then** the application should:
- Send search query to NewsAPI with keyword `"tecnología"` and language `"es"`
- Display only articles matching the search term
- Highlight the search term in article titles/descriptions
  

### Clear Search
**Given** the user has searched for `"football"`  
And >0 articles are currently displayed  
**When** the user clicks the `"X"` button in the search bar  
**Then**:
- The search bar should clear
- All available articles should be displayed again (not just sports)


### Edge Case: No Search Results
**Given** the user is viewing French articles  
**When** the user searches for `"xyz123nonsenseword"`  
**Then** the application should:
- Display `"No results found for 'xyz123nonsenseword'"`
- Suggest checking spelling or trying different keywords
- Provide a `"Clear Search"` button to return to all articles


---

## 3.2 Difficulty Ranking

### Vocabulary Ranking
**Given** NewsAPI is configured and fetches articles in the User’s target language  
**When** they are ingested and the `ContentSnapshot` normalizes the data  
**Then** words added to the `ArticleVocabulary` are assigned:
- Beginner
- Intermediate
- Advanced  
Based off of their rating in the CEFR database if available

### Article Ranking
**Given** CEFR rankings are assigned to words in the article
**When** the articles are ingested and the `ContentSnapshot` normalizes the data  
**Then** the article is assigned a difficulty rating:
- Beginner
- Intermediate
- Advanced  
Based off of the frequency of appearance of CEFR-rated words


### Edge Case: Word Not Found in CEFR Database

**Given** NewsAPI fetches an article in the user's target language  
And the article contains uncommon or newly coined words  
**When** the `ContentSnapshot` is normalized  
**Then** the system should:
- Assign a default difficulty level based on fallback heuristics (e.g., word frequency, sentence complexity)
- Flag the word as `"Unranked"` or `"Unknown"` in the `ArticleVocabulary`
- Allow the word to appear in the feed and vocabulary list with a note that CEFR data is unavailable
  And the word will not be counted when assigning article difficulty
  

### Error Case: CEFR Lookup Failure

**Given** the system attempts to assign CEFR-based difficulty rankings during normalization  
And the CEFR database or API is temporarily unavailable or returns an error  
**When** vocabulary extraction runs  
**Then** the system should:
- Log the failure with word and article reference
- Skip CEFR tagging for affected words but continue ingesting the article
- Mark the vocabulary entry as `"Difficulty pending"` or `"Estimation incomplete"`
- Retry CEFR tagging in the next scheduled ingest cycle without blocking the current job


---

## 3.3 Difficulty Filtering

### Filtered 'Articles/Feed'

**Given** a user applies difficulty filters on `/Articles/Feed` (beginner, intermediate, advanced)  
**When** they submit the filter  
**Then** the feed returns only URLs to Articles matching the filters


### Edge Case: No Articles Match Selected Difficulty

**Given** a user applies a difficulty filter (e.g., Advanced)  
And no articles in the current feed match that difficulty level  
**When** the filter is submitted  
**Then** the application should:
- Display the message `"No articles found for selected difficulty level"`
- Offer suggestions to adjust filters or try a different category
- Avoid showing an empty or broken layout


### Error Case: Difficulty Metadata Missing

**Given** the user applies difficulty filters  
And one or more articles lack `DifficultyEstimate` due to ingest or ranking failure  
**When** the feed attempts to apply the filter  
**Then** the application should:
- Exclude articles with missing difficulty metadata from filtered results
- Log the missing metadata for ingest review
- Display a fallback message such as `"Some articles could not be ranked for difficulty"`
- Continue rendering the feed without crashing or blocking the 
