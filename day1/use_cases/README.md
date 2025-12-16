# Practical Industrial Use Cases

## Overview

This session applies the multi-server MCP architecture built in Sessions 2 and 3 to real industrial scenarios. The focus is on having Claude build reusable React dashboards that pull data from multiple MCP servers using natural language prompts.

## Goals

- Apply multi-server MCP architecture to real industrial scenarios
- Demonstrate Claude building React dashboards from natural language prompts
- Show how dashboards can pull from multiple data sources via MCP

## Prerequisites

- Completed Session 2 (MQTT MCP Server)
- Completed Session 3 (MySQL MCP Server)
- Both servers running and configured in Claude Desktop

---

## Use Case 1 -- Production Status Dashboard

### Scenario

Operations manager needs a single view showing:
- Current line status (from UNS/MQTT)
- Active work order details (from MySQL)
- Production progress vs. target

### Prompt to Claude

> "Build me a React dashboard that shows the current status of packaging line 1. Include the current speed from the UNS, the active work order from mes_lite, and a progress bar showing units produced vs. target."

### What Claude Does

1. Queries MQTT server for current line values
2. Queries MySQL server for active work order
3. Generates React component with live data display
4. Creates progress visualization

### Key Teaching Points

- Claude selects appropriate tools automatically
- Data from multiple sources combined in one view
- React artifact is immediately usable

---

## Use Case 2 -- Batch Record Lookup with Live Status

### Scenario

Quality engineer needs to:
- Look up batch details from database
- See current environmental conditions
- Compare to spec limits

### Prompt to Claude

> "Show me batch record B2024-1201 from proveitdb with its quality checks. Also show the current temperature and humidity from the UNS and flag if they're outside the batch spec limits."

### What Claude Does

1. Queries MySQL for batch record and quality data
2. Queries MQTT for current environmental readings
3. Compares values to specifications
4. Generates dashboard with status indicators

### Key Teaching Points

- Cross-referencing historical and real-time data
- Conditional logic based on combined sources
- Alert/warning visualization

---

## Use Case 3 -- Equipment Performance Summary

### Scenario

Maintenance lead needs:
- Current equipment status
- Recent production metrics
- Maintenance history

### Prompt to Claude

> "Create a dashboard for the filler equipment showing current operating status from the UNS, today's production totals from mes_lite, and any recent quality issues from proveitdb."

### What Claude Does

1. Queries MQTT for equipment status tags
2. Queries MySQL mes_lite for production aggregates
3. Queries MySQL proveitdb for quality records
4. Generates multi-panel dashboard

### Key Teaching Points

- Single prompt spans three data sources
- Aggregation and summarization
- Multi-panel layout generation

---

## Prompting Techniques for Dashboards

### Be Specific About Data Sources

**Weak prompt:**
> "Show me production data"

**Strong prompt:**
> "Show me today's production count from mes_lite.production_runs and the current line speed from the UNS"

### Specify Visual Elements

**Weak prompt:**
> "Display the batch status"

**Strong prompt:**
> "Create a card showing batch B2024-1201 with a progress bar, status badge, and table of quality checks"

### Request Interactivity

> "Add a refresh button that re-queries both servers"

> "Include a dropdown to select different production lines"

---

## How Artifacts Use MCP Server Data

### Flow

```
User Prompt
    ↓
Claude reasons about what data is needed
    ↓
Claude calls MCP tools (MQTT and/or MySQL)
    ↓
Data returns to Claude
    ↓
Claude generates React component with embedded data
    ↓
Artifact renders in Claude Desktop
```

### Important Notes

- Artifacts are static snapshots -- data is fetched at generation time
- For live updates, user must regenerate or request refresh logic
- Complex dashboards may need multiple iterations

---

## Iterating on Dashboards

### Refinement Prompts

After initial dashboard generation:

> "Add a timestamp showing when the data was last fetched"

> "Change the color scheme to use our brand colors -- blue #1E40AF and gray #6B7280"

> "Make the progress bar show red when below 80% of target"

> "Add a second tab showing historical trends"

### Layout Adjustments

> "Arrange the cards in a 2x2 grid instead of a single column"

> "Move the status indicator to the top right corner"

> "Make this responsive for mobile"

---

## When to Use MCP vs. Direct Integration

| Approach | Best For | Trade-offs |
|----------|----------|------------|
| MCP + Claude | Exploration, ad-hoc queries, rapid prototyping | Requires Claude, not embedded in production apps |
| Direct API | Production applications, high-frequency updates | More development effort, less flexible |
| Hybrid | Production app with Claude-assisted configuration | Best of both, more complexity |

### MCP Excels At

- Exploratory data analysis
- Building proof-of-concept dashboards
- Natural language queries by non-developers
- Rapid iteration on visualizations

### Consider Direct Integration When

- Dashboard needs sub-second updates
- Deployed to users without Claude access
- Part of a larger production application
- Strict performance requirements

---

## Live Demo -- Building a Dashboard from Prompt

### Demo Flow

1. Start with a business question
2. Craft a prompt describing the desired dashboard
3. Watch Claude query both servers
4. Review the generated React artifact
5. Iterate with refinement prompts
6. Discuss what worked and what could improve

### Sample Starting Prompt

> "I need a production overview dashboard for the Flexible Packager line. Show me:
> - Current line state and speed from the UNS
> - Today's production totals from mes_lite
> - The active batch ID and recipe from proveitdb
> - A simple bar chart of hourly production counts
> 
> Use a clean, modern layout with cards for each section."

---

## Day 1 Wrap-Up

### What We Built

- MQTT MCP server for real-time UNS access
- MySQL MCP server for relational data queries
- Multi-server Claude Desktop configuration
- React dashboards from natural language

### Key Takeaways

- MCP bridges AI and industrial data
- Multi-server architecture enables rich queries
- Natural language replaces manual dashboard building
- Iterate quickly from prompt to visualization

---

## Preview -- Day 2 (Agent2Agent)

Tomorrow we extend this foundation:

- Agents that coordinate with each other
- Specialized agents for different domains
- Workflows that span multiple agents
- Industrial automation with collaborative AI

---

## Code Publishing

All code from Day 1 will be pushed to GitHub:

**Repository:** github.com/iiot-university/MCP_A2A_Workshop

Students can clone and replicate after the session.

---

## Files in This Section

| File | Purpose |
|------|---------|
| `README.md` | This guide |
| `example_prompts.md` | Collection of dashboard prompts to try |
| `sample_dashboards/` | Screenshots of generated dashboards |

---

## Troubleshooting

### Dashboard doesn't render

- Check that artifact generation is enabled in Claude Desktop
- Verify React syntax in generated code
- Look for JavaScript errors in browser console

### Data appears stale

- MCP queries run at generation time
- Request a refresh button in the dashboard
- Re-run the prompt to get fresh data

### Claude uses wrong data source

- Be explicit about which server/schema to use
- Reference table names directly
- Check tool descriptions in MCP servers

---

## Resources

- [React Documentation](https://react.dev)
- [Tailwind CSS](https://tailwindcss.com)
- [Recharts](https://recharts.org) -- charting library available in Claude artifacts
- [MCP Documentation](https://modelcontextprotocol.io)
