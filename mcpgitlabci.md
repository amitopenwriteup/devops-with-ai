# server.py
import asyncio
import json
from mcp.server import Server
from mcp.server.stdio import stdio_server
from mcp.types import Tool, TextContent

import git_tools
import ollama_client
import ci_generator

app = Server("git-mcp-server")

# --------------------------------------------------
#  Tool Definitions
# --------------------------------------------------

@app.list_tools()
async def list_tools() -> list[Tool]:
    return [
        Tool(
            name="scan_repo",
            description=(
                "Scan a local git repository. Returns branch, recent commits, "
                "file structure, and detected tech stack."
            ),
            inputSchema={
                "type": "object",
                "properties": {
                    "repo_path": {
                        "type": "string",
                        "description": "Absolute path to the local git repository",
                    }
                },
                "required": ["repo_path"],
            },
        ),
        Tool(
            name="get_changeset",
            description=(
                "Get the current git changeset: staged, unstaged, and untracked files, "
                "plus the full diff content."
            ),
            inputSchema={
                "type": "object",
                "properties": {
                    "repo_path": {
                        "type": "string",
                        "description": "Absolute path to the local git repository",
                    }
                },
                "required": ["repo_path"],
            },
        ),
        Tool(
            name="analyze_issues",
            description=(
                "Use Ollama (local LLM) to analyze the current git diff for bugs, "
                "security issues, or code quality problems."
            ),
            inputSchema={
                "type": "object",
                "properties": {
                    "repo_path": {
                        "type": "string",
                        "description": "Absolute path to the local git repository",
                    },
                    "model": {
                        "type": "string",
                        "description": "Ollama model to use (default: llama3.2:1b)",
                        "default": "llama3.2:1b",
                    },
                },
                "required": ["repo_path"],
            },
        ),
        Tool(
            name="generate_ci",
            description=(
                "Generate a .gitlab-ci.yml file for the repository using Ollama. "
                "Scans the repo first, then uses the LLM to produce tailored CI config."
            ),
            inputSchema={
                "type": "object",
                "properties": {
                    "repo_path": {
                        "type": "string",
                        "description": "Absolute path to the local git repository",
                    },
                    "model": {
                        "type": "string",
                        "description": "Ollama model to use (default: llama3.2:1b)",
                        "default": "llama3.2:1b",
                    },
                    "save_to_repo": {
                        "type": "boolean",
                        "description": "Write .gitlab-ci.yml to the repo root (default: false)",
                        "default": False,
                    },
                },
                "required": ["repo_path"],
            },
        ),
        Tool(
            name="list_ollama_models",
            description="List all available Ollama models installed on this machine.",
            inputSchema={"type": "object", "properties": {}},
        ),
    ]


# --------------------------------------------------
#  Tool Handlers
# --------------------------------------------------

@app.call_tool()
async def call_tool(name: str, arguments: dict) -> list[TextContent]:

    if name == "scan_repo":
        result = git_tools.scan_repo(arguments["repo_path"])
        return [TextContent(type="text", text=json.dumps(result, indent=2))]

    elif name == "get_changeset":
        result = git_tools.get_changeset(arguments["repo_path"])
        return [TextContent(type="text", text=json.dumps(result, indent=2))]

    elif name == "analyze_issues":
        changeset = git_tools.get_changeset(arguments["repo_path"])
        if "error" in changeset:
            return [TextContent(type="text", text=f"Error: {changeset['error']}")]
        if not changeset.get("has_changes"):
            return [TextContent(type="text", text="No changes detected in the repository.")]
        diff = changeset.get("diff", "")
        model = arguments.get("model", "llama3.2:1b")
        analysis = ollama_client.analyze_diff(diff, model=model)
        return [TextContent(type="text", text=analysis)]

    elif name == "generate_ci":
        repo_path = arguments["repo_path"]
        model = arguments.get("model", "llama3.2:1b")
        save = arguments.get("save_to_repo", False)

        scan = git_tools.scan_repo(repo_path)
        if "error" in scan:
            return [TextContent(type="text", text=f"Error: {scan['error']}")]

        ci_yaml = ci_generator.generate_gitlab_ci(scan, model=model)

        if save:
            output_path = f"{repo_path}/.gitlab-ci.yml"
            with open(output_path, "w") as f:
                f.write(ci_yaml)
            return [TextContent(
                type="text",
                text=f"Saved to {output_path}\n\n```yaml\n{ci_yaml}\n```",
            )]

        return [TextContent(type="text", text=f"```yaml\n{ci_yaml}\n```")]

    elif name == "list_ollama_models":
        models = ollama_client.list_models()
        if not models:
            return [TextContent(type="text", text="No Ollama models found. Run: ollama pull llama3.2:1b")]
        return [TextContent(type="text", text="Available models:\n" + "\n".join(f"- {m}" for m in models))]

    else:
        return [TextContent(type="text", text=f"Unknown tool: {name}")]


# --------------------------------------------------
#  Entry Point
# --------------------------------------------------

async def main():
    async with stdio_server() as (read, write):
        await app.run(read, write, app.create_initialization_options())


if __name__ == "__main__":
    asyncio.run(main())
