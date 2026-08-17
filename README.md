## VAT-Identifier-Discovery
## Defining the problem:
The core objective of this project is to determine whether a UK company VAT dataset can be constructed from open web sources, well enough to be offered as a product. A procurement team managing 40,000 UK suppliers already holds VAT numbers for a third of them; for the rest, suppliers are matched by company name alone, which is error-prone due to string variations - making the VAT number the only reliable identifier that appears on both the invoice and the tax record.
The problem looked simple at first, but three things make it harder than a standard lookup task.
**First,** HMRC's VAT checker only works backwards: it can confirm whether a VAT number I already have is valid and matches a company, but it cannot return a VAT number given a company name. There is no reverse lookup to rely on - discovery and verification are two separate problems.
**Second,** not every company is VAT-registered. This means a "not found" result is ambiguous: it could mean the company genuinely has no VAT number, or it could mean my method failed to find one that exists. Distinguishing between these two cases matters for reporting honest results.
**Third,** the two types of errors this process can produce are not equally costly. A missing VAT number is a visible gap - easy to spot and flag. A wrong VAT number attached to the wrong company is invisible unless checked, and it silently corrupts every downstream join that relies on it. Because there is no complete reference dataset to validate against, avoiding this second type of error is the harder and more important constraint to design around.

---

## Part 1: Data Profiling, Sampling Methodology & Research
**1. Initial Dataset & Memory Management:** 
The starting point was the Companies House `BasicCompanyDataAsOneFile-2026-08-01.csv` dataset. Due to the massive file size (over 2.7 GB), I processed the data in chunks using Python/Pandas (`chunksize=250000`, `low_memory=False`) to ensure memory efficiency and avoid dtype inference errors, particularly with alphanumeric Scottish/Northern Irish company numbers. I filtered the dataset to include only companies with an 'Active' status.

**2. Data Profiling & Internal Dead Ends:** 
Before sampling, I analyzed the `SICCode.SicText_1` (industry domain) column to assess data quality.
* **Total active companies analyzed:** 5,190,464
*	**Total unique valid industries discovered:** 991
*	**Data Quality Issue:** I identified that 216,285 active companies (4.17% of the dataset) had invalid industry categorizations, explicitly marked as "None Supplied".
*	**Conclusion:** I dropped these records from the active dataframe. First, because without a declared industry, they cannot be included in the top 10 sectors analysis. Second, companies that fail to declare their basic activity are usually very small businesses, which have a very low chance of being registered for VAT.

3. **Stratified Sampling Strategy:**
To build a highly representative "Proof of Concept" sample for VAT number extraction, a random sample from the 5.19 million companies would be statistically inefficient. Instead, I applied a stratified sampling approach:
*	**Top 10 Industries:** I aggregated the cleaned data using `.value_counts()` and isolated the top 10 largest industries in the UK by volume. Targeting these core sectors maximizes the probability of finding active B2B entities that meet the £90,000 mandatory VAT registration threshold.
*	**Final Extraction:** I grouped the data by these 10 industries and randomly sampled exactly 10 companies from each (`random_state=36`).
*	**Result:** A perfectly balanced, clean sample of 100 companies, providing a manageable and highly relevant subset for manual investigation before attempting to write an automated scraping algorithm.

4. **The Trail & The Dead Ends (Manual Investigation):**
What wasn't obvious at the start is that a 'Not Found' result is rarely a scraping failure; it is usually an accurate reflection of how public data is restricted.
### Dead End 1: Paywalls and the "Ambiguous NULL"
I started searching on Google for Company 1: SELBUD LIMITED (14918826) using a strict search query: `"SELBUD LIMITED 14918826 VAT"`. The result was 0 documents, proving that strict searches fail if the company doesn't have a strong online presence. Searching just by name and number led me to the official government website, Companies House. The "Filing history" tab contained free PDFs of "Micro company accounts." While official, this data is unstructured, and micro-entities are legally allowed to exclude tax data from these documents. This led me to commercial data platforms like Endole. Endole provides a lot of value by reading those official PDFs and displaying the information clearly on their platform, but because it is a private business, it blocks the company's contact details and website links behind a premium paywall.

> **Why it failed:** Official government records keep data unstructured in PDFs and legally hide tax details for small companies. Commercial platforms like Endole structure the financial data, but put the contact details behind a paywall and still cannot provide the missing VAT number. Furthermore, UK law allows companies to register voluntarily for VAT even below the £90k threshold. Because the VAT data is missing from both official PDFs and commercial platforms, an automated "Not Found" result is unclear: I cannot know if the VAT number truly doesn't exist, or if it exists but is legally hidden.

### Dead End 2: Private Websites and Fake Data
I hoped private company websites would have the missing VAT numbers, so I tested `vat-search.co.uk` and `vatverifier.com`.

> **Why it failed:** These websites cannot be trusted. When I searched for Best House Property LTD on vat-search.co.uk, it just said no VAT ID available. But when I searched for KOF Technology Services Limited on the same site, it showed an "Active" status and a "GB..." prefix, hiding the rest of the number behind an account creation prompt. To check if this hidden number was real, I searched for the company's VAT on Google and found nothing at all. The second site, vatverifier.com, did the same thing, showing a blurred "GB12..." prefix for TEARAINTE LIMITED. I downloaded the official PDF document for this company from Companies House, which showed 0 employees and under £15k in assets. This suggests these private websites use partially-hidden numbers to prompt account creation. An automated script would either get blocked by the login screen or extract an unverifiable partial number, which would create silent errors in our database.

### Dead End 3: Default Website Templates (The E-commerce Illusion)
I assumed the Retail/E-commerce sector (SIC 47910) would be a guaranteed source, as online stores usually display VAT numbers for customers.

> **Why it failed:** Investigating active online stores (like BUCKLE AND BAND LTD) showed they rely on default website templates (like standard Shopify Terms of Service pages). The business owners simply did not type their specific tax details into these default pages. An automated script searching the /terms page extracts valid text, but finds zero tax numbers.

### Sampling Methodology Revision: Company Size Filtering
Manual investigation of the original sample showed most failures were micro-entities with no discoverable web presence. I checked the `Accounts.AccountCategory` distribution across all 5,190,464 active companies and filtered to `FULL`, `GROUP`, or `MEDIUM` categories (102,286 companies), reasoning these file the most complete accounts and are less likely to be below the VAT registration threshold. I continued manual investigation on companies drawn from this filtered set.

### Dead End 4: Group VAT Registration (Cruden Building (Scotland) Limited)
I tested CRUDEN BUILDING (SCOTLAND) LIMITED (SC098858). Its website (cruden.co.uk) displayed a VAT number, GB268991200, but under a different company number (SC044986) than the sampled company. The HMRC checker confirmed this VAT number is valid, but registered to "CRUDEN INVESTMENTS LTD," at 16 Walker Street, Edinburgh - not to the sampled company. Cruden Building (Scotland) Limited's own filed accounts confirm Cruden Investments Limited as its intermediate parent company, and Companies House confirms Cruden Investments Ltd shares the same registered address.

> **Why it failed:** a VAT number can be valid and clearly displayed, yet belong to a different entity within the same corporate group. Had I accepted it based on name and website match alone, this would have been a false positive. This also shows that FULL-category accounts don't reliably disclose VAT numbers - Cruden Building (Scotland)'s accounts contained no VAT information anywhere in their notes, unlike Breach Farm Energy Storage Limited (also FULL-category), whose accounts disclosed VAT receivable/payable line items.
> **Lesson:** an HMRC "valid" response is not enough - the returned business name and address must also match the target company before accepting a result.

### Dead End 5: Trading Name Differs from Legal Company Name (Willmott Dixon FM Limited)
WILLMOTT DIXON FM LIMITED (07065104) does not trade under its own legal name. Its own filed accounts state it "sits above the companies that form the Fortem Brand." A search for "Willmott Dixon" initially led me to willmottdixon.co.uk, a different brand within the same wider group - not the entity that owns this specific company. I found the correct site, fortem.co.uk, and its "Legal info" page explicitly confirms: "Fortem's parent company is Willmott Dixon FM Limited (Company No: 07065104)" - matching the sampled company exactly. However, this page, like willmottdixon.co.uk's terms page, contains no VAT number anywhere.

> **Why it failed:** searching by legal company name can return the wrong brand entirely when a company operates under a trading name unrelated to its registered name. Even after finding the correct site, no VAT number was published. Unlike the Cruden case, this is a clean "not found" - not a false positive - but it shows that name-based search alone cannot reliably locate the right company website when trading names diverge from legal names.

### Dead End 6: Holdings Entity vs. Operating Subsidiary (Geo. Houlton & Sons (Holdings) Limited)
I tested GEO. HOULTON & SONS (HOLDINGS) LIMITED (300842). Its website, houlton.co.uk, listed "VAT Reg No: 167444642" under "Geo. Houlton & Sons Ltd." (Company Number 01632717) - a different company number than the sampled entity. The HMRC checker confirmed GB167444642 as valid, registered to "GEO HOULTON & SONS" at the same address shown on the website.

> **Why it failed:** the sampled "(Holdings)" entity and the operating company that publishes the VAT number share a name, brand, and address, but are distinct legal entities with different company numbers. This is a second confirmed case (after Cruden) of a valid, correctly-displayed VAT number belonging to the wrong entity within a corporate group.

### Second Sampling Revision: Excluding Non-Operating Entities
The top 10 industries within the FULL/GROUP/MEDIUM filtered set (102,286 companies) were dominated by holding companies, head offices, and non-trading entities - the source of Dead Ends 4-6. I added a second filter excluding SIC descriptions linked to non-operating entities (holding company, head office, non-trading, financial intermediation, letting and operating, financial services holding). This left 72,976 large, operating companies, from which I drew a new stratified sample of 100 (10 per industry across the top 10 industries, `random_state=36`), intended as a "happy path" sample for automated testing.

### Dead End 7: Automated Web Search Blocking (DuckDuckGo)
To automate website discovery for this new sample, I used Claude to scaffold a script querying DuckDuckGo's HTML search endpoint per company name, since I had no paid search API. A manual, isolated test worked correctly. Running it across the full 100-company sample returned 98 `no_website_found` and 2 `not_found`. I diagnosed the unexpected result by re-testing five queries in a row with Claude's help: all five returned HTTP 403 with an identical minimal response, confirming an IP-level block after roughly 100 consecutive automated queries.

Manually inspecting the 2 `not_found` results (Breach Farm Energy Storage Limited, Ranksborough Solar Limited) showed the "websites" found were `companieslist.co.uk` and `ukcompanydir.com` - aggregator directories, not the companies' own sites, since these domains were not in my ignore-list filter. This means automated discovery found zero genuine company websites out of 100, not 2 - and shows that an ignore-list approach to filtering aggregators is inherently incomplete.

### Dead End 8: Bing Search Result Extraction Failure
As an alternative to DuckDuckGo, I tested scraping Bing's search results page. The request succeeded (HTTP 200, correct page title), and my CSS selector matched 9 results. However, decoding the actual destination URL from Bing's redirect link revealed it pointed to Bing's own internal image search page (`/images/search?q=...`), not to an external company website.

> **Why it failed:** Bing's result links are wrapped in tracking redirects with an undocumented encoding, and the selector I used matched an image-carousel element rather than an organic search result. Unlike DuckDuckGo's straightforward redirect format, extracting real URLs from Bing's HTML would require further reverse-engineering with no guarantee of stability.

> **Conclusion:** two independent unauthenticated search-scraping approaches (DuckDuckGo, Bing) both failed to produce usable results at even moderate volume - one through IP blocking, the other through fragile page structure. This indicates that automated company website discovery at scale requires a paid search API rather than scraping a search engine directly.

## Part 2: Proof of Concept

Across this project I investigated a total of 109 companies: 9 manually (5 from the original unfiltered sample - SELBUD, Best House Property, KOF Technology Services, TEARAINTE, Buckle and Band; 4 from the FULL/GROUP/MEDIUM sample - Breach Farm Energy Storage, Cruden Building (Scotland), Willmott Dixon FM, Geo. Houlton & Sons (Holdings)) and 100 via the automated pipeline on the operating-companies sample.

### Results, by outcome
Of the 9 manually investigated companies, 2 produced a VAT number candidate that could be checked on HMRC's verifier (Cruden Building (Scotland) Limited and Geo. Houlton & Sons (Holdings) Limited). Both candidates were valid, real VAT numbers - and both were confirmed by HMRC to belong to a different legal entity than the one being searched. Two further companies (KOF Technology Services, TEARAINTE) showed a partially hidden VAT prefix on third-party lookup sites, which I could not verify and therefore did not count as found. One company (Breach Farm Energy Storage Limited) was confirmed VAT-registered through its own filed accounts, but no VAT number was ever located. The remaining 4 companies produced no VAT number or candidate of any kind.

Of the 100 companies run through the automated discovery pipeline, 0 produced a verifiable company website: 98 returned no website at all (due to the DuckDuckGo IP block, Dead End 7), and the 2 that did return a "website" were aggregator directories, not the sampled companies' own sites.

### False-positive rate
Every VAT number I am reporting as "found" was checked against HMRC's VAT checker, comparing both the validity of the number and the registered business name/address against the target company. Measured on the 2 candidates found, the false-positive rate was 2 out of 2 (100%). This sample of 2 is too small to generalize, but large enough to show the failure mode is real, not a one-off - it happened on two separate, unrelated corporate groups.

**True positives:** across all 109 companies investigated, 0 VAT numbers were confirmed as correctly belonging to the exact target company being searched.

### What this means
This proof of concept did not produce a working, scalable extraction method - it produced strong evidence for why the naive version of this method (search for company name, extract VAT-looking text, treat it as found) does not work, and specifically why it fails in the most dangerous way: by producing plausible, valid-looking, incorrect answers rather than visible gaps. The two confirmed false positives (Cruden, Geo Houlton) are the most important result of this project, because they show precisely the failure mode the original brief warned about - a wrong number is worse than a missing one, and it is invisible unless independently checked against both HMRC and the target company's exact registered details.

---

## Part 3: What I'd Do With Real Resources

Each dead end in Part 1 points to a specific, fixable bottleneck at scale — not a reason the problem is unsolvable, but a reason it needs infrastructure a laptop doesn't have. The relevant target is not all 40,000 suppliers, but the roughly two-thirds for whom a VAT number is currently missing.

**1. Replace search scraping with a paid search API.**
Dead Ends 7 and 8 show that unauthenticated scraping (DuckDuckGo, Bing) breaks at roughly 100 requests, through IP blocking or unstable page structure. Notably, Microsoft retired the standalone Bing Search API in August 2025, replacing it with "Grounding with Bing Search," priced around $14 per 1,000 queries as of August 2026; alternatives like Brave Search ($5/1,000) or Exa ($7/1,000) are cheaper options for this specific task. For the ~26,667 suppliers still missing a VAT number, website-discovery queries alone would cost roughly $130-$375 depending on provider, assuming one query per company succeeds — a lower-bound estimate, since at least one company in manual testing (SELBUD) required a reformulated second query after the first returned no results. Even doubled to account for retries, this remains a minor cost relative to the value of a working dataset. This is the first thing I'd fix, since it's what blocked automated testing entirely in this project.

**2. Add a step that checks whether a found VAT number actually belongs to the target company, not just a related one.** 
Dead Ends 4 and 6 (Cruden, Geo. Houlton) show the most dangerous failure mode: a valid VAT number belonging to the wrong entity in the same corporate group. At scale, this can't be caught by spot-checking — it needs a systematic name-and-address match between what HMRC returns and the target company's registered details (already available from Companies House). Where the names don't match closely, the record should be flagged for manual review rather than auto-accepted. Companies House's own data (parent/subsidiary disclosures in filed accounts, as seen with Cruden Building (Scotland) Limited and Willmott Dixon FM Limited) could also be mined to pre-flag which companies are likely to sit inside a group structure, before even attempting extraction.

**3. Accept that a meaningful share of suppliers have no findable VAT, and say so explicitly.**
Micro-entities (Dead End 1) and companies trading under a name unrelated to their legal name (Dead End 5) aren't scraping failures — they reflect companies below the VAT threshold, or brand names that don't map cleanly to a registered legal entity. No crawling budget fixes this; it needs to be reported as a hard ceiling on coverage, not chased indefinitely.

**4. What breaks first at scale:**
The name-matching step (point 2) is the most expensive and the most likely bottleneck, since it can't be fully automated without risk — a wrong automated match is worse than a missed one. Based on how often group/subsidiary structures appeared even in the small sample tested here (2 of the 9 manually investigated companies), a non-trivial share of the ~26,667 would likely need a human reviewer to resolve group/subsidiary ambiguity. That is the annotation team's actual job — not labeling VAT numbers, but confirming which entity within a group a found VAT number actually belongs to.

**5. What I'd monitor in production:**
The false-positive rate (sampled re-verification against HMRC on a rolling basis, since group structures and registrations change over time), the rate of "VAT number found but name doesn't match the target company" (a signal the group-matching step needs tuning, not that the source is bad).

---

## Debate Topics

* **UK VAT numbers are 9 digits with a checksum - what happens if you point that at HMRC's checker?**
I didn't try this myself, so this is reasoning, not a result. HMRC's checker does tell you which company a number belongs to — I saw that clearly with Cruden and Geo Houlton. But brute-forcing every valid checksum would mean submitting numbers with no idea in advance which company you'll get back. For a fixed list of 26,667 target suppliers, most hits would land on companies that aren't even on the list, so you'd still need to check every result against your supplier list by name — the same matching bottleneck as every other method here, just approached backwards. I don't think it's a good idea: it's not a targeted way to find VAT numbers for specific companies, and it would mean sending an enormous number of requests to a government service for very little useful return.

* **How would I keep the dataset current, given companies register and deregister continuously?**
Two things go out of date here. First, the Companies House file itself — it's a monthly snapshot, so it's already up to a month old the day you download it, and new companies register or close every day. Second, VAT numbers tied to corporate groups can shift over time, as seen with Cruden and Geo Houlton, where the VAT number belonged to the parent company, not the one being searched — that kind of group structure can change. To catch this, I'd re-check every VAT number against HMRC on a regular schedule, since HMRC's checker is free and doesn't require re-doing the harder part (finding the website) — just re-confirming the number is still valid and still matches the right company.

* **How would I know the dataset was wrong at scale, with nothing complete to compare against?**
The only ground truth I had access to in this project was HMRC's checker itself, used company-by-company. At scale, I would rely on the same mechanism systematically: for every VAT number in the dataset, periodically re-verify it against HMRC and check that the returned name/address still matches the target company - this is exactly the check that caught both false positives (Cruden, Geo Houlton) in this project. Beyond that, I don't have a tested method for catching errors with no reference dataset at all; this is a real limitation I ran into, not something I solved.

* **Which sources would I not be comfortable using in a product I sell, and why?**
Based on what I tested: vat-search.co.uk and vatverifier.com (Dead End 2) - both displayed partially hidden VAT numbers and pushed toward account creation, and I was unable to verify whether the hidden digits were even real. I would not use either in a product, since I have no evidence their data is accurate and the presentation pattern (blur + signup prompt) is consistent with lead generation rather than a verified data source. I would also be cautious about DuckDuckGo/Bing HTML scraping (Dead Ends 7-8) - not for accuracy reasons, but because both are unauthenticated use of a search engine's website rather than a sanctioned API, which is not a stable or appropriate foundation for a paid product.

---

## AI Tool Usage

I used Claude to scaffold the boilerplate Python code for data processing (chunked CSV reading, pandas filtering) and web scraping (HTML parsing, regex extraction, search engine querying). Claude also helped debug issues as they came up during testing (e.g. diagnosing the DuckDuckGo IP block, decoding Bing's redirect links). The sampling strategy, the interpretation of each dead end, which companies to investigate manually, and all conclusions about what the results mean were my own decisions, verified against primary sources (Companies House, HMRC) at each step.
