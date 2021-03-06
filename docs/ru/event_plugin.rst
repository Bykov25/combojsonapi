EventPlugin (`EN`_ | `RU`_)
---------------------------

**EventPlugin** позволяет:

1. Создать событийное API (RPC).
2. Интегрируется с плагином **ApiSpecPlugin** позволяя создавать документацию для RPC и отображать
   вместе с общей документацией. Описание view производится при помощи **yaml**.
3. Интегрируется с плагином **PermissionPlugin**, можно из view получить доступ к ограничения по
   любой модели. Также доступ к view ограничивается общими декораторами, которые задаются при
   инициализации API.

Работа с плагином
~~~~~~~~~~~~~~~~~
Чтобы создать RPC API связанного с какой-либо моделью, нужно:

1. Создать класс от :code:`combojsonapi.event.resource.EventsResource` ниже будет более
   подробно сказано об этом.
2. В ресурс менеджере появляется атриббут :code:`events` в нём нужно указать класс созданный в
   первом пункте. Если укажите в ресурс менеджере :code:`ResourceDetail`, то в каждую view RPC API
   будет приходить также id модели (которая указана в ресурс менеджере).

Описание работы плагина
"""""""""""""""""""""""

После того как создали класс, унаследованный от :code:`combojsonapi.event.resource.EventsResource`,
любой метод в этом классе, который начинается с :code:`event_` будет считаться самостоятельным view.
Адрес нового view будет формироваться в формате :code:`.../<url ресурс менеджера, к которому привязан
данный класс с методами RPC API>/event_<название нашего метода, после event_>`.
По умолчанию создаются POST ресурсы. Но можно создать GET ресурс, назвав метод :code:`event_get_something`.
Также поддерживается :code:`event_post_something` - создастся POST ресурс.

**Другие методы и атрибуты Event класса не будут видны внутри view.**

Описания view
"""""""""""""

1. Метод :code:`event[_опционально тип запроса]_<название метода>` должен принимать следующие параметры:
    * :code:`id: int` [опционально] - id сущности модели. Если класс с данным view указан в ресурс менеджере
      :code:`ResourceDetail`.
    * :code:`_permission_user: PermissionUser = None` - пермишены для данного авторизованного
      пользователя (при условии, что подключен плагин **PermissionPlugin**)
    * :code:`*args`
    * :code:`**kwargs`
2. При описании ответов view используйте формат JSONAPI
3. В начале метода нужно описать документацию к view на yaml, чтобы хорошо прошла интеграция с
   плагином автодокументации **ApiSpecPlugin**


Пример подключения плагина
~~~~~~~~~~~~~~~~~~~~~~~~~~

Рассмотрим пример, когда нам нужно загрузить аватарку для пользователя. В примере также подключим
плагин **ApiSpecPlugin**, чтобы посмотреть его в действии

.. code:: python

    import os
    from flask import Flask, request
    from flask_sqlalchemy import SQLAlchemy
    from sqlalchemy import Column, Integer, String
    from sqlalchemy.orm import Query, load_only, scoped_session
    from flask_combo_jsonapi.marshmallow_fields import Relationship
    from flask_combo_jsonapi import Api, ResourceList, ResourceDetail
    from flask_combo_jsonapi.plugin import BasePlugin
    from flask_combo_jsonapi.querystring import QueryStringManager
    from combojsonapi.event.resource import EventsResource
    from combojsonapi.event import EventPlugin
    from combojsonapi.spec import ApiSpecPlugin
    from marshmallow_jsonapi.flask import Schema
    from marshmallow_jsonapi import fields


    app = Flask(__name__)
    app.config['SQLALCHEMY_DATABASE_URI'] = 'sqlite:///:memory:'
    app.config['SQLALCHEMY_ECHO'] = True
    db = SQLAlchemy(app)

    """Описание моделей"""

    class User(db.Model):
        __tablename__ = 'users'
        id = Column(Integer, primary_key=True)
        name = Column(String)
        fullname = Column(String)
        email = Column(String)
        url_avatar = Column(String)
        password = Column(String)


    db.create_all()

    """Описание схем моделей"""

    class UserSchema(Schema):
        class Meta:
            type_ = 'user'
            self_view = 'user_detail'
            self_view_kwargs = {'id': '<id>'}
            self_view_many = 'user_list'
            ordered = True
        id = fields.Integer(as_string=True)
        name = fields.String()
        fullname = fields.String()
        email = fields.String()
        url_avatar = fields.String()
        password = fields.String()

    """Описание ресурс менеджеров для API"""

    class UserResourceDetailEvents(EventsResource):
        def event_update_avatar(self, *args, id: int = None, **kwargs):
            # language=YAML
            """
            ---
            summary: Обновление аватарки пользователя
            tags:
            - User
            parameters:
            - in: path
              name: id
              required: True
              type: integer
              format: int32
              description: 'id пользователя'
            - in: formData
              name: new_avatar
              type: file
              description: Новая аватарка пользователя
            consumes:
            - application/json
            responses:
              200:
                description: Ничего не вернёт
            """
            user = User.query.filter(User.id == id).one_or_none()
            if user is None:
                raise AccessDenied('You can not work with the user')

            avatar = request.files.get('new_avatar')
            if avatar:
                if avatar:
                    filename = avatar.filename
                    avatar.save(os.path.join(filename))
                user.url_avatar = os.path.join(filename)
                db.session.commit()
            return 'success', 201

        def event_get_info(self, *args, **kwargs):
            return {'message': 'GET INFO'}

        def event_post_info(self, *args, **kwargs):
            data = request.json
            data.update(message='POST INFO')
            return data

    class UserResourceDetail(ResourceDetail):
        schema = UserSchema
        events = UserResourceDetailEvents
        methods = ['GET']
        data_layer = {
            'session': db.session,
            'model': User,
        }

    class UserResourceList(ResourceList):
        schema = UserSchema
        methods = ['GET', 'POST']
        data_layer = {
            'session': db.session,
            'model': User,
        }

    """Инициализация API"""

    app.config['OPENAPI_URL_PREFIX'] = '/api/swagger'
    app.config['OPENAPI_SWAGGER_UI_PATH'] = '/'
    app.config['OPENAPI_SWAGGER_UI_VERSION'] = '3.22.0'

    api_spec_plugin = ApiSpecPlugin(
        app=app,
        # Объявляем список тегов и описаний для группировки api в группы (api можно не группировать в группы,
        # в этом случае они будут группирваться автоматически по названию типов схем (type_))
        tags={
            'User': 'API для user'
        }
    )

    api_json = Api(
        app,
        plugins=[
            api_spec_plugin,
            EventPlugin()
        ]
    )
    api_json.route(UserResourceDetail, 'user_detail', '/api/user/<int:id>/', tag='User')
    api_json.route(UserResourceList, 'user_list', '/api/user/', tag='User')


    if __name__ == '__main__':
        for i in range(10):
            u = User(name=f'name{i}', fullname=f'fullname{i}', email=f'email{i}', password=f'password{i}')
            db.session.add(u)
        db.session.commit()
        app.run(port='9999')

.. _`EN`: https://github.com/AdCombo/combojsonapi/blob/master/docs/en/event_plugin.rst
.. _`RU`: https://github.com/AdCombo/combojsonapi/blob/master/docs/ru/event_plugin.rst