What already exists:

techcrunch_summary_workflow.json — the n8n workflow, ready to import
README with credential setup steps
What needs to happen to run n8n locally on this Windows PC:

n8n is a Node.js app, not a Python app — so "venv" here means we use a local Node.js environment isolated to this project folder (using npm locally, not globally). Here's exactly what I'll do:

Step 1 — Check prerequisites

Check if node and npm are installed and their versions
n8n requires Node.js 18+
Step 2 — Set up local n8n in the project folder (the "venv" equivalent for Node)

Run npm init -y inside d:\n8n news summary workflow\ to create a local package.json
Install n8n locally: npm install n8n (this keeps it isolated to this folder, not global)
Add an npm start script so you can launch it with npm start
Step 3 — Create a .env file

Add environment variables: N8N_PORT, N8N_BASIC_AUTH_ACTIVE, N8N_USER_FOLDER (points to local data folder so workflow data stays in project dir)
This .env will also be reused later for Render deployment
Step 4 — Create a launch script

A start.cmd script that loads the .env and starts n8n, so you don't have to type commands manually each time
Step 5 — Add .gitignore

Ignore node_modules/, .env, and n8n's local data folder so secrets and heavy deps don't get committed
Step 6 — Verify it starts

Run n8n, confirm it opens at http://localhost:5678
Walk you through importing the workflow JSON and connecting credentials
What I will NOT touch:

The workflow JSON — no changes to it
The README / implementation plan
Any workflow logic or architecture
Your action items (I can't do these for you):

Provide your Gemini/OpenAI API key and Gmail OAuth credentials after n8n is running
Do the browser-based credential setup in the n8n UI

What's done:

✅ n8n installed locally in node_modules/ (project-scoped, not global)
✅ .env configured with port, timezone, data folder pointing to .n8n/ in project
✅ start.cmd — double-click this anytime to start n8n or to bring n8n back up.
✅ .gitignore — excludes node_modules/, .n8n/, .env
✅ n8n is running right now at http://localhost:5678