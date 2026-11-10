"""Fetch public repositories and rewrite the project table in README.md."""
import json
import os
import re
import urllib.request

USER = os.environ.get("GH_USER", "ayoub-bakkouri")
TOKEN = os.environ.get("GITHUB_TOKEN", "")
MAX_PROJECTS = 6
EXCLUDE = {USER.lower(), f"{USER.lower()}.github.io"}  # profile repo, pages repo
README = "README.md"

START = "<!--PROJECTS:START-->"
END = "<!--PROJECTS:END-->"


def fetch_repos():
    url = f"https://api.github.com/users/{USER}/repos?per_page=100&sort=pushed"
    headers = {"Accept": "application/vnd.github+json", "User-Agent": "profile-readme"}
    if TOKEN:
        headers["Authorization"] = f"Bearer {TOKEN}"
    with urllib.request.urlopen(urllib.request.Request(url, headers=headers)) as r:
        return json.load(r)


def build_table(repos):
    rows = [
        "| Project | Description | Language | Stars | Last update |",
        "| --- | --- | --- | --- | --- |",
    ]
    for r in repos:
        desc = (r.get("description") or "No description yet").replace("|", "/")
        lang = r.get("language") or "-"
        updated = (r.get("pushed_at") or "")[:10]
        rows.append(
            f"| [{r['name']}]({r['html_url']}) | {desc} | {lang} | {r['stargazers_count']} | {updated} |"
        )
    return "\n".join(rows)


def main():
    repos = [
        r
        for r in fetch_repos()
        if not r["fork"] and not r["archived"] and r["name"].lower() not in EXCLUDE
    ]
    # Most starred first, then most recently pushed
    repos.sort(key=lambda r: (r["stargazers_count"], r["pushed_at"]), reverse=True)
    table = build_table(repos[:MAX_PROJECTS])

    with open(README, encoding="utf-8") as f:
        content = f.read()

    pattern = re.compile(f"{re.escape(START)}.*?{re.escape(END)}", re.DOTALL)
    new_content = pattern.sub(f"{START}\n{table}\n{END}", content)

    if new_content != content:
        with open(README, "w", encoding="utf-8") as f:
            f.write(new_content)
        print("README updated")
    else:
        print("No changes")


if __name__ == "__main__":
    main()
