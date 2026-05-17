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



import time
from datetime import datetime, timedelta
import pandas as pd

class VKParser:
    
    def __init__(self, access_token):
        self.access_token = access_token
        self.vk_api = None
        
        try:
            import vk_api
            self.vk_api = vk_api
            print(" VK API подключён")
        except ImportError:
            print(" Установите vk-api: pip install vk-api")
    
    def search_posts(self, query, days_back=7, limit=10):
        """
        Поиск постов по ключевому слову
        
        Args:
            query: поисковый запрос
            days_back: за сколько дней
            limit: сколько постов
        
        Returns:
            DataFrame с постами
        """
        if not self.vk_api:
            return pd.DataFrame()
        
        # Подключаемся
        vk_session = self.vk_api.VkApi(token=self.access_token)
        vk = vk_session.get_api()
        
        # Дата начала поиска
        start_date = int((datetime.now() - timedelta(days=days_back)).timestamp())
        
        posts = []
        
        try:
            response = vk.wall.search(
                q=query,
                count=limit,
                start_time=start_date,
                v='5.131'
            )
            
            for item in response['items']:
                post = {
                    'id': item['id'],
                    'date': datetime.fromtimestamp(item['date']).strftime('%Y-%m-%d'),
                    'text': item.get('text', '')[:200],
                    'likes': item['likes']['count'],
                    'comments': item['comments']['count'],
                    'reposts': item['reposts']['count'],
                    'views': item.get('views', {}).get('count', 0),
                    'url': f"https://vk.com/wall{item['owner_id']}_{item['id']}"
                }
                posts.append(post)
            
            print(f" Найдено постов: {len(posts)}")
            
        except Exception as e:
            print(f"Ошибка: {e}")
        
        return pd.DataFrame(posts)
    
    def get_post_stats(self, post_id, owner_id):

        if not self.vk_api:
            return {}
        
        vk_session = self.vk_api.VkApi(token=self.access_token)
        vk = vk_session.get_api()
        
        try:
            response = vk.wall.getById(posts=f"{owner_id}_{post_id}")
            if response:
                item = response[0]
                return {
                    'likes': item['likes']['count'],
                    'comments': item['comments']['count'],
                    'reposts': item['reposts']['count'],
                    'views': item.get('views', {}).get('count', 0)
                }
        except:
            pass
        
        return {}
```python
