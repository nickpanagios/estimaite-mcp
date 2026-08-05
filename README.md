# Estimaite MCP server

Construction takeoff and estimating for AI agents.

Estimaite reads a construction drawing PDF, measures and counts the scope on the sheet, prices it against
editable material and labour rates, and returns an estimate. This repository holds the public manifest and
connection docs for its **remote MCP server**, so an MCP client can drive the whole job: upload a plan set,
set or auto-detect scale, take off by trade, price it, and export a priced estimate or proposal.

The server is hosted. There is nothing to install and no source in this repository.

## Connect

**Endpoint**

```
https://mcp.esti-maite.com/mcp
```

Transport is streamable HTTP. Authentication is OAuth 2.1: the client opens a browser, you approve the
connection against your Estimaite account, and the client stores the token. Any MCP client that supports
remote servers with OAuth will work, including Claude and other MCP-capable assistants.

**Claude Desktop / Claude Code**

Add it as a remote MCP server pointing at the endpoint above and complete the browser approval when prompted.

## What it can do

Roughly, and grouped the way an estimator works:

- **Drawings** — upload a plan set, list sheets, read a sheet, extract text and schedules from it
- **Scale** — calibrate from a known dimension, or auto-detect the plan scale
- **Takeoff** — count symbols, measure linear runs, areas and volumes, organise into layers by system, size
  and material, and audit a sheet for what was missed
- **Estimating** — price takeoffs against cost items, apply overhead and profit, and produce a proposal
- **Projects** — create and manage projects, clients and assignees, and read the bid board

The full tool reference is in [`docs/tools.md`](docs/tools.md). A connected client lists the live tool set,
which is always the authoritative answer.

## Trades

Nine, covering concrete and masonry, structural and misc steel, carpentry and framing, drywall, insulation and
ceilings, HVAC and sheet metal, plumbing and fire protection, electrical and low voltage, finishes and roofing,
and sitework and utilities. The method follows how each trade is actually taken off rather than one generic
workflow.

## Accuracy

Estimaite makes no published accuracy claim. Measurement accuracy is being evaluated on a small sample and no
number is published, so none is asserted here. What the product does claim is structural: measurements are
produced before an estimator opens the drawing, and the estimator reviews them against an overlay of what was
measured.

## Links

- Product: https://www.esti-maite.com
- How agents drive it: https://www.esti-maite.com/mcp
- About: https://www.esti-maite.com/about
- Support: support@esti-maite.com
