---
name: patent-dd
description: "Patent portfolio due diligence analysis. Use this skill whenever the user wants to evaluate, assess, or conduct due diligence on a patent, patent portfolio, or IP assets for acquisition, licensing, or investment. Trigger on: 'due diligence', 'DD', 'patent analysis', 'IP audit', 'patent valuation', 'portfolio review', 'freedom to operate', 'FTO', 'patent acquisition', 'evaluate this patent', 'assess IP risk', or any request to review patent filings, claims, prior art, chain of title, or prosecution history. Also trigger when the user uploads patent documents (PDF of granted patents, office actions, assignment records, patent overviews in .xlsx) and wants them analyzed."
---

# Patent Portfolio Due Diligence Skill

This skill guides a thorough, structured due diligence analysis of patent portfolios for acquisition or licensing. It encodes best practices learned from real DD engagements.

## Workflow Overview

A patent DD proceeds in phases. Do not skip phases — each one feeds the next.

### Phase 1: Intake and Inventory

1. **Identify all assets.** Read any seller-provided patent overview (often an .xlsx file). Extract: internal reference numbers, short names, application numbers, publication numbers, filing dates, jurisdictions, applicant names, inventor names, status, and next steps.
2. **Build a family map.** Group assets by patent family (same priority chain). Identify the priority filing, Paris Convention deadlines, and all family members across jurisdictions.
3. **Flag missing information immediately:** unpublished applications (no public data available), missing jurisdictions (e.g., EP filed but no US, or vice versa), upcoming deadlines (priority deadlines, response deadlines, maintenance fees).

### Phase 2: Public Register Verification

For each family member, verify against the public register of the relevant patent office:

| Jurisdiction | Register | What to Check |
|---|---|---|
| US | USPTO Assignment Center, Patent Center, Global Dossier | Applicant, assignee, assignment chain (Reel/Frame), prosecution history, published claims, office actions |
| EP | Espacenet, European Patent Register | Applicant, designated states, search report (EESR), publication status |
| DE | DPMA Register (DPMAregister) | Owner (Inhaber), registration date, transfer status |
| AT | Austrian Patent Office | Owner, registration status |
| PCT | WIPO PatentScope | National phase entries, designated states |

**Critical checks:**
- **Chain of title:** Verify that the entity selling the portfolio is actually the registered owner in each jurisdiction. Unregistered transfers are a major red flag (especially in DE where unregistered transfers are not enforceable against third parties under §30(3) PatG).
- **Inventor assignments:** For US applications, verify that inventor-to-company assignments are recorded in the USPTO Assignment Center. Missing assignments create chain-of-title vulnerabilities.
- **Maintenance fees:** Check that all maintenance/renewal fees are current.

### Phase 3: Prior Art and Grace Period Analysis

This is often the most critical phase. For each family:

1. **Identify all pre-filing publications by the inventors.** Academic papers (arXiv, conference proceedings), GitHub repositories, blog posts, press releases, product launches.
2. **Build a timeline:** publication dates vs. filing dates for each jurisdiction.
3. **Apply grace period rules by jurisdiction:**

| Jurisdiction | Grace Period | Scope |
|---|---|---|
| US | 12 months (35 U.S.C. § 102(b)(1)(A)) | Covers inventor self-disclosures. Broad protection. |
| DE (Gebrauchsmuster) | 6 months (§ 3(1) GebrMG) | Covers inventor's own publications. Only for utility models, NOT patents. |
| AT (Gebrauchsmuster) | 6 months (§ 4a GMG) | Very narrow — only "obvious abuse" against applicant. Voluntary self-publication is generally NOT covered. |
| EP (EPC) | None (Art. 54 EPC) | No grace period. Any pre-filing publication destroys novelty, even by inventors. |
| PCT | Depends on national phase | Follow the rules of the designated state. |

4. **Grace period assessment is binary:** if the deadline is respected, the risk is Low. If not respected, the risk is High. The margin (3 days vs. 3 months) is irrelevant — a deadline is either met or it isn't.

### Phase 4: Claim Analysis

For each family's main independent claim:

1. **Extract claim features.** Break independent claim 1 into discrete features/elements.
2. **Build a claim chart vs. prior art.** For each feature, identify where it appears in the prior publication (paper, code repo, etc.). Use exact page/section references.
3. **Assess novelty risk:** if all features are disclosed in a single prior art reference, there is a novelty problem. If features are split across multiple references, it's an obviousness/inventive step question.
4. **Check dependent claims:** identify which dependent claims add features NOT disclosed in the prior art — these represent the "fallback position" if independent claims fail.

### Phase 5: Open Source and Licensing Analysis

If the technology involves software, check for publicly available source code:

1. **Identify all relevant repositories** (company repos, university repos, personal repos of inventors).
2. **Check the license of each repo.** The license type fundamentally affects patent enforceability:

| License | Commercial Use | Patent Impact |
|---|---|---|
| MIT | Yes — fully permissive | **Danger:** implied patent license risk. If patent holder publishes code under MIT, a defendant may argue implicit authorization to use the patented method (De Forest Radio Tel. Co. v. United States). |
| Apache 2.0 | Yes — with explicit patent grant | **Explicit patent grant** in Section 3. Using Apache-licensed code comes with a patent license from contributors. |
| GPL/AGPL | Yes — with copyleft | Patent grant implied by distribution rights. Copyleft may deter commercial use but doesn't prevent it. |
| CC-BY-NC | Non-commercial only | Lower risk — commercial competitors cannot use the code. Patent adds protection layer. |
| Proprietary / ENPL / MNPL | Requires commercial license | **Best for patent enforcement.** License and patent work in the same direction. |

3. **Check repo ownership.** A repo owned by the patent applicant's company is different from a repo owned by a university where the inventors work. University repos may have been published without the company's control, complicating the implied license analysis.
4. **Assess design-around facilitation.** Public code (regardless of license) makes it easier for competitors to understand the implementation and engineer around the claims. Note the specific architectural elements exposed.
5. **If no open source exists, check the company's Terms of Use / Terms of Service.** Look for clauses prohibiting reverse engineering, decompilation, or source code derivation (typically in "Restrictions" or "Intellectual Property" sections). Such contractual protections strengthen patent enforceability by adding a contractual layer on top of patent rights — competitors cannot legally study the implementation to design around claims. Include the URL of the ToS and the specific section number in the memo. Note: ToS restrictions only bind users who accept them; they do not prevent independent development or clean-room reverse engineering by parties who never used the platform.

### Phase 6: Commercial Relevance

1. **Map patents to products.** For each patent family, identify which commercial product implements the claimed technology. If the company has a "model zoo" or product catalog, check whether the patented technology appears in it.
2. **Build a product-to-patent claim chart** showing how each claim element maps to product features.
3. **Identify gaps:** patents without commercial implementation (reduces valuation), and products without patent coverage (reduces exclusivity value). A patent covering technology absent from the product catalog may still have value if it's used as a backend component of another product — ask the seller to clarify.
4. **Assess revenue linkage:** can the seller demonstrate that specific revenue streams depend on the patented technology? No proven revenue = significant discount.

### Phase 6b: Prosecution Cost Estimation

When pending applications face office action rejections, estimate realistic prosecution costs:

| Action | Typical Cost Range |
|---|---|
| Counsel-drafted response to Office Action | $5,000 – $15,000 |
| RCE (Request for Continued Examination) filing | $2,000 – $4,000 (filing fee) + counsel time |
| PTAB appeal (ex parte) | $10,000 – $30,000 |
| Continuation with revised claims | $8,000 – $18,000 |
| Full prosecution through appeal | $15,000 – $30,000 total |

**Important:** Always cross-reference cost estimates against the narrative sections of the memo. Scenario tables in recap documents must be consistent with the cost ranges stated in the body of the memo. Err on the lower side — inflated prosecution costs undermine credibility with the client.

### Phase 7: Risk Register and Valuation Impact

Build a comprehensive risk table. For each risk, assess:
- **Severity** (High / Medium / Low)
- **Impact on valuation** (what discount does this risk justify?)
- **Mitigation** (can the seller fix this before closing?)

**Common risk categories and negotiation levers:**

1. **Invalid titles** (prior art destroys novelty) → direct valuation discount
2. **Unregistered transfers** → condition closing on registration, or escrow
3. **Missing inventor assignments** → condition closing on filing
4. **Unexamined rights** (utility models, pending applications) → discount for uncertainty
5. **Open source exposure** (MIT-licensed code) → discount for enforceability risk
6. **Unfiled applications** (planned but not yet deposited) → do not pay for intentions; condition on actual filing
7. **Upcoming deadlines** (priority, office action response) → identify who bears cost and risk
8. **Key inventor dependency** → require cooperation clauses in acquisition agreement
9. **Limited geographic coverage** → discount vs. global portfolio
10. **§101/Alice risk** (US software patents) → discount for litigation vulnerability

### Phase 8: Deliverables

#### 8.1 Full DD Memo (HTML)

The DD memo should be structured as a single HTML document with:

1. Executive summary (1 page max)
2. Portfolio overview table (all family members, all jurisdictions)
3. Chain of title verification
4. Family-by-family detailed analysis (each family gets: technology description, family members table, independent claims, claim chart vs. prior art, grace period analysis, GitHub/open source analysis)
5. Open source licensing vs. patent enforceability analysis (always include, even if conclusion is "no exposure")
6. Commercial relevance and product-to-patent mapping
7. Grace period summary table (all families × all jurisdictions)
8. Risk register with severity ratings
9. Recommendations (prioritized, with immediate vs. medium-term items)
10. Suggested closing conditions
11. Documents to obtain (with purpose for each)

**Formatting:** Use CSS classes for risk coloring (risk-high, risk-medium, risk-low), grace period status (grace-ok, grace-risk, grace-warn), and structured tables throughout. Include Google Patents links for all published applications.

**Source attribution:** When information comes from the seller (e.g., from a patent overview spreadsheet), always cite the exact filename and the specific column/row where the information was found. Mark seller-provided data as "unverified seller declaration" to distinguish from independently verified data.

#### 8.2 Recap Document (DOCX + PDF)

In addition to the full memo, produce a recap assessment document (landscape, suitable for Google Docs import) with:

1. **Family Member Assessment table** — one row per family member, columns: Family, Member, Jurisdiction, Validity, Scope, Interest, Price Leverage. Color-code rows with only two colors: green (valid/positive) and red (invalid/problematic). Do not use a third color — simplicity is critical for readability.
2. **Open Source / License Impact table** (if applicable) — one row per repository, columns: Family, Repository, License, Risk, Impact.
3. **Prosecution Outlook table** (if pending applications exist) — scenario-based with probability, timeline, estimated cost, and impact on price.
4. **Negotiation Levers table** — numbered list of arguments to reduce the acquisition price.
5. **Bottom Line** — 2-3 paragraphs summarizing the portfolio's real value.

**Interest column guidance:** If the acquirer is not yet identified or the strategic fit is uncertain, use "OK TBC" (to be confirmed) rather than "Strong" or "High". Reserve "Strong" for cases where the acquirer's specific business model clearly benefits from the technology. Don't speculate on strategic interest.

**Technical implementation:** Use `docx` npm library (docx-js) for DOCX generation, then convert to PDF via LibreOffice (`libreoffice --headless --convert-to pdf`).

## Key Principles

- **Verify everything against public registers.** Never rely solely on the seller's representations.
- **Grace periods are binary.** Respected = Low risk. Not respected = High risk. Margin size is irrelevant.
- **Open source licenses matter for patent enforceability.** MIT ≠ CC-BY-NC ≠ proprietary. Always check.
- **Unfiled applications have zero value.** A "planned" filing is a statement of intent, not an asset.
- **Utility models are not examined.** They may be "granted" but their validity has never been tested.
- **The acquirer pays for what exists, not what's promised.** Price negotiations should reflect the actual state of the portfolio, not the seller's roadmap.
- **Cite your sources.** Every factual claim in the memo must be traceable to a public register, a specific document, or marked as an unverified seller declaration.
- **Keep cost estimates conservative.** Inflated numbers erode trust. Use the lower end of market ranges and cross-check against the memo narrative.
- **Two colors, not three.** In recap tables, use green (valid/positive) and red (invalid/problematic) only. A third color introduces ambiguity that clients don't need.
