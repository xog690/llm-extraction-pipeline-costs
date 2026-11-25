# LLM data extraction: why pipelines break at the fetching step, what accuracy numbers really mean, and which ScraperAPI plan fits (full plan comparison)

Most LLM data extraction projects that stall do so before the model sees a single token. The prompt is usually fine. What fails is the step before it: getting clean text out of pages that change, load with JavaScript, or sit behind bot protection. The second thing that fails is verification, because nobody budgets time for checking the model's output against the source.

This article separates the three jobs in an extraction pipeline (fetching, extracting, validating), gives verified numbers for each, and lays out ScraperAPI's current plans and credit costs so you can size the fetching side before you commit. Here is the short version if you already know you want a scraping API.

> **Most readers should start on the free plan.** ScraperAPI's free tier gives you 1,000 API credits a month with 5 concurrent connections, plus 5,000 credits for the first 7 days after signup, which is enough to run a real target list before paying anything. Move to a paid tier once you know your actual per-URL credit cost. 👉 [Start free and check current plan rates](https://www.scraperapi.com/pricing/?fp_ref=coupons)

## Every ScraperAPI plan side by side

ScraperAPI bills on API credits, not bandwidth and not seats. Credits reset at each renewal and do not roll over, so the plan you pick should match your monthly request volume rather than your peak month.

| Plan | Monthly API credits | Concurrent threads | Overage and billing | Link |
| --- | --- | --- | --- | --- |
| Free | 1,000 (5,000 in first 7 days) | 5 | Pauses at limit | [Free plan details](https://www.scraperapi.com/pricing/?fp_ref=coupons) |
| Hobby | 100,000 | 20 | Auto-upgrade opt-in | [Hobby plan](https://www.scraperapi.com/pricing/?fp_ref=coupons) |
| Startup | 1,000,000 | 50 | Auto-upgrade opt-in | [Startup plan](https://www.scraperapi.com/pricing/?fp_ref=coupons) |
| Business | 3,000,000 | 100 | Auto-upgrade opt-in | [Business plan](https://www.scraperapi.com/pricing/?fp_ref=coupons) |
| Scaling | 5,000,000 | 200 | Auto-upgrade or pay-as-you-go | [Scaling plan](https://www.scraperapi.com/pricing/?fp_ref=coupons) |
| Professional | 10,500,000 | 300 | Auto-upgrade or pay-as-you-go | [Professional plan](https://www.scraperapi.com/pricing/?fp_ref=coupons) |
| Advanced | 21,500,000 | 500 | Pay-as-you-go only | [Advanced plan](https://www.scraperapi.com/pricing/?fp_ref=coupons) |
| Enterprise | 22,000,000+ | 500+ | Custom quote | [Enterprise and custom pricing](https://www.scraperapi.com/pricing/?fp_ref=coupons) |

Two settings change between tiers and are easy to miss in the headline credit number:

- **Geotargeting.** Hobby and Startup are limited to US and EU proxies. Country-level geotargeting starts at Business.
- **Analytics history.** Hobby and Startup keep 30 days. Business and above keep it unlimited.

All plans bill monthly by default. Annual billing takes **10% off** and delivers the full year's credits upfront, which you can spend on any timeline. Enterprise pricing is quoted per workload, and ultra-premium requests carry their own minimum rate even on custom contracts.

## What a request actually costs, which is where budgets go wrong

"A credit per page" is the number people plan around and the number that makes budgets wrong. ScraperAPI prices by target domain and by the features you switch on.

| Request type | Credits per request |
| --- | --- |
| Standard page | 1 |
| Amazon, Walmart, eBay | 5 |
| Google and Bing (including subdomains) | 25 |
| LinkedIn | 30 |
| Anti-bot bypass (Cloudflare, DataDome, PerimeterX, Akamai) | +10 on base cost |
| JavaScript rendering (`render=true`) | 10 |
| Premium proxy (`premium=true`) | 10 |
| Screenshot (`screenshot=true`) | 10 |
| Premium proxy plus JS rendering | 25 |
| Ultra premium (`ultra_premium=true`) | 30 |
| Ultra premium plus JS rendering | 75 |

Geotargeting, sessions, device type, output format and autoparse add nothing. That means 100,000 credits is 100,000 plain pages, but only about 4,000 Google SERP requests, and fewer still once you combine proxy and rendering options. A price-monitoring job on bot-protected retail sites can burn 20 credits or more per page depending on which levers you pull, so the plan tier that looks generous against a per-page estimate can be tight against the real number.

Every response returns its own cost in the `sa-credit-cost` header, and the dashboard has a Domain Cost Estimator that prices a single URL before you scrape it. Set `max_cost` per request and no single call can exceed your ceiling. Run your ten hardest target URLs through the estimator first, multiply by your daily volume, and pick the tier from that. 👉 [Compare plan credit allowances before you size a budget](https://www.scraperapi.com/pricing/?fp_ref=coupons)

One billing constraint worth knowing before you choose: pay-as-you-go overage only exists from Scaling upward. On Hobby, Startup and Business, running out of credits pauses your scraping unless auto-upgrade is switched on, and you get a 429 rather than a charge. If your workload is spiky, that difference decides the plan more than the credit count does. 👉 [See which plans include pay-as-you-go](https://www.scraperapi.com/pricing/?fp_ref=coupons)

## The three jobs in an extraction pipeline

### Job 1: fetch the text

The tooling here has known failure modes, and they map to page types rather than to skill level.

| Approach | Good for | Fails on |
| --- | --- | --- |
| XPath / CSS selectors | Fast, cheap, deterministic on stable pages | Layout changes, JavaScript-rendered content |
| Regular expressions | Known patterns, precision | Hand configuration, transfers poorly between sites |
| Headless browsers (Selenium, Playwright) | Dynamic pages, SPAs, infinite scroll | Cost, speed, maintenance burden |
| Template methods | Large sets of near-identical pages | Structural variation |
| Fine-tuned IE models | High accuracy within one domain | Requires markup per entity type and per source |

The honest read of that table: a self-built Selenium or Playwright cluster is a reasonable choice for one site you control or a handful of sources that rarely change. Running it across dozens of sites with different bot protections means your team owns the proxy rotation, the retry logic, the headless browser upgrades and the breakage when a target redesigns its layout. A metered scraping API moves that cost from engineering hours to a per-request line item, and the line item is visible before you spend it.

### Job 2: turn the text into variables

Prompt design in production extraction is layered rather than uniform. The lymphedema extraction pipeline built on Stanford's SecureGPT instance mixes strategies by variable: zero-shot for affected-limb fields, a mix of zero-shot and few-shot for the rest, few-shot for surgery and compression bandaging, and added clarifying language for pneumatic pumps, manual edema therapy and trauma. Swelling improvement became a three-way categorical field, and stage allowed values 0 through 3 with 9 reserved for uncertain. That last detail matters more than it sounds, because giving the model a legal way to say "unsure" is the only thing that makes an accuracy score interpretable later.

On the engineering side, the schema is the contract. DELM, an open-source Python toolkit for extraction pipelines, runs each call through Instructor with a Pydantic schema, adds exponential backoff retries, and caches results against a key built from the rendered prompt, schema, model and generation parameters. Each run writes out a config file recording data provenance, prompt text, temperature and evaluation metrics, so a rerun is reproducible. It also tracks input and output tokens as separate cost lines, which is the correct way to measure extraction spend: a 3,900-token document with a 63-token output is not priced like a chatbot conversation.

### Job 3: prove the output matches the source

The verification method that costs least and catches most is temperature variance. One published pipeline issues three queries per document at different temperature settings, then compares the resulting texts by embedding similarity to flag fields where the model's answer drifts. It layers formal checks on top (format precision, required fields, type validity), and routes mismatches to a human or a rerun. The similarity implementation uses `bert-base-multilingual-uncased` with mean pooling over the last hidden state excluding CLS and SEP, normalized and compared by cosine; per-category thresholds sit between 85% and 89%.

That is a usable template, not a guarantee. It catches answers that wobble under resampling. It does not catch a confidently wrong answer that is stable across all three temperatures, which is why a sampled human review against the original source stays in the process.

## How to read extraction accuracy claims

Two clinical systems give concrete numbers, and both come with conditions that get stripped out when people quote them.

The lymphedema pipeline reports variable accuracy against the original clinical notes above 90% across the board, measured on random subsets of 25 and 50 patients. It gets there by forcing multiple passes over the same information: note-level summarization, then patient-level summarization, then variable extraction. More tokens per document, better fidelity to the source. The authors are explicit that they did not run an independent human extraction to compare against, and that the study goal was to characterize LLM mistakes rather than to prove equivalence to human chart review.

CIDER, a locally deployed system for Hungarian-language histopathology reports, processed 2,073 patient reports with a median input of **3,926 tokens**, a median output of **63 tokens**, and a mean of **14.0 seconds per report**, 2,948 seconds for the full validation cohort. Across that run there were no invalid JSON outputs, no formatting errors and no processing failures. Its primary metric was exact agreement with a manually built database, supplemented by weighted F1 and Cohen's kappa.

Both are document sets, not websites. The CIDER reports were native-text PDFs, one tumor entity per file, no OCR, no navigation menus, no cookie banners. The clean-JSON result says a lot about schema-constrained decoding on tidy input and very little about what happens when you feed a model the output of a page that half-rendered. Treat 90% as evidence that the approach works on documents with a stable structure, and plan on measuring your own accuracy on your own pages.

## Two failure modes that look like model errors but are not

**Terminology granularity.** CIDER normalizes conservatively: if the model outputs "pancreatic ductal adenocarcinoma" and the manual annotation says "adenocarcinoma", the field counts as wrong even though the model is more specific. That is a defensible choice for a registry, and it is a trap for anyone scoring a pipeline without writing the granularity rule into the schema first. Decide whether subtypes collapse upward before you measure, or you will spend a week chasing mismatches that are definitional.

**Filling in missing fields.** In the lymphedema work, only 13 of 100 reviewed notes stated disease stage explicitly. The model added a stage in 9 of 100 note summaries, and all nine were correct against the source. At the patient-summary level it added stage in 7 of 50 cases at 100% accuracy. At the final extraction step, stage was added in 11 instances and **3 of those 11 were wrong**. The same behavior that helped early in the pipeline introduced errors at the end. If a field can be inferred rather than read, validate it at the step where it first appears, not only at the output.

## Where the money actually goes

Two separate bills, and they scale on different axes.

The fetching bill scales with pages and difficulty. On ScraperAPI it is credits per request, priced by domain and by the features you enable, with no rollover and pay-as-you-go only from the Scaling tier up.

The model bill scales with tokens and with where the model runs. CIDER runs a 32B-parameter open-weight vision-language model (Qwen3-VL-32B-Instruct-FP8) on local NVIDIA A40 GPUs. The A40 has no native FP8 acceleration, but dropping to FP8 reduced memory enough to fit a 32B model inside the institution's existing hardware, and the authors chose the model for deployability and multilingual coverage rather than benchmark leadership. The trade is explicit: you accept a smaller model and your own ops burden in exchange for keeping patient data on-premises. At 14 seconds per document, throughput is a hardware question you can plan for.

Running against a frontier API reverses the trade. The Stanford pipeline used a PHI-compliant GPT-5 endpoint through an institutional API that was free to researchers at the time, and the authors note that pricing has since fluctuated with institutional contracts. If you go that route, the cost line moves with your provider agreement, and per-document spend becomes something you re-check rather than lock in.

Caching sits between the two. Deterministic caching keyed on prompt, schema, model and parameters means re-running an evaluation does not re-bill the tokens, and iterating on your scoring code costs nothing at the API.

## Picking a setup for your case

- **A few hundred pages, once.** Skip the pipeline. A script plus a manual spot check of 30 records beats anything you'd build, and the error rate you'd be measuring is smaller than the time cost of measuring it.
- **Continuous monitoring across many sites.** Fix fetching first. Self-built headless browsers across a changing target list is the most expensive part of the stack to maintain, and it is the part a metered API replaces. Start on the free plan, price your real URLs with the Domain Cost Estimator, then choose between a fixed tier and pay-as-you-go on Scaling.
- **Regulated or confidential data.** Keep inference local, as CIDER did, and keep the fetching layer under your own control. Expect to spend your effort on GPU capacity and schema design rather than on prompt wording.
- **Spiky volume.** The pay-as-you-go threshold matters more than the credit count. If you cannot predict monthly page counts, Hobby through Business will pause you at the cap unless auto-upgrade is enabled.

Before you order, confirm two things: the per-URL credit cost for your ten hardest targets, and whether the tier you picked includes the overage behavior you want. ScraperAPI offers a **7-day, no-questions-asked refund** if the credit math turns out wrong, and the free plan is enough to discover that before you pay.

👉 [Check current ScraperAPI plans and the 7-day refund terms](https://www.scraperapi.com/pricing/?fp_ref=coupons)
