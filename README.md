# Сравнение активности аудитории в соцсетях 



# Состав группы
Тагаева Алина


# Идея компании
Разрабатываем сервис для SMM-специалистов и малого бизнеса, который автоматически сравнивает вовлечённость аудитории в популярных соцсетях: VK, Telegram, YouTube на основе одних и тех же тем

Продаем подписку на аналитический дашборд, который показывает:
- Уровень вовлеченности аудитории по каждой соцсети
- Среднее число лайков/комментариев/репостов
- Рекомендации: куда выгоднее постить в первую очередь

Кому:
- SMM-менеджеры и маркетинговые агентства 
- Владельцы небольших брендов и медиа 
- Блогеры и инфлюенсеры

# Импорт библиотек
```python
import asyncio
import json
import os
import random
from abc import ABC, abstractmethod
from collections import defaultdict
from dataclasses import dataclass
from datetime import datetime, timedelta

try:
    import httpx
    HAS_HTTPX = True
except ImportError:
    HAS_HTTPX = False

VK_TOKEN = os.getenv("VK_ACCESS_TOKEN", "")
VK_GROUP = os.getenv("VK_GROUP_ID", "")
TG_TOKEN = os.getenv("TG_BOT_TOKEN", "")
TG_CHANNEL = os.getenv("TG_CHANNEL_ID", "")
YT_KEY = os.getenv("YT_API_KEY", "")
YT_CHANNEL = os.getenv("YT_CHANNEL_ID", "")
```

# Модели данных

```python
@dataclass
class Post:
    id: str
    platform: str
    text: str
    topic: str
    published_at: datetime
    likes: int = 0
    comments: int = 0
    shares: int = 0
    views: int = 0
    url: str = ""

@dataclass
class EngagementMetrics:
    platform: str
    topic: str
    total_posts: int = 0
    total_likes: int = 0
    total_comments: int = 0
    total_shares: int = 0
    total_views: int = 0
    avg_likes: float = 0.0
    avg_comments: float = 0.0
    avg_shares: float = 0.0
    avg_views: float = 0.0
    engagement_rate: float = 0.0

    @property
    def avg_engagement_per_post(self) -> float:
        return self.avg_likes + self.avg_comments + self.avg_shares

    def to_dict(self) -> dict:
        return {
            "platform": self.platform,
            "topic": self.topic,
            "total_posts": self.total_posts,
            "total_likes": self.total_likes,
            "total_comments": self.total_comments,
            "total_shares": self.total_shares,
            "total_views": self.total_views,
            "avg_likes": round(self.avg_likes, 1),
            "avg_comments": round(self.avg_comments, 1),
            "avg_shares": round(self.avg_shares, 1),
            "avg_views": round(self.avg_views, 1),
            "engagement_rate": round(self.engagement_rate, 2),
            "avg_engagement_per_post": round(self.avg_engagement_per_post, 1),
        }
```
# Парсеры
```python
class BaseParser(ABC):
    platform: str

    @abstractmethod
    async def fetch_posts(self, topic: str, limit: int = 10) -> list[Post]:
        ...

def _mock_posts(topic: str, limit: int, platform: str) -> list[Post]:
    posts = []
    for i in range(limit):
        posts.append(Post(
            id=f"{platform.lower()}_{topic[:5]}_{i}",
            platform=platform,
            text=f"Mock post about {topic} on {platform} #{i}",
            topic=topic,
            published_at=datetime.now() - timedelta(hours=i),
            likes=random.randint(5, 500),
            comments=random.randint(0, 100),
            shares=random.randint(0, 80),
            views=random.randint(100, 10000),
        ))
    return posts


```
# Парсинг VK

```python
class VKParser(BaseParser):
    platform = "VK"

    async def fetch_posts(self, topic: str, limit: int = 10) -> list[Post]:
        if not VK_TOKEN or not HAS_HTTPX:
            return _mock_posts(topic, limit, self.platform)
        async with httpx.AsyncClient() as client:
            params = {"access_token": VK_TOKEN, "v": "5.199", "query": topic, "count": limit}
            if VK_GROUP:
                params["owner_id"] = f"-{VK_GROUP}"
            resp = await client.post("https://api.vk.com/method/wall.search", params=params)
            data = resp.json()
        posts = []
        for item in data.get("response", {}).get("items", []):
            posts.append(Post(
                id=str(item.get("id")), platform=self.platform,
                text=str(item.get("text", ""))[:200], topic=topic,
                published_at=datetime.fromtimestamp(item.get("date", 0)),
                likes=item.get("likes", {}).get("count", 0),
                comments=item.get("comments", {}).get("count", 0),
                shares=item.get("reposts", {}).get("count", 0),
                views=item.get("views", {}).get("count", 0),
                url=f"https://vk.com/wall{item.get('owner_id')}_{item.get('id')}",
            ))
        return posts or _mock_posts(topic, limit, self.platform)
```
# Парсинг telegram
```python
class TelegramParser(BaseParser):
    platform = "Telegram"

    async def fetch_posts(self, topic: str, limit: int = 10) -> list[Post]:
        if not TG_TOKEN or not HAS_HTTPX:
            return _mock_posts(topic, limit, self.platform)
        async with httpx.AsyncClient() as client:
            resp = await client.get(
                f"https://api.telegram.org/bot{TG_TOKEN}/getUpdates", params={"timeout": 0}
            )
            data = resp.json()
        posts = []
        for update in data.get("result", []):
            msg = update.get("channel_post") or update.get("message", {})
            text = msg.get("text") or msg.get("caption", "")
            if topic.lower() not in (text or "").lower():
                continue
            posts.append(Post(
                id=str(update.get("update_id")), platform=self.platform,
                text=(text or "")[:200], topic=topic,
                published_at=datetime.fromtimestamp(msg.get("date", 0)),
                views=msg.get("views", 0),
                url=f"https://t.me/{TG_CHANNEL.strip('@')}/{msg.get('message_id')}" if TG_CHANNEL else "",
            ))
            if len(posts) >= limit:
                break
        return posts or _mock_posts(topic, limit, self.platform)
```

# Парсинг youtube
```python
class YouTubeParser(BaseParser):
    platform = "YouTube"

    async def fetch_posts(self, topic: str, limit: int = 10) -> list[Post]:
        if not YT_KEY or not HAS_HTTPX:
            return _mock_posts(topic, limit, self.platform)
        async with httpx.AsyncClient() as client:
            params = {"part": "snippet", "q": topic, "type": "video",
                       "maxResults": limit, "key": YT_KEY}
            if YT_CHANNEL:
                params["channelId"] = YT_CHANNEL
            search = await client.get("https://www.googleapis.com/youtube/v3/search", params=params)
            video_ids = [item["id"]["videoId"] for item in search.json().get("items", [])]
            if not video_ids:
                return _mock_posts(topic, limit, self.platform)
            stats = await client.get(
                "https://www.googleapis.com/youtube/v3/videos",
                params={"part": "statistics,snippet", "id": ",".join(video_ids), "key": YT_KEY},
            )
            items = stats.json().get("items", [])
        posts = []
        for item in items:
            snippet = item.get("snippet", {})
            stat = item.get("statistics", {})
            posts.append(Post(
                id=item.get("id", ""), platform=self.platform,
                text=snippet.get("title", "")[:200], topic=topic,
                published_at=datetime.fromisoformat(
                    snippet.get("publishedAt", "2024-01-01T00:00:00Z").replace("Z", "+00:00")
                ),
                likes=int(stat.get("likeCount", 0)),
                comments=int(stat.get("commentCount", 0)),
                views=int(stat.get("viewCount", 0)),
                url=f"https://youtu.be/{item.get('id', '')}",
            ))
        return posts
```
