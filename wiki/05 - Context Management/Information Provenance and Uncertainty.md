---
tags:
  - domain-5
  - task-5-6
  - provenance
  - source-attribution
  - uncertainty
  - conflicting-data
  - temporal-metadata
domain: "5 - Context Management & Reliability"
task: "5.6"
---

# Information Provenance and Uncertainty

> **Task 5.6** — Preserve source attribution through summarization and synthesis, handle conflicting data with annotations, enforce temporal metadata, and render findings with appropriate uncertainty markers.

---

## Source Attribution Lost During Summarization

When a research agent summarizes findings from multiple sources, the most common casualty is **source attribution**. A finding like *"The global AI market is projected to reach $407 billion by 2027"* is useless without knowing where that number came from — because a different credible source might say $390 billion, and the consumer needs to evaluate the sources, not just the numbers.

Summarization naturally strips provenance because it optimizes for brevity. The fix is structural: maintain claim-source mappings as a data structure that passes through every synthesis step, not as inline citations that get condensed away.

---

## Structured Claim-Source Mappings

Instead of free-text summaries with parenthetical citations, maintain a structured mapping between claims and their sources. This mapping survives summarization because it is treated as data (like case facts in context management), not as prose.

Each claim-source entry should include:
- **Claim**: The specific factual assertion
- **Source**: Document name, URL, or identifier
- **Excerpt**: The exact quote or data point from the source
- **Date**: When the source was published or the data was collected
- **Confidence**: How directly the source supports the claim

---

## Conflicting Statistics

When two credible sources provide different numbers for the same metric, the correct approach is to **annotate the conflict with source attribution** rather than:

- Silently picking one number
- Averaging them
- Picking the more recent one without noting the discrepancy

Conflicts are information. They tell the consumer that the metric is contested, that measurement methodologies differ, or that the data was collected at different times.

---

## Temporal Data Requirements

Statistics and data points without publication or collection dates are ambiguous and potentially misleading. Consider: *"Unemployment is 4.2%."* Is that from last month, last year, or five years ago? Two sources quoting different unemployment figures might not actually conflict — they might simply reflect different time periods.

Every data point should carry temporal metadata:
- **Publication date**: When the source was published
- **Data collection date**: When the underlying data was gathered (often different from publication)
- **Reference period**: What time period the data describes (e.g., "Q1 2026", "fiscal year 2025")

---

## Rendering Different Content Types Appropriately

Not all findings should be rendered the same way. The format should match the content type:

| Content Type | Appropriate Format | Why |
|-------------|-------------------|-----|
| Financial data | Tables with aligned columns | Enables comparison across periods/entities |
| News/narrative | Prose paragraphs | Context and sequence matter |
| Technical specifications | Bulleted lists | Scannable, parallel structure |
| Conflicting claims | Side-by-side with source labels | Makes disagreement visible |
| Temporal trends | Ordered lists or tables by date | Shows progression |

---

## Examples

### Example 1 — Claim-Source Mapping Structure

```python
# Structured claim-source mappings that survive summarization.
# These are maintained as data, not inline prose citations.

research_findings = {
    "topic": "Global AI Market Size Projections",
    "claims": [
        {
            "claim_id": "C1",
            "claim": "The global AI market is projected to reach $407 billion by 2027.",
            "source": {
                "name": "IDC Worldwide AI Spending Guide",
                "url": "https://www.idc.com/getdoc.jsp?containerId=prUS51243526",
                "document_type": "market_research_report",
                "publication_date": "2026-03-15",
                "data_collection_date": "2026-Q1",
                "credibility": "high — primary research firm",
            },
            "excerpt": "Worldwide spending on AI solutions is forecast to reach "
                       "$407B in 2027, growing at a CAGR of 26.4% from 2023.",
            "confidence": "direct — explicit statement in source",
        },
        {
            "claim_id": "C2",
            "claim": "AI market growth is driven primarily by generative AI adoption "
                     "in enterprise applications.",
            "source": {
                "name": "McKinsey Global Survey on AI",
                "url": "https://www.mckinsey.com/capabilities/quantumblack/our-insights/ai-survey-2026",
                "document_type": "survey_report",
                "publication_date": "2026-02-28",
                "data_collection_date": "2025-Q4",
                "credibility": "high — large-sample enterprise survey",
            },
            "excerpt": "72% of respondents report deploying generative AI in at least "
                       "one business function, up from 65% in 2024.",
            "confidence": "inferred — survey supports the claim but does not directly "
                          "attribute market growth to generative AI spending",
        },
    ],
    "synthesis_note": "Both sources support strong AI market growth. C1 provides "
                      "the specific projection; C2 provides the adoption driver. "
                      "Note: C2's confidence is 'inferred' — the survey measures "
                      "adoption, not spending.",
}

# When this is summarized for a report, the claim-source mapping is preserved
# as structured data, even if the prose summary is condensed.
# The consumer can always drill into the mapping to verify any claim.
```

---

### Example 2 — Conflicting Statistics from Two Credible Sources

```python
# Two credible sources disagree on a key metric.
# The correct approach: annotate the conflict, don't pick a winner.

conflicting_claims = {
    "metric": "Cloud infrastructure market share — AWS",
    "claims": [
        {
            "source": "Synergy Research Group Q1 2026 Report",
            "publication_date": "2026-04-10",
            "data_period": "Q1 2026",
            "value": "32%",
            "methodology": "Revenue-based market share across IaaS, PaaS, hosted private cloud",
        },
        {
            "source": "Canalys Cloud Market Report Q1 2026",
            "publication_date": "2026-04-12",
            "data_period": "Q1 2026",
            "value": "29%",
            "methodology": "Revenue-based market share for IaaS and PaaS only (excludes hosted private cloud)",
        },
    ],
    "conflict_analysis": {
        "likely_explanation": "Methodological difference — Synergy includes hosted "
                             "private cloud in their definition of cloud infrastructure, "
                             "while Canalys excludes it. AWS has a larger share of the "
                             "hosted private cloud segment.",
        "resolution": "Both figures may be correct under their respective definitions. "
                      "Report both with methodology notes.",
    },
}

# CORRECT output for a report:
report_section = """
### AWS Cloud Market Share (Q1 2026)

AWS's cloud infrastructure market share in Q1 2026 is reported at **32%** by
Synergy Research Group and **29%** by Canalys. The discrepancy reflects differing
scope definitions:

| Source | Share | Scope | Published |
|--------|-------|-------|-----------|
| Synergy Research | 32% | IaaS + PaaS + hosted private cloud | 2026-04-10 |
| Canalys | 29% | IaaS + PaaS only | 2026-04-12 |

Both figures cover Q1 2026 revenue. The 3-percentage-point gap is attributable
to Synergy's inclusion of hosted private cloud, where AWS holds a larger share.
"""

# WRONG approaches:
# 1. "AWS has 32% market share." (silently picked one)
# 2. "AWS has approximately 30-32% market share." (vague averaging)
# 3. "AWS has 29% market share (Canalys, 2026)." (picked the more recent one)
```

---

### Example 3 — Temporal Data with Publication Dates Preventing False Contradictions

```python
# Without temporal metadata, these two data points appear to contradict:
# Source A: "US unemployment rate is 4.2%"
# Source B: "US unemployment rate is 3.7%"
# Are they disagreeing? No — they're from different time periods.

temporal_claims = [
    {
        "claim": "US unemployment rate is 4.2%",
        "source": "Bureau of Labor Statistics — Employment Situation Report",
        "publication_date": "2026-04-04",
        "data_collection_date": "2026-03-12",  # Survey reference week
        "reference_period": "March 2026",
        "value": 4.2,
        "unit": "percent",
    },
    {
        "claim": "US unemployment rate is 3.7%",
        "source": "Bureau of Labor Statistics — Employment Situation Report",
        "publication_date": "2025-10-03",
        "data_collection_date": "2025-09-12",
        "reference_period": "September 2025",
        "value": 3.7,
        "unit": "percent",
    },
]

# With temporal metadata, the synthesis is correct:
synthesis = """
### US Unemployment Rate — Trend

| Period | Rate | Source | Published |
|--------|------|--------|-----------|
| September 2025 | 3.7% | BLS Employment Situation | 2025-10-03 |
| March 2026 | 4.2% | BLS Employment Situation | 2026-04-04 |

The unemployment rate increased from 3.7% to 4.2% over the six-month period
from September 2025 to March 2026, based on Bureau of Labor Statistics data.
"""

# Without temporal metadata, a naive synthesis might say:
# "Sources disagree on the unemployment rate: one says 4.2%, another says 3.7%."
# This is misleading — there is no disagreement, only a temporal trend.

# Subagent instruction for enforcing temporal metadata:
RESEARCH_SUBAGENT_INSTRUCTION = """
For EVERY statistic or data point you extract, you MUST include:
1. publication_date: When the source document was published
2. data_collection_date: When the underlying data was gathered (if stated)
3. reference_period: What time period the data describes (e.g., "Q1 2026")

If any of these dates cannot be determined from the source, explicitly note
"date unknown" rather than omitting the field. Never present undated statistics
as current facts.
"""
```

---

### Example 4 — Report Sections Distinguishing Well-Established from Contested Findings

```python
# A multi-agent research system produces a report that explicitly labels
# the certainty level of each section.

def render_research_report(findings: list[dict]) -> str:
    """Render findings grouped by certainty level with appropriate formatting."""

    well_established = [f for f in findings if f["certainty"] == "well_established"]
    contested = [f for f in findings if f["certainty"] == "contested"]
    uncertain = [f for f in findings if f["certainty"] == "uncertain"]

    sections = []

    # Well-established findings: confident prose, tables for data
    if well_established:
        sections.append("## Well-Established Findings\n")
        sections.append("*Multiple credible sources agree on these points.*\n")
        for f in well_established:
            sections.append(f"### {f['topic']}")
            sections.append(f"{f['summary']}")
            # Financial data rendered as table
            if f.get("data_type") == "financial":
                sections.append(render_financial_table(f["data"]))
            # Narrative rendered as prose
            elif f.get("data_type") == "narrative":
                sections.append(f["detailed_narrative"])
            sections.append(f"*Sources: {', '.join(s['name'] for s in f['sources'])}*\n")

    # Contested findings: side-by-side source comparison
    if contested:
        sections.append("## Contested Findings\n")
        sections.append("*Sources disagree on these points. Both perspectives "
                       "are presented with attribution.*\n")
        for f in contested:
            sections.append(f"### {f['topic']}")
            sections.append(f"{f['conflict_summary']}")
            # Render conflicting positions side by side
            for position in f["positions"]:
                sections.append(
                    f"- **{position['source']['name']}** ({position['source']['publication_date']}): "
                    f"{position['claim']}"
                )
            if f.get("conflict_analysis"):
                sections.append(f"\n*Analysis*: {f['conflict_analysis']}")
            sections.append("")

    # Uncertain findings: hedged language, explicit gaps
    if uncertain:
        sections.append("## Preliminary / Uncertain Findings\n")
        sections.append("*Limited or low-confidence sources. Treat as directional only.*\n")
        for f in uncertain:
            sections.append(f"### {f['topic']}")
            sections.append(f"{f['summary']}")
            sections.append(f"*Confidence: {f['confidence_note']}*")
            sections.append(f"*Gaps: {f['coverage_gap']}*\n")

    return "\n".join(sections)

# Example output structure:
#
# ## Well-Established Findings
# *Multiple credible sources agree on these points.*
#
# ### Q1 2026 Revenue
# Revenue grew 12% YoY to $4.2B.
# | Segment     | Revenue | YoY Change |
# |-------------|---------|------------|
# | Cloud       | $1.8B   | +23%       |
# | On-Premise  | $1.4B   | +3%        |
# | Services    | $1.0B   | +8%        |
# *Sources: 10-K filing 2026-03-01, Earnings call transcript 2026-02-28*
#
# ## Contested Findings
# *Sources disagree on these points.*
#
# ### Market Share
# - **Synergy Research** (2026-04-10): 32% cloud infrastructure share
# - **Canalys** (2026-04-12): 29% cloud infrastructure share
#
# *Analysis*: Methodological difference in scope definition...
#
# ## Preliminary / Uncertain Findings
# ### Expansion into Healthcare AI
# Unconfirmed reports suggest a strategic partnership...
# *Confidence: Single anonymous source, not corroborated*
# *Gaps: No official announcement; competitor analysis incomplete*
```

This rendering approach makes uncertainty visible to the consumer rather than presenting all findings with equal confidence. Well-established facts get tables and confident prose; contested findings get explicit source comparison; uncertain findings carry hedged language and gap annotations.

---

## Key Takeaways for the Exam

1. **Maintain claim-source mappings as structured data** — inline citations get summarized away; structured mappings survive.
2. **Never silently resolve conflicts** — annotate conflicting statistics with source, methodology, and date rather than picking one.
3. **Require temporal metadata on every data point** — publication date, collection date, and reference period prevent false contradictions.
4. **Match rendering format to content type** — tables for financial data, prose for narratives, side-by-side for conflicts.
5. **Label certainty explicitly** — distinguish well-established findings from contested claims and preliminary intelligence in the output structure.

---

## Related Topics

- [[Domain 5 - Context Management & Reliability]] — Parent domain overview
- [[Context Window Management]] — Preserving structured data through summarization
- [[Error Propagation in Multi-Agent Systems]] — Coverage annotations parallel provenance annotations
- [[Human Review Workflows]] — Confidence calibration and routing based on certainty
- [[Multi-Agent Orchestration]] — Subagent outputs that carry source metadata for coordinator synthesis
