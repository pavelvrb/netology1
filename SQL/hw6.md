<pre>import os
import logging

import psycopg2
import psycopg2.extensions
from pymongo import MongoClient
import pandas as pd
from sqlalchemy import create_engine
from sqlalchemy import Table, Column, Integer, Float, MetaData, String
from sqlalchemy.orm import sessionmaker
from sqlalchemy.ext.declarative import declarative_base


logging.basicConfig(format='%(asctime)s : %(levelname)s : %(message)s', level=logging.INFO)
logger = logging.getLogger(__name__)

# Задание по Psycopg2
# --------------------------------------------------------------

logger.info("Создаём подключёние к Postgres")
params = {
    "host": os.environ['APP_POSTGRES_HOST'],
    "port": os.environ['APP_POSTGRES_PORT'],
    "user": 'postgres'
}
conn = psycopg2.connect(**params)

# дополнительные настройки
psycopg2.extensions.register_type(
    psycopg2.extensions.UNICODE,
    conn
)
conn.set_isolation_level(
    psycopg2.extensions.ISOLATION_LEVEL_AUTOCOMMIT
)
cursor = conn.cursor()


user_item_query_config = {
    "MIN_AVG_RATING": 3
}
sql_str = (
        """
            SELECT * INTO movies_top
FROM
    (SELECT movieId, COUNT(rating) as ratings_num, AVG(rating) as ratings_avg
    FROM ratings
    GROUP BY movieId) as ratings_set
WHERE ratings_avg > %(MIN_AVG_RATING)d;
        """ % user_item_query_config
)


cursor.execute(sql_str)
conn.commit()

# Проверка - выгружаем данные
cursor.execute("SELECT * FROM movies_top LIMIT 10")
logger.info(
    "Выгружаем данные из таблицы movies_top: (movieId, ratings_num, ratings_avg)\n{}".format(
        [i for i in cursor.fetchall()])
)


# Задание по SQLAlchemy
# --------------------------------------------------------------
Base = declarative_base()


class MoviesTop(Base):
    __tablename__ = 'movies_top'

    movieid = Column(Integer, primary_key=True)
    ratings_num = Column(Float)
    ratings_avg = Column(Float)

    def __repr__(self):
        return "<User(movieid='%s', ratings_num='%s', ratings_avg='%s')>" % (self.movieid, self.ratings_num, self.ratings_avg)


# Создаём сессию

engine = create_engine('postgresql://postgres:@{}:{}'.format(os.environ['APP_POSTGRES_HOST'], os.environ['APP_POSTGRES_PORT']))
Session = sessionmaker(bind=engine)
session = Session()


# --------------------------------------------------------------
# Ваш код здесь
# выберите контент у которого больше 15 оценок (используйте filter)
# и средний рейтинг больше 3.5 (filter ещё раз)
# отсортированный по среднему рейтингу (используйте order_by())
# id такого контента нужно сохранить в массив top_rated_content_ids

top_rated_query = session.query(MoviesTop)

logger.info("Выборка из top_rated_query\n{}".format([i for i in top_rated_query.limit(4)]))

top_rated_content_ids = [
    i[0] for i in top_rated_query.values(MoviesTop.movieid)
][:5]
# --------------------------------------------------------------

# Задание по PyMongo
mongo = MongoClient(**{
    'host': os.environ['APP_MONGO_HOST'],
    'port': int(os.environ['APP_MONGO_PORT'])
})

# Получите доступ к коллекции tags
db = mongo["movie"]
tags_collection = db['tags']

# id контента используйте для фильтрации - передайте его в модификатор $in внутри find
# в выборку должны попать теги фильмов из массива top_rated_content_ids
mongo_query = tags_collection.find(
        {'id': {}}
)
mongo_docs = [
    i for i in mongo_query
]

print("Достали документы из Mongo: {}".format(mongo_docs[:5]))

id_tags = [(i['id'], i['name']) for i in mongo_docs]


</pre>
