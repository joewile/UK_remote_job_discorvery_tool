# UK Remote Analyst Job Finder

I built this to spend less time looking for remote, entry-level analyst jobs I could do from the UK. I was seeing the same vacancies across searches, and I still had to open each one to check whether “remote” meant fully remote or whether “entry-level” came with an experience requirement. The workflow does that first pass for me and sends me a ranked list by email.

## How it works

### 1. Run the search

The master workflow runs daily in n8n. I can also trigger it manually. It calls three separate workflows in order: search Indeed, assess the jobs, then email the ranked results.

<img width="673" height="268" alt="0 Workflow (Master)" src="https://github.com/user-attachments/assets/52e0916c-2b3b-4d20-b96a-51994d5ed1f1" />

### 2. Collect listings and remove duplicates

Apify runs a broad **Analyst** search on Indeed and a second search with the remote filter enabled. n8n appends the two sets of results, removes duplicate Indeed job IDs, checks the Data Table for IDs it has already seen, and inserts new rows.

<img width="667" height="209" alt="1 Workflow (Scrape, Simple Model)" src="https://github.com/user-attachments/assets/c3608961-1889-4c33-828d-bdccafccbb0a" />

### 3. Assess and rank the jobs

The ranking workflow reads vacancies from the Data Table and sends them to Gemini one at a time. I use the prompt to find roles that fit my skills and the kind of work I want to do. Gemini reads each job description, checks it against my essential criteria and flags anything I need to verify. The master workflow labels this step **Gemini 3.5**.

Each morning, I review the potential matches in my inbox, read the original adverts and decide which roles I'd want to pursue and commit to.

Each assessment returns a decision (match, review or reject), a personal fit score out of 100, and a short explanation grounded in the advert. The review category covers roles that look promising but have a requirement I still need to check. A Structured Output Parser keeps the response in a format n8n can save. The workflow writes each assessment back to the table and waits between jobs to help manage API rate limits.


<img width="690" height="343" alt="2 Workflow (Classificaiton)" src="https://github.com/user-attachments/assets/06a50c40-93fa-464b-953c-7f863735f373" />

### 4. Send the digest

The email workflow reads ranked rows from the Data Table and builds a digest. Gmail sends the message, then n8n updates the rows used for the digest.

<img width="672" height="336" alt="3 Workflow (Email)" src="https://github.com/user-attachments/assets/48f99fc3-c93a-476f-aba6-e540a26d6cc2" />

## Improvements Made 

I wanted to see whether applying the remote filter during scraping would save work without missing remote vacancies. I ran the broad and filtered searches with the same **Analyst** keyword, exported the results as CSVs, and compared job IDs for listings labelled remote.

Both searches found remote-labelled job IDs the other one missed. I changed the workflow to run both searches and deduplicate their combined results before sending descriptions to Gemini. That gave me better coverage across the two searches while avoiding repeat assessments of the same ID.

### Rerunning the comparison

My hypothesis was that running both a broad and a remote-filtered search would surface more unique remote-labelled vacancies before classification than either search on its own. I wanted to find out whether the extra listings justified keeping the broad scrape in the workflow. In this test, Apify displayed $0.05 for the broad run and $0.00 for the filtered run.

On 9 November 2026, I ran two Indeed UK searches in Apify using detailed mode:

1. **Broad search.** I used **data analyst**, the query that has surfaced the widest range of analyst-related roles in my own searches. I left the remote filter blank, sorted by relevance and kept the other settings at their defaults.
2. **Remote-filtered search.** I enabled the remote filter and used a query covering business analyst, data analyst, business intelligence, BI analyst, Power BI and reporting analyst roles.
3. 
<img width="720" height="43" alt="Broad Search Run" src="https://github.com/user-attachments/assets/402c775e-69a2-4d34-903e-c5128ba4bb0d" />
<img width="751" height="48" alt="Remote Search Run" src="https://github.com/user-attachments/assets/a0072dce-b3ec-4f43-8a39-587e5c2eab61" />

I exported both results as CSVs and checked that each row had an Indeed job ID and that neither export contained duplicate IDs. Then I compared which IDs appeared in each export. I ran the comparison again using only listings with the scraper's remote flag set to true, because those were the extra listings I wanted to measure before sending anything to Gemini.

| Scope | Broad only | In both | Filtered only | Combined unique IDs |
| --- | ---: | ---: | ---: | ---: |
| All listings | 531 | 11 | 27 | 569 |
| Listings flagged remote | 3 | 11 | 27 | 41 |

Combining the two searches raised the number of distinct remote-labelled listings from **38 to 41** compared with the filtered search alone. The broad run added **three** remote-labelled listings for the **$0.05** shown in Apify. The 11 shared listings only needed to be assessed once. That gave me a reason to keep both searches and deduplicate by job ID before classification.

The two runs used different keyword queries as well as different remote settings, so the result reflects the two search configurations together. The cost shown covers the Apify scrape. Gemini assessment is a separate cost, and the broad search also adds listings without a remote flag. I use the descriptions to check UK eligibility and office attendance during assessment.


## Outputs

- **Indeed Job Database.** The n8n Data Table stores listings and assessment results. The screenshot shows **1,331 collected Indeed job rows** and visible fields for the job ID, title, company, location and score. The total includes listings that may fall outside my criteria.
- **Ranked email digest.** Gmail sends a ranked list of newly assessed jobs. The digest puts matches first, followed by jobs marked for review and rejected jobs. Each row gives the rank, score out of 100, decision, listing link and company, salary when stated, remote or office-visit details, experience requirements and an explanation of any concerns.

### Example email

This run assessed **240 jobs**. The screenshot shows the first five rows, all marked **review**. For example, the Financial Analyst advert describes the job as hybrid and says it requires no prior professional experience, but it gives no office attendance schedule. The assessment flags that gap so I know what to check before applying. The score reflects how closely the role fits my search criteria.

<img width="797" height="639" alt="Emailed Output" src="https://github.com/user-attachments/assets/43c34b39-76d2-414a-9da6-69399456b8b8" />

## Stack and data handling

n8n runs the workflow and stores the listings and assessments. Apify collects Indeed listings, Google Gemini assesses the descriptions, and Gmail sends the digest. JavaScript nodes prepare records and assemble the email.

Job descriptions go to Gemini for assessment. API keys, OAuth credentials, recipient details and private configuration belong in the private n8n setup.

## Limitations

Indeed and the scraper only cover the listings they return for those searches. The comparison shows how the two result sets differed in this experiment. It doesn’t estimate how common fully remote analyst jobs are across the UK.

Remote labels can be ambiguous, and descriptions can omit details about office visits or experience requirements. Gemini can also misread those details, so I use the score to decide what to read first and confirm the requirements with the employer. Deduplication by job ID catches overlap across searches, while reposts with new IDs can still appear.

