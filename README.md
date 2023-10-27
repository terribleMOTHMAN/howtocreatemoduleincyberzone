# howtocreatemoduleincyberzone
## Что это?
Здесь будет рассказано как написать модуль для cyberzone от а до я на примере того, что [написал я](https://git.cyberzone.dev/cyberzone-website/backend/-/tree/WEB-12_Add_feedback_model_to_organization)
## Начнем с создания базы edgedb
Важно помнить, что каждый пишет не модуль, а под модуль. Соответственно каждой команде достался модуль, а каждому в команде писать подмодуль этого модуля
В моем случае моя команда писала модуль organizations, а я подмодуль feedback
Начнем...
Сначала заходим в папку dbschema и создаем файл {имя вашего МОДУЛЯ}.esdl
У меня был файл organizations.esdl, его структура...
```
module organizations {
    type Feedback extending default::Auditable{
      required name: str;
      required email: str;
      required description: str;
    }
}
```
в строке `module organizations {` используется название МОДУЛЯ, то есть создается схема вашего модуля, а дальше...
`type Feedback extending default::Auditable{` FeedBack - это подмодуль(таблица базы), `extending default::Auditable` означает, что данная таблица создается на базе другой, у которой название `Auditable`
Дальше пишем поля...
```
      required name: str;
      required email: str;
      required description: str;
```
required - обозначает что поле обязательное(если хотите чтобы поле было необязательным не пишите required и все), дальше название поля, и через двоеточие пишем тип поля
Отлично мы создали базу модуля и таблицу подмодуля.
Остается создать миграцию и все!
```cli
edgedb migration create
edgedb migrate
```
Profit!
## Изучим структуру нашего подмодуля
Заходим в папку cyberzone и создаем папку с названием нашего модуля
В моем случае это organizations
Дальше создаем нужные файлы:
1) __init__.py - инициализирует папку как модуль для работы
2) endpoints.py - хранит ссылки нашего api
3) schema.py - хранит схемы нашего модуля
4) table_data_getaways - важная папка, о которой поговорим позже
![image](https://github.com/terribleMOTHMAN/howtocreatemoduleincyberzone/assets/65505901/072bcfc1-b390-4ddc-9aee-985b5f70de58)

Создали? Красавчики! Начнем кодить
Начинаем со схем, оринтируясь на созданую таблицу
```py
import uuid
from pydantic import BaseModel


class Feedback(BaseModel):
    id: uuid.UUID
    name: str
    email: str
    description: str

class FeedbackCreate(BaseModel):
    name: str
    email: str
    description: str
```
Что у вас здесь вообще происходит? Надеюсь вы знаете ооп так что начнем
Создаем класс Feedback, который наследует готовую модель pydantic под названием BaseModel
Дальше объявляем поля думаю все понятно
Второй класс используется для создания request, потому что в функцию create мы не передаем uuid. Edgedb сделает все за нас :3 

Теперь перейдем в папку table_data_getaways и создаем такую структуру:
1) __init__.py
2) organizations.py
Начнем кодить!
Щас будет биговый и страшный код. НЕ БОИМСЯ У НАС БОЛЬШЕ!
```py
import uuid
import edgedb.errors

from cyberzone.core.utils import make_pydantic_model
from cyberzone.core.databases.table_data_gateways import BaseTableDataGateway
from cyberzone.organizations.schemas import Feedback, FeedbackCreate


class FeedbackTBG(BaseTableDataGateway):
    """
    Feedback TBG
    """

    async def get(self,
                  name: str | None = None,
                  email: str | None = None,
                  description: str | None = None,
                  limit: int | None = 20,
                  offset: int = 0
                  ) -> list[Feedback] | None:
        """
        Getting all feedback models with the ability to filter by all fields
        """
        query = f"""
        with
            name := <optional str>$name
            email := <optional str>$email
            description := <optional str>$description
        select organizations::Feedback {{ * }} 
            filter (.name = name or name ?= <optional str>{{}})
            and
            filter (.email = email or email ?= <optional str>{{}})
            and
            filter (.description = description or description ?= <optional str>{{}})
        offset <int64>$offset {'limit <int64>$limit' if limit else ''}
        """
        return await self.database.query(query, name=name, email=email, description=description, limit=limit,
                                         offset=offset)

    async def get_by_uuid(self, feedback_uuid: uuid.UUID) -> Feedback | None:
        query = """
                select organizations::Feedback { * } filter .id = <uuid>$feedback_uuid
                """
        if result := await self.database.query(query, feedback_uuid=feedback_uuid):
            return make_pydantic_model(Feedback, result[0])
        return None

    async def create(self, feedback: FeedbackCreate) -> Feedback | None:
        query = """
                with 
                    data := <tuple<
                    name: str,
                    email: str,
                    description: str,
                    >> $organizations
                select (insert organizations::Feedback {
                    name := data.name,
                    email := data.email,
                    description := data.description
                }
                ) {*}
                """

        data = tuple(value for value in feedback.__dict__.values())
        try:
            created_board_game = (await self.database.query(query, feedback=data))[0]
        except edgedb.errors.EdgeDBError:
            return None
        return make_pydantic_model(Feedback, created_board_game)
```
Пройдемся по коду! УХ, БУДЕТ ПОТНО ЛЕССС ГОООУ
Мы создаем класс с {название таблицы}TBG и наследуем от базовой модели написанной уже до нас...
Дальше пишем методы класса не забывая про оформление!!!
Рассмотрим метод get, который возвращает знаения из таблицы
Переменная query содержит в себе строковый запрос в бд
Кто знает sql хотя бы на базовом уровне, то все понятно, остальным заболезную...
Дальше возвращаем из функции результат этого запроса

Дальше метод get_by_uuid
Он возвращает ОДНУ строку из таблицы
Проверим если вообще что то вернулось после нашего запроса
Если мы получили данные возвращаем их, если нет, то возвращаем None

Дальше create 
Берем значения из переданных данных. Создаем из них кортеж
Дальше пробуем создать модель подобную edgedb, если все работает, то из функции возвращается созданная строка для таблицы, если не, то None

Выходим из папки
Открываем endpoints.py
```py
import uuid

from fastapi.routing import APIRouter
from fastapi import Depends

from cyberzone.organizations.table_data_gateways.organizations import FeedbackTBG
from cyberzone.organizations.schemas import Feedback, FeedbackCreate

from cyberzone.auth.schemas import User
from cyberzone.auth.dependencies import AccessControl

router = APIRouter(prefix="/feedback")


@router.get("/", response_model=list[Feedback])
async def get_feedback(
        name: str | None = None,
        email: str | None = None,
        description: str | None = None,
        limit: int | None = 20,
        offset: int = 0,
        user: User = Depends(AccessControl()),
) -> list[Feedback]:
    return await FeedbackTBG(user.id).get(
        name=name, email=email, description=description, limit=limit, offset=offset
    )


@router.get("/{feedback_id}", response_model=Feedback)
async def get_feedback(
        feedback_uuid: uuid.UUID,
        user: User = Depends(AccessControl())  # noqa,
) -> Feedback:
    return await FeedbackTBG(user.id).get_by_uuid(feedback_uuid=feedback_uuid)


@router.post("/", response_model=Feedback)
async def create_feedback(
        feedback: FeedbackCreate,
        user: User = Depends(AccessControl("organization_insert"))  # noqa,
) -> Feedback:
    return await FeedbackTBG(user.id).create(feedback)
```

Создаем роутер с префиксом нашего подмодуля :3 мы молодцы

Дальше для api переписываем все наши методы из класса, который писали ранее
@router.get
Кидаем запрос в метод get класса и возвращаем его

@router.get
Кидаем запрос в метод get_by_uuid класса и возвращаем его

@router.post
Рассмотрим строку `user: User = Depends(AccessControl("organization_insert"))`
Тут проверяется разрешение: может ли человек делающий запрос вообще создавать что либо
А дальше все по стандарту

Выходим в cyberzone. Затем заходим в core, потом в router. Смотрим...
Так выглядить должно у вас:
```py

from fastapi.routing import APIRouter

from cyberzone.core.config import settings

from cyberzone.core.endpoints import router as system_router
from cyberzone.auth.endpoints import router as auth_router
from cyberzone.booking.endpoints import router as booking_router
from cyberzone.device.router import router as device_router
from cyberzone.board_game.endpoints import router as board_game_router

router = APIRouter(prefix=settings.API_V1_STR)
router.include_router(system_router, prefix="/system", tags=["System"])
router.include_router(auth_router, prefix="/auth", tags=["Auth"])
router.include_router(booking_router, prefix="/booking", tags=["Booking"])
router.include_router(device_router, prefix="/device", tags=["Device"])
router.include_router(board_game_router, prefix="/board_game", tags=["Board Game"])
```
Так выглядит у меня:
```py

from fastapi.routing import APIRouter

from cyberzone.core.config import settings

from cyberzone.core.endpoints import router as system_router
from cyberzone.auth.endpoints import router as auth_router
from cyberzone.booking.endpoints import router as booking_router
from cyberzone.device.router import router as device_router
from cyberzone.board_game.endpoints import router as board_game_router
from cyberzone.organizations.endpoints import router as organizations_router

router = APIRouter(prefix=settings.API_V1_STR)
router.include_router(system_router, prefix="/system", tags=["System"])
router.include_router(auth_router, prefix="/auth", tags=["Auth"])
router.include_router(booking_router, prefix="/booking", tags=["Booking"])
router.include_router(device_router, prefix="/device", tags=["Device"])
router.include_router(board_game_router, prefix="/board_game", tags=["Board Game"])
router.include_router(organizations_router, prefix="/organizations", tags=["Organizations"])
```
Найдите 10 отличий :3 
Не нашли? Давай те помогу
`from cyberzone.organizations.endpoints import router as organizations_router` - импортируем наш роутер, который мы написали
`router.include_router(organizations_router, prefix="/organizations", tags=["Organizations"])` - добавляем его в основной (ГЛАВНЫЙ) роутер

Вуаля! Вы написали модуль! Вопросы? Пишите в тг: @terribleMOTHMAN
![image](https://github.com/terribleMOTHMAN/howtocreatemoduleincyberzone/assets/65505901/1822817c-2c52-4333-b523-f9f5c8b1b148)
