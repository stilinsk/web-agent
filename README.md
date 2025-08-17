# web-agent
```
import os
import sqlite3
import hashlib
from datetime import datetime, date
from IPython.display import Markdown
from openai import OpenAI, RateLimitError
from crewai_tools import SerperDevTool

# === 🔐 API KEYS (Set them directly here) ===
OPENAI_API_KEY =
SERPER_API_KEY = 

# === Initialize clients ===
client = OpenAI(api_key=OPENAI_API_KEY)
search_tool = SerperDevTool(api_key=SERPER_API_KEY)

# === SQLite database for tracking unique posts ===
class VectorDatabase:
    def __init__(self):
        self.conn = sqlite3.connect("post_history.db")
        self.cursor = self.conn.cursor()
        self.cursor.execute("""
            CREATE TABLE IF NOT EXISTS posts (
                id INTEGER PRIMARY KEY AUTOINCREMENT,
                content_hash TEXT UNIQUE,
                content TEXT,
                created_at TIMESTAMP
            )
        """)
        self.conn.commit()

    def check_unique(self, content):
        content_hash = hashlib.md5(content.encode()).hexdigest()
        self.cursor.execute("SELECT content_hash FROM posts WHERE content_hash = ?", (content_hash,))
        return self.cursor.fetchone() is None

    def save_post(self, content):
        content_hash = hashlib.md5(content.encode()).hexdigest()
        self.cursor.execute("INSERT INTO posts (content_hash, content, created_at) VALUES (?, ?, ?)",
                           (content_hash, content, datetime.now()))
        self.conn.commit()

    def __del__(self):
        self.conn.close()

# === Determine topic of the day ===
def get_daily_topic():
    topics = [
        "Data Engineering and AI",
        "Data Science and AI",
        "Data Analytics and AI"
    ]
    day_of_year = date.today().timetuple().tm_yday
    return topics[day_of_year % len(topics)]

# === Search trends from Serper API ===
def research_trends(topic):
    try:
        query = f"current trends in {topic} 2025"
        results = search_tool.run(query=query, limit=5)
        trends = []
        for result in results.get('organic', []):
            trends.append({
                'title': result.get('title', ''),
                'snippet': result.get('snippet', ''),
                'link': result.get('link', '')
            })
        return trends
    except Exception as e:
        print(f"Error researching trends: {e}")
        return [
            {"title": f"AI-Driven {topic.split(' and ')[0]} Automation", "snippet": f"AI is automating key {topic.lower()} processes, enhancing efficiency."},
            {"title": "Real-Time Insights", "snippet": f"Real-time {topic.lower()} is powered by AI for faster decision-making."},
            {"title": "AI-Enhanced Governance", "snippet": f"AI improves data quality and compliance in {topic.lower()} workflows."}
        ]

# === Generate post using OpenAI ===
def generate_post(topic, trends):
    try:
        field = topic.split(" and ")[0].lower()
        trends_context = "\n".join([f"- {t['title']}: {t['snippet']}" for t in trends])
        prompt = f"""
        You are an expert in {field} tasked with writing a LinkedIn post about '{topic}' for 2025, similar to the style of the following example:

        # How AI is Reshaping Data Engineering: Key Trends for 2025
        One of the most transformative trends in data engineering is the increasing use of AI-powered tools...

        Write a professional LinkedIn post (≤3000 characters) about '{topic}' for 2025.
        - Include a catchy hook, 3-4 key trends with detailed subsections (2-3 sentences each), and a strong CTA.
        - Use markdown formatting with a main heading, subheadings, and 3-5 hashtags.
        - Base the content on the following trends:
        {trends_context}
        """

        response = client.chat.completions.create(
            model="gpt-4",
            messages=[{"role": "user", "content": prompt}],
            temperature=0.6,
            max_tokens=1000
        )
        return response.choices[0].message.content.strip()

    except RateLimitError as e:
        print(f"OpenAI API quota exceeded: {e}")
        return None
    except Exception as e:
        print(f"Error generating post: {e}")
        return None

# === Polish and finalize the post ===
def polish_post(post):
    if not post:
        return None
    lines = post.split("\n")
    if not any(line.startswith("# ") for line in lines):
        lines[0] = f"# {lines[0]}"
    if not any("#" in line for line in lines[-3:]):
        lines.append("\n#DataEngineering #AI #TechTrends2025 #MachineLearning #BigData")
    return "\n".join(lines)

# === Master pipeline ===
def create_linkedin_post(retry=3):
    vector_db = VectorDatabase()

    topic = get_daily_topic()
    print(f"📅 Today's topic: {topic}")

    print("🔍 Researching trends...")
    trends = research_trends(topic)
    if not trends:
        print("⚠️ No trends found, using default.")

    print("🤖 Generating post...")
    post = generate_post(topic, trends)
    if not post:
        print("❌ Failed to generate post.")
        return None

    print("📝 Polishing post...")
    polished_post = polish_post(post)

    if vector_db.check_unique(polished_post):
        vector_db.save_post(polished_post)
        print("✅ Unique post generated and saved.")
        return polished_post
    elif retry > 0:
        print(f"♻️ Duplicate post detected. Retrying ({retry} retries left)...")
        return create_linkedin_post(retry - 1)
    else:
        print("❌ Max retries reached. No unique post.")
        return None

# === Run the script ===
if __name__ == "__main__":
    result = create_linkedin_post()
    if result:
        display(Markdown(result))
    else:
        print("⚠️ No unique post was generated.")

````
