# LinkedIn

import os
import sys
from pathlib import Path

import requests
from dotenv import load_dotenv
from crewai import Agent, Crew, LLM, Task
from crewai.tools import BaseTool


load_dotenv()

if hasattr(sys.stdout, "reconfigure"):
    sys.stdout.reconfigure(encoding="utf-8")
if hasattr(sys.stderr, "reconfigure"):
    sys.stderr.reconfigure(encoding="utf-8")


class TavilySearchTool(BaseTool):
    """Search recent web information for the research agent."""

    name: str = "Web Search"
    description: str = "Search the web for recent information about AI and FMCG trends."

    def _run(self, query: str) -> str:
        api_key = os.getenv("TAVILY_API_KEY")
        if not api_key:
            raise ValueError(
                "TAVILY_API_KEY is missing. Add it to your .env file or environment variables."
            )

        response = requests.post(
            "https://api.tavily.com/search",
            json={
                "api_key": api_key,
                "query": query,
                "max_results": 5,
            },
            timeout=30,
        )
        response.raise_for_status()
        data = response.json()

        results = []
        for item in data.get("results", []):
            title = item.get("title", "Untitled")
            url = item.get("url", "")
            snippet = item.get("content") or item.get("snippet") or ""
            result_text = f"{title} - {url}"
            if snippet:
                result_text = f"{result_text}\n{snippet[:300]}"
            results.append(result_text)

        if not results:
            return "No web search results were returned."

        return "\n\n".join(results)


def get_openai_api_key() -> str:
    api_key = os.getenv("OPENAI_API_KEY") or os.getenv("AZURE_OPENAI_API_KEY")
    if not api_key:
        raise ValueError(
            "No API key found. Set OPENAI_API_KEY or AZURE_OPENAI_API_KEY in your .env file."
        )
    return api_key


def build_llm(api_key: str):
    return LLM(
        model="gpt-4o-mini",
        api_key=api_key,
        temperature=0,
        max_completion_tokens=3500,
    )


def build_crew():
    search_tool = TavilySearchTool()
    api_key = get_openai_api_key()
    os.environ["OPENAI_API_KEY"] = api_key

    llm = build_llm(api_key)

    researcher = Agent(
        role="AI Researcher",
        goal="Find the latest advancements in AI for FMCG using web search.",
        backstory=(
            "You are an expert in AI research trends and can identify recent, credible "
            "advancements relevant to fast-moving consumer goods."
        ),
        verbose=False,
        allow_delegation=False,
        llm=llm,
        max_iter=2,
        tools=[search_tool],
    )

    writer = Agent(
        role="Executive Writer",
        goal="Turn findings into a crisp, leadership-ready summary.",
        backstory="You turn messy research into clear, concise executive communication.",
        verbose=False,
        allow_delegation=False,
        llm=llm,
        max_iter=2,
        tools=[search_tool],
    )

    task_research = Task(
        description=(
            "Use web search to identify the top 3 recent AI advancements relevant to FMCG. "
            "Focus on practical commercial use cases, benefits, and examples."
        ),
        expected_output=(
            "A structured list of the top 3 AI advancements in FMCG with brief explanations, examples, "
            "and why they matter."
        ),
        agent=researcher,
    )

    task_write = Task(
        description=(
            "Write a concise executive summary based on the research notes. "
            "Requirements: max 100 words, use bullet points, and only cover the 3 key advancements."
        ),
        expected_output="A brief executive summary in bullet-point format under 100 words.",
        agent=writer,
        context=[task_research],
    )

    return Crew(
        agents=[researcher, writer],
        tasks=[task_research, task_write],
        verbose=False,
    )


def save_output(output_text: str) -> Path:
    output_dir = Path(__file__).resolve().parent / "output"
    output_dir.mkdir(exist_ok=True)
    output_file = output_dir / "fmcg_ai_summary.txt"
    output_file.write_text(output_text, encoding="utf-8")
    return output_file


def main():
    try:
        crew = build_crew()
        result = crew.kickoff()
    except ValueError as exc:
        print(f"Configuration error: {exc}")
        print("Add your API keys to a .env file or export them in your shell before running this script.")
        return 1
    except Exception as exc:  # pragma: no cover - keeps script usable for debugging
        print(f"The workflow failed: {exc}")
        return 1

    print("\nFinal Output:\n")
    output = result.raw if hasattr(result, "raw") else str(result)
    print(output)

    output_file = save_output(output)
    print(f"\nSaved summary to: {output_file}")
    return 0


if __name__ == "__main__":
    raise SystemExit(main())
