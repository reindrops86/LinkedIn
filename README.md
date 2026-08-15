I built an automated content pipeline that generates LinkedIn-style posts from structured brand inputs and routes them through a controlled approval flow before publishing.
The architecture has three layers.
First, a FastAPI microservice exposes a linkedin endpoint and a health endpoint. It accepts topic, audience, and tone, then returns a formatted post payload.
Second, an n8n orchestration workflow handles end-to-end logic: Brand Config, AutoGen Microservice call, Compose Final, Approval Gate, and delivery routing.
Third, Slack is used as a dry-run destination so we can validate output safely before enabling live LinkedIn publishing.
The key technical decisions were about reliability and safety.
I normalized all generated output into one field called final_post so downstream nodes are deterministic.
I added an approval gate to block weak or empty content and route failed cases to a rejection path instead of publishing.
I also added a mode-based delivery strategy so the same workflow supports dry-run and live mode without rewiring.
For validation, I tested each component independently first, then tested node-by-node, then ran repeated end-to-end executions with different topics.
