%Marshmallow
%De la sérialization à la construction d'une API REST

## Sommaire

- La sérialisation
- marshmallow
- L'écosystème marshmallow
- Construction d'une API REST: flask-smorest

# La sérialisation

## C'est quoi?

Codage d'objets Python sous une forme adaptée pour

- Sauvegarde
- Transport

Processus réversible (désérialisation)

## Pickle

Objet → _Byte stream_ → Objet

- Avantages
    - Bibliothèque standard
    - Rapide

- Inconvénients
    - Compatibilité : désérialisation nécessite Python
    - Sécurité : injection de code

Utilisation typique : stockage temporaire

## json (1)

Objet → JSON → Objet

```python
import json

team = {'name': 'A-Team'}

json.dumps()
# '{"name": "A-Team"}'

json.loads('{"name": "A-Team"}')
# {'name': 'A-Team'}
```

## json (2)

JSON ne définit que des types basiques.

```python
import json
import datetime as dt

team = {'name': 'A-Team', 'creation_date': dt.datetime(1983, 1, 23)}

json.dumps()
# TypeError: datetime.datetime(1983, 1, 23, 0, 0) is not JSON serializable
```

## json (3)

- Avantages
    - Standard / Inter-opérable
    - Lisible

- Inconvénients
    - Ne représente que certains types Python de base

Note : Plus ou moins équivalent à YAML, XML.

## Bibliothèque de sérialisation (1)

Transforme un objet Python en dictionnaire de types simples, JSONisable.

Objet → _dict_ → Objet

Surcouche de json

Objet → _dict_ → JSON → _dict_ → Objet

## Bibliothèque de sérialisation (2)

- Avantages
    - Standard / Inter-opérable
    - Lisible
    - Pas limité aux types simples

- Inconvénients
    - Nécessite de définir la sérialisation des objets non standards
    - Bibliothèque non standard

## Bibliothèque de sérialisation (3)

Utilisations typiques :

- Format d'échange (ex: service web)
- Fichier de configuration
- Sauvegarde de résultats de calcul
- ...

# marshmallow


## Fonctionnalités

- Sérialisation vers _dict_ ou JSON
- Désérialisation depuis _dict_ ou JSON
- Validation lors de la désérialisation

## Schémas et champs

```python
import datetime as dt
import marshmallow as ma

class TeamSchema(ma.Schema):
    name = ma.fields.String()
    creation_date = ma.fields.DateTime()

schema = TeamSchema()

team = {'name': 'A-Team', 'creation_date': dt.datetime(1983, 1, 23)}

schema.dump(team)
# {'name': 'A-Team', 'creation_date': '1983-01-23T00:00:00'}

schema.dumps(team)
# '{"name": "A-Team", "creation_date": "1983-01-23T00:00:00"}'
```

## Séparation modèle / vue


```python
# Modèle

import orm

class Team(orm.Model):
    name = orm.StringField()
    creation_date = orm.DateTiemField()

# Schéma

import datetime as dt
import marshmallow as ma

class TeamSchema(ma.Schema):
    name = ma.fields.String()
    creation_date = ma.fields.DateTime()

# Vue

from .models import Team
from .schemas import TeamSchema

team = Team.get(name='A-Team')

schema = TeamSchema()

schema.dump(team)
# {'name': 'A-Team', 'creation_date': '1983-01-23T00:00:00'}
```

## Champs imbriqués

```python

class MemberSchema(ma.Schema):
    first_name = ma.fields.String()
    last_name = ma.fields.String()
    birthdate = ma.fields.DateTime()

class TeamSchema(ma.Schema):
    name = ma.fields.String()
    members = ma.fields.List(ma.fields.Nested(MemberSchema))

team = {
    'name': 'Ghostbusters',
    'members': [
        {'first_name': "Egon", 'last_name': "Spengler", 'birthdate': dt.datetime(1958, 10, 2)},
        {'first_name': "Peter", 'last_name': "Venkman", 'birthdate': dt.datetime(1960, 9, 6)},
    ]
}

TeamSchema().dumps(team)
# {'name': 'Ghostbusters',
#  'members': [
#   {'first_name': 'Egon', 'last_name': 'Spengler', 'birthdate': '1958-10-02T00:00:00'},
#   {'first_name': 'Peter', 'last_name': 'Venkman', 'birthdate': '1960-09-06T00:00:00'}
# ]}
```



## TODO

- dump_only, load_only
- only, exclude
- data_key
- attribute
- collections: many

- post/pre load/dump
- loading to object

- validation:
    required
    length,range,...
    errors dict structure

## Questions


# Intégration ORM/ODM

## Intégration avec différents ORM/ODM

ORM : Object-Relation Mapping

ODM : Object-Document Mapping

Génération automatique de schémas marshmallow depuis le modèle

Types et validateurs inférés des classes du modèle

Permet de générer des schémas d'API en minimisant la duplication de code

- SQLAlchemy → marshmallow-sqlalchemy
- peewee → marshmallow-peewee
- MongoEngine → marshmallow-mongoengine

## marshmallow-mongoengine

### Modèle

```python
import mongoengine as me

class Team(me.Document):
    name = me.StringField(max_length=40)
    members = me.ListField(me.ReferenceField('Member'))

class Member(me.Document):
    first_name = me.StringField()
    last_name = me.StringField()
    birthdate = me.DateTimeField()
```

---------------------------------------------------

### Schémas

```python
from marshmallow_mongoengine import ModelSchema

class TeamSchema(ModelSchema):
    class Meta:
        model = Team

class MemberSchema(ModelSchema):
    class Meta:
        model = Member

team = Team.objects.get(name='Ghostbusters')

TeamSchema().dump(team)
# {'id': 1,
#   'name': 'Ghostbusters',
#  'members': [
#   {'first_name': 'Egon', 'last_name': 'Spengler', 'birthdate': '1958-10-02T00:00:00'},
#   {'first_name': 'Peter', 'last_name': 'Venkman', 'birthdate': '1960-09-06T00:00:00'}
# ]}

TeamSchema().load({'name': 'This name is too long to pass validation.'})
# marshmallow.exceptions.ValidationError: {'name': ['Longer than maximum length 40.']}
```

## umongo : ODM MongoDB

Alternative à MongoEngine + marshmallow-mongoengine

Utilise marshmallow pour serialization/désérialisation MongoDB BSON

Génère schemas marshmallow pour API

Fonctionne avec Pymogo, TxMongo, motor_asyncio

# webargs

## webargs : désérialisation de requêtes

Désérialise et valide les requêtes HTTP

Prend en charge nativement les principaux serveurs web :

Flask, Django, Bottle, Tornado, Pyramid, webapp2, Falcon, aiohttp

## Sans webargs

```python
from flask import Flask, request

app = Flask(__name__)

team = TeamSchema()

@app.route("/teams/")
def index():
    # Désérialisation et validation
    body = request.json
    try:
        team_data = team_schema.load(body)
    except ValidationError as exc:
        abort(422)
    # Traitement
    team = Team(**team_data)
    team.save()
    return team_schema.dump(team), 201
```

---------------------------------------------------

## Avec webargs (1)

```python
from flask import Flask
from webargs.flaskparser import use_args

app = Flask(__name__)

@app.route("/")
@use_args(TeamSchema, location='json')
def index(team_data):
    team = Team(**team_data)
    team.save()
    return team_schema.dump(team), 201
```

---------------------------------------------------

## Avec webargs (2)

Inclure les erreurs de validation dans la réponse

```python
from flask import jsonify

# Return validation errors as JSON
@app.errorhandler(422)
def handle_error(err):
    messages = err.data.get("messages", ["Invalid request."])
    return jsonify({"errors": messages}), err.code
```

# apispec

## apispec : Documentation OpenAPI (Swagger)

Génération de la documentation OpenAPI

Introspection des schémas marshmallow

Prise en charge de flask, bottle, tornado via apispec-webframeworks

## webargs + apispec : exemple

```python
from flask import Flask, request
from marshmallow import Schema, fields
from webargs.flaskparser import use_args
from apispec import APISpec
from apispec.ext.marshmallow import MarshmallowPlugin
from apispec_webframeworks.flask import FlaskPlugin

spec = APISpec(
    title="Team manager",
    version="1.0.0",
    openapi_version="3.0.2",
    plugins=[FlaskPlugin(), MarshmallowPlugin()],
)

app = Flask(__name__)
```
---------------------------------------------------

```python
@app.route("/teams/", methods=["POST"])
@use_args(TeamSchema, location='json')
def post_team():
    """Post team
    ---
    post:
      description: Add a new team.
      requestBody:
        description: Team
        required: true
        content:
          application/json:
            schema: TeamSchema
      responses:
        200:
          content:
            application/json:
              schema: TeamSchema
    """
def index(team_data):
    team = Team(**team_data)
    team.save()
    return team_schema.dump(team), 201

spec.path(view=post_team)
```

## webargs + apispec : limitations

- Duplication : YAML
- Sérialisation manuelle

# flask-smorest

## flask-smorest : marshmallow + webargs + apispec

- Injection de paramètres avec webargs
- Doc auto avec apispec, sans YAML
- MethodView + Blueprint
- ETag
- Pagination

## Structuration d'une ressource

- ``Blueprint`` → ressource
- ``MethodView`` → GET, POST, PUT, DELETE

---------------------------------------------------

```python
@blp.route('/')
class Teams(MethodView):

    @blp.arguments(TeamQueryArgsSchema, location='query')
    @blp.response(TeamSchema(many=True))
    def get(self, args):
        """List teams"""
        return Team.query.filter_by(**args)

    @blp.arguments(TeamSchema)
    @blp.response(TeamSchema, code=201)
    def post(self, new_item):
        """Add a new team"""
        item = Team(**new_item)
        db.session.add(item)
        db.session.commit()
        return item
```

---------------------------------------------------

```python

@blp.route('/<uuid:item_id>')
class TeamsById(MethodView):

    @blp.response(TeamSchema)
    def get(self, item_id):
        """Get team by ID"""
        return Team.query.get_or_404(item_id)

    @blp.arguments(TeamSchema)
    @blp.response(TeamSchema)
    def put(self, new_item, item_id):
        """Update an existing team"""
        item = Team.query.get_or_404(item_id)
        TeamSchema().update(item, new_item)
        db.session.add(item)
        db.session.commit()
        return item

    @blp.response(code=204)
    def delete(self, item_id):
        """Delete a team"""
        item = Team.query.get_or_404(item_id)
        db.session.delete(item)
        db.session.commit()
```

## Pagination

- Pagination `a posteriori` des resources renvoyant une liste
- Validation des paramètres d'entrée ``page`` et ``page_size`` (query args)
- Éléments de pagination renvoyés dans un Header

```python
headers['X-Pagination']
{
    'total': 1000, 'total_pages': 200,
    'page': 2, 'first_page': 1, 'last_page': 200,
    'previous_page': 1, 'next_page': 3,
}
```

## Pagination d'un curseur de base de donnée

```python
from flask_smorest import Page

class SQLCursorPage(Page):
    @property
    def item_count(self):
        return self.collection.count()


@blp.route('/')
class Teams(MethodView):

    @blp.arguments(TeamQueryArgsSchema, location='query')
    @blp.response(TeamSchema(many=True))
    @blp.paginate(SQLCursorPage)
    def get(self, args):
        """List teams"""
        return Team.query.filter_by(**args)
```

# Communauté, vie, feuille de route

- Rythme des sorties, versions, etc.
- Star / watch GitHub
- Inclusivité, bonnes pratiques


# Projets

## Nos projets

Citer nos projets utilisant marshmallow/flask-smorest

Description rapide, type d'utilisation, leçons


# Questions


# Contact
