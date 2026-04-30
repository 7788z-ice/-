openai>=1.0.0
requests>=2.31.0
beautifulsoup4>=4.12.0
pydantic>=2.0.0
python-dotenv>=1.0.0
feedparser>=6.0.0
markdown>=3.5.0
typing-extensions>=4.8.0
"""
config.py
import os
from dotenv import load_dotenv
from pydantic import BaseModel
from typing import List, Dict, Optional

load_dotenv()

class Config:
    OpenAI 配置（或兼容的 API）
    OPENAI_API_KEY = os.getenv("OPENAI_API_KEY", "your-api-key")
    OPENAI_BASE_URL = os.getenv("OPENAI_BASE_URL", "https://api.openai.com/v1")
    OPENAI_MODEL = os.getenv("OPENAI_MODEL", "gpt-4o-mini")
    
    竞品监控配置
    COMPETITORS = {
        "competitor_a": {
            "name": "竞品A",
            "release_notes_url": "https://example.com/release-notes",
            "help_center_url": "https://example.com/help",
            "blog_url": "https://example.com/blog"
        },
        "competitor_b": {
            "name": "竞品B", 
            "rss_feed": "https://example.com/feed.xml"
        }
    }
    
    内部 PRD 存储路径
    PRD_DIRECTORY = "./prd_docs"
    PRD_VECTOR_DB_PATH = "./prd_vector_db"
    
    Agent 调度配置
    CHECK_INTERVAL_HOURS = 24  每日检查一次
    MAX_FEEDS_PER_RUN = 50
    
    输出配置
    NOTIFICATION_WEBHOOK = os.getenv("DINGTALK_WEBHOOK", "")
    REPORT_OUTPUT_DIR = "./reports"


class PRDDocument(BaseModel):
   
    file_name: str
    feature_name: str
    feature_category: str
    planned_version: Optional[str] = None
    status: str  # planned, developing, released, deprecated
    description: str
    key_points: List[str]


class CompetitorUpdate(BaseModel):
   
    source: str  # competitor_a, competitor_b
    update_type: str  # new_feature, ui_change, pricing_change, doc_update
    title: str
    content: str
    url: str
    published_at: str
    raw_html: Optional[str] = None


class GapAnalysis(BaseModel):
 
    original_update: CompetitorUpdate
    is_covered: bool  # 我们的 PRD 是否已覆盖
    match_reason: str  # 匹配/不匹配的原因
    our_planned_feature: Optional[str] = None  # 匹配到的我们已有的功能
    gap_severity: str  # critical, major, minor, info
    suggested_action: str  # 建议行动：ignore, review, add_to_backlog, escalate
    core/web_scraper.py
import requests
import feedparser
from bs4 import BeautifulSoup
from typing import List, Optional
from datetime import datetime
import hashlib
from config import Config, CompetitorUpdate

class WebScraperAgent:
    
    def __init__(self):
        self.session = requests.Session()
        self.session.headers.update({
            "User-Agent": "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36",
            "Accept": "text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8"
        })
        self.seen_hashes = set() 
    
    def fetch_webpage(self, url: str) -> Optional[str]:
        """抓取单个网页"""
        try:
            response = self.session.get(url, timeout=10)
            response.raise_for_status()
            return response.text
        except Exception as e:
            print(f"抓取失败 {url}: {e}")
            return None
    
    def parse_html_content(self, html: str, url: str) -> List[CompetitorUpdate]:
     
        soup = BeautifulSoup(html, "html.parser")
        
        
        for script in soup(["script", "style", "nav", "footer"]):
            script.decompose()
        
        text = soup.get_text()
        lines = [line.strip() for line in text.splitlines() if line.strip()]
        content = "\n".join(lines[:500])  
        
        
        content_hash = hashlib.md5(content.encode()).hexdigest()
        
        if content_hash in self.seen_hashes:
            return []
        
        self.seen_hashes.add(content_hash)
        
      
        title = soup.find("title")
        title_text = title.get_text() if title else "无标题"
        
        return [CompetitorUpdate(
            source=url,
            update_type="doc_update",
            title=title_text[:200],
            content=content[:3000],
            url=url,
            published_at=datetime.now().isoformat(),
            raw_html=html[:5000]
        )]
    
    def fetch_rss_feed(self, feed_url: str) -> List[CompetitorUpdate]:
        """抓取 RSS Feed"""
        updates = []
        try:
            feed = feedparser.parse(feed_url)
            for entry in feed.entries[:Config.MAX_FEEDS_PER_RUN]:
                content = entry.get("summary", "") or entry.get("description", "")
                update = CompetitorUpdate(
                    source=feed_url,
                    update_type="doc_update",
                    title=entry.get("title", "无标题")[:200],
                    content=content[:3000] if content else entry.get("link", ""),
                    url=entry.get("link", ""),
                    published_at=entry.get("published", datetime.now().isoformat())
                )
                updates.append(update)
        except Exception as e:
            print(f"RSS 抓取失败 {feed_url}: {e}")
        
        return updates
    
    def fetch_all_updates(self) -> List[CompetitorUpdate]:
        """抓取所有配置的竞品更新"""
        all_updates = []
        
        for competitor_key, competitor_info in Config.COMPETITORS.items():
            print(f"正在抓取 {competitor_info.get('name', competitor_key)}...")
            
            抓取 Release Notes
            if url := competitor_info.get("release_notes_url"):
                html = self.fetch_webpage(url)
                if html:
                    all_updates.extend(self.parse_html_content(html, url))
            
            抓取 RSS Feed
            if rss_url := competitor_info.get("rss_feed"):
                all_updates.extend(self.fetch_rss_feed(rss_url))
        
        print(f"共抓取到 {len(all_updates)} 条更新")
        return all_updates
        # core/prd_retriever.py
import os
import json
from typing import List, Optional
import hashlib
from pathlib import Path
from config import Config, PRDDocument, GapAnalysis, CompetitorUpdate

class PRDRetrieverAgent:
   
    
    def __init__(self):
        self.prd_documents: List[PRDDocument] = []
        self._load_prd_documents()
    
    def _load_prd_documents(self):
     
        prd_path = Path(Config.PRD_DIRECTORY)
        if not prd_path.exists():
           
            self._create_sample_prd()
        
        for file_path in prd_path.glob("*.md"):
            prd = self._parse_prd_markdown(file_path)
            if prd:
                self.prd_documents.append(prd)
        
        for file_path in prd_path.glob("*.json"):
            prd = self._parse_prd_json(file_path)
            if prd:
                self.prd_documents.append(prd)
        
        print(f"已加载 {len(self.prd_documents)} 个 PRD 文档")
    
    def _create_sample_prd(self):
        """创建示例 PRD 文档用于演示"""
        os.makedirs(Config.PRD_DIRECTORY, exist_ok=True)
        
        sample_prds = [
            {
                "file_name": "user_center.md",
                "feature_name": "统一用户中心",
                "feature_category": "用户管理",
                "planned_version": "v3.0",
                "status": "developing",
                "description": "支持手机号、邮箱、微信多方式登录，提供统一的用户资料管理界面",
                "key_points": ["OAuth2.0集成", "多端同步", "安全加固"]
            },
            {
                "file_name": "payment_v2.json", 
                "feature_name": "支付系统升级V2",
                "feature_category": "交易",
                "planned_version": "v2.5",
                "status": "planned",
                "description": "新增微信支付、支付宝快捷支付，支持分期付款",
                "key_points": ["微信支付集成", "分期付款", "自动对账"]
            }
        ]
        
        for prd in sample_prds:
            file_path = prd_path / prd["file_name"]
            if file_path.suffix == ".md":
                content = f"""# {prd['feature_name']}

 {prd['feature_category']}
 {prd['planned_version']}
 {prd['status']}


{prd['description']}


{chr(10).join(f'- {point}' for point in prd['key_points'])}
"""
                file_path.write_text(content)
            else:
                file_path.write_text(json.dumps(prd, ensure_ascii=False, indent=2))
    
    def _parse_prd_markdown(self, file_path: Path) -> Optional[PRDDocument]:

        content = file_path.read_text(encoding="utf-8")
        
        lines = content.split("\n")
        feature_name = ""
        feature_category = ""
        planned_version = ""
        status = "planned"
        description = ""
        key_points = []
        
        for i, line in enumerate(lines):
            if line.startswith(" "):
                feature_name = line[2:].strip()
            elif line.startswith("功能分类"):
                feature_category = line.split(" ")[-1].strip()
            elif line.startswith("计划版本"):
                planned_version = line.split("**")[-1].strip()
            elif line.startswith("状态"):
                status = line.split(" ")[-1].strip()
            elif line.startswith("功能描述"):
                description = lines[i+1].strip() if i+1 < len(lines) else ""
            elif line.startswith(" ") and description: 
                key_points.append(line[2:].strip())
        
        if not feature_name:
            return None
        
        return PRDDocument(
            file_name=file_path.name,
            feature_name=feature_name,
            feature_category=feature_category or "未分类",
            planned_version=planned_version or None,
            status=status,
            description=description[:500],
            key_points=key_points[:5]
        )
    
    def _parse_prd_json(self, file_path: Path) -> Optional[PRDDocument]:
        """解析 JSON 格式的 PRD"""
        try:
            data = json.loads(file_path.read_text(encoding="utf-8"))
            return PRDDocument(
                file_name=file_path.name,
                feature_name=data.get("feature_name", ""),
                feature_category=data.get("feature_category", "未分类"),
                planned_version=data.get("planned_version"),
                status=data.get("status", "planned"),
                description=data.get("description", ""),
                key_points=data.get("key_points", [])
            )
        except Exception as e:
            print(f"解析 JSON 失败 {file_path}: {e}")
            return None
    
    def retrieve_relevant_prds(self, update: CompetitorUpdate, top_k: int = 3) -> List[tuple[PRDDocument, float]]:
        """检索与竞品更新相关的 PRD 文档
        
        实际应用中应使用向量数据库（如 Chroma、FAISS）和 Embedding
        这里使用简单的关键词匹配作为演示
        """
        import re
        from collections import Counter
        
        提取关键词
        update_text = f"{update.title} {update.content[:500]}".lower()
        words = re.findall(r'\w+', update_text)
        word_counts = Counter(words)
        keywords = [w for w, c in word_counts.most_common(20) if len(w) > 2 and w not in ['the', 'and', 'for', 'this', 'with']]
        
        if not keywords:
            return []
        
        计算相似度
        relevance_scores = []
        for prd in self.prd_documents:
            prd_text = f"{prd.feature_name} {prd.description} {' '.join(prd.key_points)}".lower()
            
            简单的 Jaccard 相似度
            prd_words = set(re.findall(r'\w+', prd_text))
            intersection = len(set(keywords) & prd_words)
            union = len(set(keywords) | prd_words)
            similarity = intersection / union if union > 0 else 0
            
            if similarity > 0.05:  最小阈值
                relevance_scores.append((prd, similarity))
        
        按相似度排序并返回 top_k
        relevance_scores.sort(key=lambda x: x[1], reverse=True)
        return relevance_scores[:top_k]
        core/analyzer.py
import json
from typing import Optional
from openai import OpenAI
from config import Config, CompetitorUpdate, PRDDocument, GapAnalysis

class LLMAnalyzerAgent:
    
    def __init__(self):
        self.client = OpenAI(
            api_key=Config.OPENAI_API_KEY,
            base_url=Config.OPENAI_BASE_URL
        )
        self.model = Config.OPENAI_MODEL
    
    def analyze_update_with_prd(
        self, 
        update: CompetitorUpdate, 
        relevant_prds: list[tuple[PRDDocument, float]]
    ) -> GapAnalysis:
        
        prd_context = ""
        if relevant_prds:
            for prd, score in relevant_prds:
                prd_context += f"""
相关 PRD: {prd.feature_name} (相似度: {score:.2f})
 {prd.feature_category}
 {prd.status}
 {prd.planned_version or '未指定'}
 {prd.description}
- 核心要点: {', '.join(prd.key_points)}
"""
        else:
            prd_context = "未找到相关的 PRD 文档"
        
        构建提示词
        system_prompt = """你是一个专业的竞品分析专家。你的任务是比较竞品的更新内容与内部 PRD（产品需求文档），判断差异。

请按照以下思维链（Chain of Thought）进行分析：
1. 理解竞品更新的核心功能/变化是什么
2. 评估这个变化的重要性（critical/major/minor/info）
3. 检查我们的 PRD 中是否有类似的功能规划
4. 如果有匹配，说明覆盖率；如果没有，判断这是否是一个我们需要跟进的功能
5. 给出具体的建议行动

输出必须是严格的 JSON 格式。"""
        
        user_prompt = f"""
竞品更新内容
来源: {update.source}
标题: {update.title}
内容摘要: {update.content[:2000]}
更新时间: {update.published_at}

内部 PRD 上下文
{prd_context}

请分析并返回 JSON。"""
        
        try:
            response = self.client.chat.completions.create(
                model=self.model,
                messages=[
                    {"role": "system", "content": system_prompt},
                    {"role": "user", "content": user_prompt}
                ],
                temperature=0.3,
                response_format={"type": "json_object"}
            )
            
            result = json.loads(response.choices[0].message.content)
            
            解析 LLM 返回结果
            is_covered = result.get("is_covered", False)
            match_reason = result.get("match_reason", "")
            
            映射严重程度
            severity_map = {
                "critical": "critical",
                "major": "major", 
                "minor": "minor",
                "info": "info"
            }
            gap_severity = severity_map.get(result.get("gap_severity", "info"), "info")
            
            映射建议行动
            action_map = {
                "ignore": "ignore",
                "review": "review",
                "add_to_backlog": "add_to_backlog",
                "escalate": "escalate"
            }
            suggested_action = action_map.get(result.get("suggested_action", "review"), "review")
            
            return GapAnalysis(
                original_update=update,
                is_covered=is_covered,
                match_reason=match_reason[:500],
                our_planned_feature=result.get("our_planned_feature"),
                gap_severity=gap_severity,
                suggested_action=suggested_action
            )
            
        except Exception as e:
            print(f"LLM 分析失败: {e}")
            返回默认分析结果
            return GapAnalysis(
                original_update=update,
                is_covered=False,
                match_reason=f"分析失败: {str(e)}",
                gap_severity="info",
                suggested_action="review"
            )
            core/notifier.py
import requests
import json
from datetime import datetime
from typing import List
from config import Config, GapAnalysis

class NotifierAgent:
    """负责通知和报告生成"""
    
    def generate_markdown_report(self, analyses: List[GapAnalysis]) -> str:
        """生成 Markdown 格式的分析报告"""
        timestamp = datetime.now().strftime("%Y-%m-%d %H:%M:%S")
        
        分类统计
        covered = [a for a in analyses if a.is_covered]
        uncovered = [a for a in analyses if not a.is_covered]
        critical_gaps = [a for a in analyses if a.gap_severity == "critical"]
        major_gaps = [a for a in analyses if a.gap_severity == "major"]
        
        report = f"""# 竞品情报与PRD差异检测报告

生成时间: {timestamp}
分析更新总数: {len(analyses)}
已覆盖: {len(covered)}
未覆盖: {len(uncovered)}
> 关键差异: {len(critical_gaps)} | 主要差异: {len(major_gaps)}

---
"""
        
        if critical_gaps:
            report += "\n🔴 关键差异（必须处理）\n\n"
            for gap in critical_gaps:
                report += f"""
{gap.original_update.title}
- 竞品: {gap.original_update.source}
- 分析: {gap.match_reason}
- 建议行动: {gap.suggested_action}
- [查看原文]({gap.original_update.url})

"""
        
        if major_gaps:
            report += "\n 🟡 主要差异（建议跟进）\n\n"
            for gap in major_gaps:
                report += f"""
{gap.original_update.title}
- 竞品: {gap.original_update.source}
- 分析: {gap.match_reason}

"""
        
        if uncovered:
            report += "\n  未覆盖功能清单\n\n"
            report += "| 竞品 | 标题 | 严重程度 | 建议行动 |\n"
            report += "|------|------|----------|----------|\n"
            for gap in uncovered:
                report += f"| {gap.original_update.source} | {gap.original_update.title[:50]} | {gap.gap_severity} | {gap.suggested_action} |\n"
        
        return report
    
    def send_to_dingtalk(self, report: str, webhook: str = None) -> bool:
        """发送到钉钉群"""
        webhook = webhook or Config.NOTIFICATION_WEBHOOK
        if not webhook:
            print("未配置钉钉 Webhook")
            return False
        
        钉钉消息最大长度限制
        if len(report) > 5000:
            report = report[:4500] + "\n\n... (报告过长已截断)"
        
        data = {
            "msgtype": "markdown",
            "markdown": {
                "title": "竞品情报日报",
                "text": report
            }
        }
        
        try:
            response = requests.post(webhook, json=data, timeout=5)
            result = response.json()
            if result.get("errcode") == 0:
                print("钉钉通知发送成功")
                return True
            else:
                print(f"钉钉通知失败: {result}")
                return False
        except Exception as e:
            print(f"发送钉钉通知失败: {e}")
            return False
    
    def save_report(self, report: str) -> str:
        """保存报告到本地"""
        import os
        os.makedirs(Config.REPORT_OUTPUT_DIR, exist_ok=True)
        
        timestamp = datetime.now().strftime("%Y%m%d_%H%M%S")
        file_path = f"{Config.REPORT_OUTPUT_DIR}/competitor_report_{timestamp}.md"
        
        with open(file_path, "w", encoding="utf-8") as f:
            f.write(report)
        
        print(f"报告已保存: {file_path}")
        return file_path
        main.py
from typing import List
from core.web_scraper import WebScraperAgent
from core.prd_retriever import PRDRetrieverAgent
from core.analyzer import LLMAnalyzerAgent
from core.notifier import NotifierAgent
from config import Config, GapAnalysis

class CompetitorIntelAgent:
    """
    竞品情报与PRD差异检测 Agent
    主控类，协调所有子 Agent 完成工作流
    """
    
    def __init__(self):
        self.scraper = WebScraperAgent()
        self.prd_retriever = PRDRetrieverAgent()
        self.analyzer = LLMAnalyzerAgent()
        self.notifier = NotifierAgent()
    
    def run(self) -> List[GapAnalysis]:
        """执行完整的分析工作流"""
        print("=" * 50)
        print("竞品情报 Agent 开始运行")
        print("=" * 50)
        
      Step 1: 抓取竞品更新
        print("\n[Step 1] 抓取竞品更新...")
        updates = self.scraper.fetch_all_updates()
        
        if not updates:
            print("未发现新的竞品更新")
            return []
        
      Step 2: 对每条更新进行差异分析
        print(f"\n[Step 2] 分析 {len(updates)} 条更新...")
        all_analyses = []
        
        for i, update in enumerate(updates, 1):
            print(f"  分析 [{i}/{len(updates)}]: {update.title[:50]}...")
            
            2.1 检索相关 PRD
            relevant_prds = self.prd_retriever.retrieve_relevant_prds(update)
            
            2.2 LLM 长链推理分析
            analysis = self.analyzer.analyze_update_with_prd(update, relevant_prds)
            all_analyses.append(analysis)
        
      Step 3: 生成报告
        print("\n[Step 3] 生成分析报告...")
        report = self.notifier.generate_markdown_report(all_analyses)
        
      打印摘要到控制台
        uncovered = [a for a in all_analyses if not a.is_covered]
        critical = [a for a in all_analyses if a.gap_severity == "critical"]
        print(f"\n分析完成: 共 {len(all_analyses)} 条更新,")
        print(f"  未覆盖: {len(uncovered)} 条,")
        print(f"  关键差异: {len(critical)} 条")
        
      Step 4: 保存报告
        report_path = self.notifier.save_report(report)
        
      Step 5: 发送通知（如有严重差异）
        if critical or (uncovered and len(uncovered) > 3):
            print("\n[Step 4] 发送钉钉通知...")
            self.notifier.send_to_dingtalk(report)
        
        print("\n" + "=" 50)
        print("Agent 运行完成")
        print(f"完整报告: {report_path}")
        print("=" 50)
        
        return all_analyses
    
    def run_with_mock_data(self) -> List[GapAnalysis]:
        """使用模拟数据运行（用于演示/测试）"""
        print("=" 50)
        print("竞品情报 Agent 运行（演示模式）")
        print("=" 50)
        
        from datetime import datetime
        from core.web_scraper import CompetitorUpdate
        
        模拟竞品更新数据
        mock_updates = [
            CompetitorUpdate(
                source="竞品A",
                update_type="new_feature",
                title="竞品A 发布 AI 智能客服功能",
                content="竞品A 最新版上线了 AI 驱动的智能客服系统，支持 7x24 小时自动回复，可识别用户情感并转接人工...",
                url="https://example.com/release/ai-support",
                published_at=datetime.now().isoformat()
            ),
            CompetitorUpdate(
                source="竞品B",
                update_type="ui_change",
                title="竞品B 移动端首页全新改版",
                content="采用瀑布流布局，优化了加载速度，新增夜间模式支持...",
                url="https://example.com/b/redesign",
                published_at=datetime.now().isoformat()
            ),
            CompetitorUpdate(
                source="竞品A",
                update_type="pricing_change",
                title="竞品A 定价策略调整",
                content="入门版从 $29/月 降至 $19/月，新增企业版 $99/月 支持无限坐席...",
                url="https://example.com/pricing-changes",
                published_at=datetime.now().isoformat()
            )
        ]
        
        执行分析
        all_analyses = []
        for update in mock_updates:
            relevant_prds = self.prd_retriever.retrieve_relevant_prds(update)
            analysis = self.analyzer.analyze_update_with_prd(update, relevant_prds)
            all_analyses.append(analysis)
        
        生成报告
        report = self.notifier.generate_markdown_report(all_analyses)
        self.notifier.save_report(report)
        
        print(f"\n分析完成，处理了 {len(mock_updates)} 条模拟更新")
        return all_analyses



class Scheduler:

    
    @staticmethod
    def run_once():
     
        agent = CompetitorIntelAgent()
        agent.run()
    
    @staticmethod
    def run_daily():
        """每日定时运行（生产环境建议使用 Celery / Airflow / Cron）"""
        import time
        from datetime import datetime
        
        print(f"调度器启动，将每 {Config.CHECK_INTERVAL_HOURS} 小时运行一次")
        
        while True:
            print(f"\n[{datetime.now()}] 开始执行定时任务...")
            
            agent = CompetitorIntelAgent()
            agent.run()
            
            等待下一次执行
            wait_seconds = Config.CHECK_INTERVAL_HOURS * 3600
            print(f"等待 {Config.CHECK_INTERVAL_HOURS} 小时后下次执行...")
            time.sleep(wait_seconds)


入口函数
if name == "main":
    import argparse
    
    parser = argparse.ArgumentParser(description="竞品情报与PRD差异检测 Agent")
    parser.add_argument("--mode", choices=["once", "daily", "mock"], default="once",
                       help="运行模式: once=执行一次, daily=定时任务, mock=演示模式")
    parser.add_argument("--demo", action="store_true",
                       help="使用模拟数据运行（用于测试）")
    
    args = parser.parse_args()
    
    if args.mode == "mock":
        agent = CompetitorIntelAgent()
        agent.run_with_mock_data()
    elif args.mode == "daily":
        Scheduler.run_daily()
    else:  once
        if args.demo:
            agent = CompetitorIntelAgent()
            agent.run_with_mock_data()
        else:
            Scheduler.run_once()
