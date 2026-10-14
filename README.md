# Heat Pump Copilot Dashboard — Shape Capstone

Shape AI capstone project: **Heat Pump Copilot Dashboard** (Home Energy & HVAC sector). This repo currently holds the **Round 1** submission; Round 2 will build on top of it in the same place.

## Structure

```
.
├── data/                      # Raw/sample data used for research & dashboard
│   └── manuals/                   # Manual knowledge base grounding the n8n copilot (see manuals/README.md)
├── research/
│   ├── sector_research.md         # Market/industry overview
│   ├── opportunities_risks.md     # Opportunities & risks for the use case
│   ├── use_cases.md               # Candidate + selected use case(s)
│   └── preliminary_analysis.html  # Visual opportunity/risk + dataset-landscape analysis
├── dashboard/
│   ├── dashboard.twbx         # Tableau dashboard (agreed alternative to Power BI)
│   └── dashboard_documentation.md
├── n8n/
│   ├── workflow.json          # Exported n8n automation workflow
│   └── workflow_documentation.md
├── langsmith/
│   ├── run_trace_sample.py    # Traces the Cohere classification step + runs a LangSmith experiment
│   └── monitoring_notes.md    # What's monitored, why it matters, how to run
├── cost_estimation/
│   ├── cost_analysis.md       # Build + run cost breakdown
│   └── timeline_estimate.md   # Milestones & schedule
├── feedback/
│   └── round1_decision.md     # Keep or change industry/use case + why
├── presentation/
│   ├── Heat_Pump_Copilot_Dashboard_Round1.pptx  # Round 1 presentation deck
│   └── one_pager.html         # One-page pitch brief for instructor demo
├── requirements.txt
├── .env.example
└── README.md
```

## Status

In progress. Done: research pack (sector research, opportunities/risks, 3 candidate use cases spanning reactive/preventive/predictive lifecycle stages, preliminary analysis), dataset selection + synthetic fault dataset, cost/timeline estimate, Tableau dashboard (5 market-context sheets + 7 stakeholder operations-metrics sheets), one-page pitch brief for instructor review, n8n fault-triage POC on Telegram (deterministic lookup — now including 8 real Vaillant fault codes — plus Cohere classification fallback grounded in a manual knowledge base, see `data/manuals/`), LangSmith monitoring sample (trace script + dataset/experiment, run and verified end-to-end — 5/5 classifications correct, real experiment link in langsmith/monitoring_notes.md). Presentation deck (`presentation/Heat_Pump_Copilot_Dashboard_Round1.pptx`) is in. Pending: live import/test of the n8n workflow against a real Telegram bot, and the post-presentation `feedback/round1_decision.md`.

## Setup

```bash
cp .env.example .env   # fill in real credentials
pip install -r requirements.txt
```

## Related

GitHub: [anandnarasipuram/Capstone-Project-Heat-Pump-Copilot-Dashboard](https://github.com/anandnarasipuram/Capstone-Project-Heat-Pump-Copilot-Dashboard)
