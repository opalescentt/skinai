# SkinAI

## Description

Skin AI (name TBD) is a mobile app that allows users to track their skincare routine, changes in their skin and different factors to learn over time what their skin reacts to well and hwat causes problems.

### Problem

People use 5-10 products, change several at once, and judge results by feel. Skin also shifts with season, hormones, stress and diet, so the cause of a breakout or a good week is hard to see. Existing apps scan ingredient lists but rarely connect them to what happened to your own skin.

**Who it is for.** Someone with a pre-existing skincare routine and skin concerns such as acne, dryness or hyperpigmentation, willing to log for 4-8 weeks.

**What it does, in one line.** You log products and how your skin looks each day. Ask "what changed before my breakout?" and the app checks your log for timing patterns, looks up the ingredients involved, and answers with sources and honest uncertainty.

### Prototype success criteria

- On synthetic logs with planted effects, the agent names the right product in at least 70% of cases (starting target; revise after the first run).
- On a 25-30 question test set, answers are grounded in cited sources, and questions outside the corpus get "I don't know" instead of a guess.
- Logging one day takes under one minute.
- The app never diagnoses a condition and always routes red-flag symptoms to a professional.

## Requirements

### User Stories

| Priority | User Story |
|---|---|
| P0 | As a user, I want to be able to log which skin products I used today and at what time of the day. |
| P0 | As a user, I want to be able to view information about the ingredients each of my skincare products. |
| P0 | As a user, I want to be able to take pictures of my skin to include in my daily log. |
| P0 | As a user, I want to receive a rating out of 5 on my skin based on a picture from the app, and add in my own personal rating. |
| P0 | As a user, I want to receive insights into what products could be benefiting or harming my skin based on the ingredients and my personal logs. |
| P0 | As a user, I want to know how confident the app is, and when it cannot tell, I am told why. |
| P1 | As a user, I want to conduct and track experiments of custom lengths to focus on what the introduction of a new skin product could be doing to my skin. |
| P1 | As a user, I want to be able to view potential beneficial or harmful interactions between my skincare products. |
| P2 | As a user, I want to be able to track non-product related triggers, such as diet and weather as part of the app's analysis. |

### Non-functional requirements

- Every insight shows its confidence level and its sources.
- The app never offers diagnosis or treatment advice.
- A photo that is too dark, blurry or off-angle gets a retake request instead of a rating.

### AI Model Components

| # | Component | Supports | Priority | Effort |
|---|---|---|---|---|
| B1 | Ingredient database and name normalization | F1, F3 | Must | Medium |
| B2 | Analysis engine: before/after comparison, lag, check for overlapping changes | F5, F6, F8 | Must | Large |
| B3 | Knowledge base with sources, for citations | F3, F5, F7 | Must | Medium |
| B4 | Photo quality check (too dark, blurry, off-angle) | F4 | Must | Medium |
| B5 | Safety layer: red-flag symptoms, non-diagnostic wording | Non-functional | Must | Small |
| B6 | Interactions file | F9 | Should | Small |
| B7 | Test tooling: synthetic data generator, evaluation set | Success criteria | Must (dev only) | Medium |

## Data and knowledge base

The knowledge base has three layers plus a personal log, because ingredient questions mix exact lookups with explanations.

1. **Structured layer:** Products, ingredients, aliases and functions. Answers "what is in this?" with no model involved.
2. **Text corpus:** Authoritative passages chunked by ingredient and topic, with source metadata. Answers "why" and "how" questions.
3. **Curated interactions file (YAML or JSON):** 20-30 hand-written, sourced conflicts and cautions. Checked directly by a tool so it never depends on retrieval luck.
4. **Personal log:** The user's product usage and skin check-ins over time.

**Starting sources.** Open Beauty Facts (product ingredient lists), CosIng (ingredient functions), Cosmetic Ingredient Review assessments, PubMed Central open-access reviews, and Wikipedia ingredient pages. Access method, format and license for each are verified during the data spike (Nov 23 - Dec 6) using the deep research report, and any source that cannot be downloaded or whose license blocks storage is dropped.

**Starting scope.** About 50 actives and 200 products, focused on two or three concerns (acne, dryness, hyperpigmentation).

### Structured schema

| Table | Key fields |
|---|---|
| `ingredient` | id, inci_name, common_name, functions, concern_tags, evidence_level |
| `ingredient_alias` | alias, ingredient_id |
| `product` | id, name, brand, source, source_url, version_date |
| `product_ingredient` | product_id, ingredient_id, position |
| `interaction` | ingredient_a, ingredient_b, effect, severity, source_url, evidence_level |
| `usage_log` | date, product_id, time_of_day |
| `skin_checkin` | date, rating, breakouts, dryness, redness, note |

**Chunk metadata (every chunk).** ingredient, topic (mechanism, efficacy, side effects, interactions), source, source_url, publication date, evidence type (review, trial, assessment, encyclopedia).

**Synthetic data for testing.** A generator writes 8 weeks of daily logs with random noise and 2-3 planted effects, such as "product X causes breakouts after 7-10 days of use". Because the truth is known, the agent can be scored on whether it finds it.

**Evaluation set.** 25-30 questions with expected answers, written before tuning, including several that should return "I don't know".

## System and AI design

The model never does the math or recalls facts on its own: it calls tools for data and statistics, then explains what they returned. A question passes from the interface through the safety layer to the agent, which calls tools and answers only from what they return; the answer travels back up the same path.

### Agent Tools

| Tool | What it does | Returns |
|---|---|---|
| `lookup_product(name)` | Queries the ingredient database | Normalized ingredient list with functions |
| `search_knowledge(query, ingredient)` | Retrieves from the corpus, filtered by metadata | Top passages with source links |
| `check_interactions(items)` | Reads the curated interactions file | Conflicts, severity, source (or "no data") |
| `get_user_log(start, end)` | Reads product usage and skin check-ins | Rows for the date range |
| `analyze_change(event_date, window)` | Runs code: products started or stopped, before/after averages, lag correlation | Numbers with sample sizes and a confounding warning |

### Rules in the system prompt

- Answer only from tool results; if none support an answer, say so.
- Cite a source for every factual claim about an ingredient.
- State confidence and data limits. With under three weeks of data, or several products changed at once, say the signal is weak.
- Use cautious wording ("commonly associated with"), never "causes" or "cures".
- No diagnosis and no treatment or dosing advice. Red-flag symptoms get a recommendation to see a professional (the exact list comes from the research report; examples to confirm include spreading swelling, signs of infection, eye involvement, and a changing mole).

### Evaluation plan

- **Retrieval:** for each test question, whether the right chunk appears in the top 5.
- **Answers:** correct, cited, and refusing when the corpus has nothing.
- **Change analysis:** hit rate on synthetic logs against the Vision target.
- **Safety:** every red-flag and diagnosis-request test case gets a referral response.
- Rerun all four after every change and keep a simple scoreboard.

## UX design

The interface has three screens and one daily habit: open the app, log in under a minute, and ask a question when curious.

### Core daily flow

1. Open the Today screen.
2. Tick the products used this morning or evening (the routine is pre-filled from yesterday).
3. Fill the check-in: rating, breakouts, dryness, redness, optional note.
4. Save. The app shows one short line, such as "Day 12 logged. New product started 5 days ago."

### Screens (low-fidelity wireframes, to be sketched in the design phase)

| Screen | Layout | Key elements |
|---|---|---|
| Today | Single column. Routine checklist on top, check-in form below, Save button at the bottom | Products grouped AM and PM; 1-5 rating buttons; number box for breakouts; sliders for dryness and redness; note field |
| Ask | Chat layout. Message box at the bottom, answers above | Each answer shows source links and a confidence line; suggested questions when the chat is empty; a visible "not medical advice" note on red-flag responses |
| Product | Search box, then a result card | Normalized ingredient list with each active's function; interaction warnings with other products in your routine |
| History (post-MVP) | Timeline of ratings with product start and stop markers | Hover a day to see what you used |

## Timeline

### Phase Deadlines

| Phase | Weeks | Dates |
|---|---|---|
| Experimentation: data and evaluation | 1-2 | Nov 2 - Nov 15 |
| Experimentation: polish and tooling | 3-4 | Nov 16 - Nov 29 |
| Design | 5-6 | Nov 30 - Dec 13 |
| Build sprint 1: log and ingredients | 7-8 | Dec 14 - Dec 27 |
| Build sprint 2: insights | 9-10 | Dec 28 - Jan 10 |
| Build sprint 3: photo rating | 11-12 | Jan 11 - Jan 24 |
| Build sprint 4: remaining features and testing | 13-14 | Jan 25 - Feb 7 |

### Experimentation: data and evaluation (Nov 2 - 15)

**By Nov 8**

- [ ] Run the deep research report
- [ ] Choose the 2-3 skin concerns the knowledge base will cover
- [ ] Write 25-30 test questions with expected answers, including some that should return "I don't know"
- [ ] Verify access and license for each data source
- [ ] Start taking a daily photo and rating your own skin out of 5 (this becomes the test data for photo rating later)

**By Nov 15**

- [ ] Download the ingredient and product data
- [ ] Download the text sources
- [ ] Build the ingredient alias table and normalize ingredient names
- [ ] Chunk the text by ingredient and topic, with source metadata on every chunk
- [ ] Run the first retrieval evaluation and record the scores

### Experimentation: polish and tooling (Nov 16 - 29)

**By Nov 22**

- [ ] Improve chunking and retrieval based on the evaluation results
- [ ] Build the product lookup tool
- [ ] Build the knowledge search tool
- [ ] Write the interactions file (20-30 sourced entries)

**By Nov 29**

- [ ] Build the interaction check tool
- [ ] Build the synthetic data generator with planted effects
- [ ] Build the before/after analysis tool
- [ ] Add "I don't know" behavior and confidence rules
- [ ] Rerun the evaluation and compare to the first scores
- [ ] Gate check: answers are cited, out-of-scope questions get "I don't know", and the analysis finds the planted product in at least 70% of synthetic logs. If not, extend this phase before designing.
- [ ] (Optional) Test whether a model rates the same photos consistently

### Design (Nov 30 - Dec 13)

**By Dec 6**

- [ ] Write the user flows
- [ ] Decide where photos are processed (on the phone or sent to an AI service)
- [ ] Choose the iOS approach and where the backend runs
- [ ] Settle the app name, or decide to defer it

**By Dec 13**

- [ ] Wireframe the core screens: daily log, photo capture, ingredient view, insights
- [ ] Define the app's data model
- [ ] Write acceptance criteria for the skin rating, insights and confidence stories
- [ ] Write the end-of-phase note

### Build sprint 1: log and ingredients (Dec 14 - 27)

**By Dec 20**

- [ ] Set up the iOS project and backend
- [ ] Build the product library (add a product by name and ingredient list)
- [ ] Build the ingredient view

**By Dec 27**

- [ ] Build the daily log (products used, morning or evening)
- [ ] Add your own daily rating out of 5
- [ ] Check that logging a day takes under a minute
- [ ] Write the end-of-sprint note

### Build sprint 2: insights (Dec 28 - Jan 10)

**By Jan 3**

- [ ] Connect the analysis tools to the app
- [ ] Build the insights screen with sources

**By Jan 10**

- [ ] Show a confidence level and reason on every insight
- [ ] Build the safety layer (red-flag symptoms, non-diagnostic wording)
- [ ] Confirm all safety test cases produce a referral
- [ ] Write the end-of-sprint note

### Build sprint 3: photo rating (Jan 11 - 24)

**By Jan 17**

- [ ] Add photo capture and attach photos to the daily log
- [ ] Build the photo quality check (too dark, blurry, off-angle)

**By Jan 24**

- [ ] Rate skin out of 5 from a photo
- [ ] Compare against your own ratings (target: within 1 point in at least 70% of logs)
- [ ] Write the end-of-sprint note

### Build sprint 4: remaining features and testing (Jan 25 - Feb 7)

**By Jan 31**

- [ ] Ask questions in chat
- [ ] Experiments of custom length

**By Feb 7**

- [ ] Product interactions view
- [ ] View, edit and delete past logs and photos
- [ ] Use the app on your own routine
- [ ] Write your findings and a go/no-go on next steps

**Each sprint ends with a short written note on what worked, what broke and what changes next.**

## Risks, open questions and next steps

The biggest risk is that skin outcomes are too noisy for any model to find real causes in a few weeks of data, so the prototype is built to test that early.

| Risk | Why it matters | Mitigation |
|---|---|---|
| Confounding (season, hormones, diet, stress) | The app could blame the wrong product | Nudge one new product at a time; report confidence; warn when several changes overlap |
| Too little real data | Four weeks of one person's log may show nothing | Validate on synthetic data first; treat the real-use test as a learning exercise |
| Data sources unavailable or restricted | Plan depends on downloadable, usable data | Verify access and license in the data spike; keep a manual fallback of 200 products |
| Model states things not in the sources | Wrong skincare claims, possible harm | Citation required; refuse when retrieval is weak; test "I don't know" cases |
| Drifting into medical advice | Safety and regulatory exposure | Safety layer, non-diagnostic phrasing, red-flag referral from day one |
| Scope creep | Photos and recommendations can swallow the timeline | Non-goals list; cut features 9-10 first |
| Time capacity | The timeline assumes 8-10 hours per week | Sprint 3 (Jan 4 - 17) is the flex buffer |

### Open questions

- Which 2-3 concerns does the MVP cover?
- Which LLM API and embedding model will be used, and what is the monthly cost cap?
- Is this for personal use only, or might other people use it? (This changes the license and privacy decisions.)
- Which weekly hours are actually available around other commitments?

### Next steps

- Launch the deep research report using the prompt drafted earlier (this week)
- Choose the 2-3 concerns and write the final scope sentence (by Nov 8)
- Start your own daily skin log in a simple sheet so real data exists by January (this week)
- Verify and download the first data sources (data spike, Nov 23 - Dec 6)
