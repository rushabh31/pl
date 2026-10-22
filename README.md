# PromptLab design package

Exported 22 October 2026 from the PromptLab doc. Start with `00-Overview.md`.

| File | What it holds |
|---|---|
| `00-Overview.md` | Summary, map of the package, where to start by role, key decisions, roadmap |
| `01-Architecture-and-design.md` | The full design: research, principles, architecture, interception planes, memoized execution, API contracts, workflow inputs, evaluation, statistics, security, roadmap, decisions |
| `02-Diagrams.html` | Architecture with supported features, user journey, and the refund service's API flow and dataflow |
| `03-Requirements.md` | Testable requirements for every phase, API-contract compatibility, non-functional and UI/UX requirements |
| `04-Integration-cookbook.md` | Every supported service shape, ADK and LangGraph scenario catalogs, and worked end-to-end examples |
| `05-Workflow-developer-guide.md` | How to build ADK and LangGraph services that integrate cleanly, with a readiness checklist |
| `diagrams-source/*.jsx` | Source of each diagram, for editing and re-rendering |

## About the diagrams

`02-Diagrams.html` is a single self-contained page with no external files or scripts. Each diagram is inline SVG, so it stays sharp at any zoom, its text can be selected and searched, and it follows the system's light or dark theme. Open it in any browser; printing to PDF keeps each diagram on one page.

To change a diagram, edit its file in `diagrams-source/` (a small JSX module that returns one SVG) and re-render it with React's `renderToStaticMarkup`.

## Notes

- Code in the cookbook and developer guide uses each framework's documented API. Everything named `promptlab` or `pl` is the proposed PromptLab interface from this design, and model names are placeholders.
- Numbers marked *(proposed)* in the requirements are starting points to confirm with your team.
