% Marshmallow \
  De la sérialization \
  à la construction d'une API REST
% Jérôme Lafréchoux
% PyConFR - 3 novembre 2019

## Sommaire

- La sérialisation
- marshmallow
- L'écosystème marshmallow
- Construction d'une API REST: flask-smorest

# La sérialisation

## C'est quoi?

- Codage d'objets Python sous une forme adaptée pour
    - Sauvegarde
    - Transport

- Processus réversible (désérialisation)

- Utilisations typiques
    - Format d'échange (ex: service web)
    - Fichier de configuration
    - Sauvegarde de résultats de calcul
    - Stockage temporaire sur disque
    - ...

## Pickle

Objet → _Byte stream_ → Objet

- Avantages
    - Bibliothèque standard
    - Rapide

- Inconvénients
    - Compatibilité : désérialisation nécessite Python
    - Sécurité : injection de code

## json (1)

Objet → JSON → Objet

```python
import json

user = {"name": "Roger"}

json.dumps(user)
# '{"name": "Roger"}'

json.loads('{"name": "Roger"}')
# {'name': 'Roger'}
```

## json (2)

Mais JSON ne définit que des types basiques :
_number_, _string_, _boolean_, _array_, _object_, _null_.

```python
import json
import datetime as dt

user = {"name": "Roger", "birth_date": dt.datetime(1983, 1, 23)}

json.dumps(user)
# TypeError: datetime.datetime(1983, 1, 23, 0, 0) is not JSON serializable
```

## json (3)

- Avantages
    - Standard / Inter-opérable
    - Lisible

- Inconvénients
    - Ne représente que quelques types basiques

## Bibliothèque de sérialisation (1)

Transforme un objet Python en dictionnaire de types simples, _JSONisable_

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

# marshmallow

## Fonctionnalités

- Sérialisation vers _dict_ ou JSON
- Désérialisation depuis _dict_ ou JSON
- Validation lors de la désérialisation


```{.ascii-art}
 --------                ------------
|        | === dump ==> |            |
| Object |              | dict / str |
|        | <== load === |            |
 --------                ------------
```

```{.ascii-art}
 --------                 ------
|        | === dumps ==> |      |
| Object |               | JSON |
|        | <== loads === |      |
 --------                 ------
```


## Schémas et champs

```python
import datetime as dt
import marshmallow as ma

class UserSchema(ma.Schema):
    name = ma.fields.String()
    birth_date = ma.fields.DateTime()

schema = UserSchema()

user = {"name": "Roger", "birth_date": dt.datetime(1983, 1, 23)}

schema.dump(user)
# {'name': 'Roger', 'birth_date': '1983-01-23T00:00:00'}

schema.dumps(user)
# '{"name": "Roger", "birth_date": "1983-01-23T00:00:00"}'
```

## Séparation modèle / vue (1)

Modèle

```python
import orm

class Member(orm.Model):
    first_name = orm.StringField()
    last_name = orm.StringField()
    birthdate = orm.DateTimeField()
    age = orm.IntegerField()
    password = orm.StringField()

class Team(orm.Model):
    name = orm.StringField()
    creation_date = orm.DateTimeField()
    members = orm.ManyToMany(Member)
```

## Séparation modèle / vue (2)

Schémas

```python
import marshmallow as ma

class TeamSchema(ma.Schema):
    name = ma.fields.String()
    creation_date = ma.fields.DateTime()
```

Ressources

```python
from .models import Team
from .schemas import TeamSchema

team = Team.get(name="Ghostbusters")

schema = TeamSchema()

schema.dump(team)
# {'name': 'Ghostbusters', 'creation_date': '1983-01-23T00:00:00'}
```

## Read-only, Write-only

```python
class MemberSchema(ma.Schema):
    first_name = ma.fields.String()
    last_name = ma.fields.String()
    birthdate = ma.fields.DateTime()
    age = ma.fields.Int(dump_only=True)
    password = ma.fields.Str(load_only=True)

member = Member.get_one(last_name='Venkman')

MemberSchema().dump(member)
# {'first_name': 'Peter', 'last_name': 'Venkman', 'birthdate': '1960-09-06T00:00:00', 'age': 59}
```

## Sélection dynamique de champs

```python
class MemberSchema(ma.Schema):
    first_name = ma.fields.String()
    last_name = ma.fields.String()
    birthdate = ma.fields.DateTime()

MemberSchema(only=("last_name", )).dump(member)
# {'last_name': 'Venkman'}

MemberSchema(exclude=("last_name", )).dump(member)
# {'first_name': 'Peter', 'birthdate': dt.datetime(1960, 9, 6)}
```

## Découplage des noms de champs entre modèle et API

```python
class MemberSchema(ma.Schema):
    first_name = ma.fields.String(data_key="firstName")
    last_name = ma.fields.String(data_key="lastName")
    birthdate = ma.fields.DateTime(data_key="birth-date")

MemberSchema().dump(member)
# {'firstName': 'Peter', 'lastName': 'Venkman', 'birth-date': '1960-09-06T00:00:00'}
```

## Collections

```python
class MemberSchema(ma.Schema):
    first_name = ma.fields.String(data_key="firstName")
    last_name = ma.fields.String(data_key="lastName")

members = Member.get_all()

schema = MemberSchema(many=True)

schema.dump(members)
# [
#     {'first_name': 'Egon', 'last_name': 'Spengler',},
#     {'first_name': 'Peter', 'last_name': 'Venkman',},
# ]
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

team = Team.get_one(name="Ghostbusters")

TeamSchema().dumps(team)
# {'name': 'Ghostbusters',
#  'members': [
#   {'first_name': 'Egon', 'last_name': 'Spengler', 'birthdate': '1958-10-02T00:00:00'},
#   {'first_name': 'Peter', 'last_name': 'Venkman', 'birthdate': '1960-09-06T00:00:00'}
# ]}
```

## Pré-post traitements

``pre_load`` / ``post_load`` / ``pre_dump`` / ``post_dump``

```python
class MemberSchema(ma.Schema):
    first_name = ma.fields.String()
    last_name = ma.fields.String()
    birthdate = ma.fields.DateTime()

    @ma.post_load
    def make_instance(self, data, **kwargs):
        return Member(**data)

    member = MemberSchema().load(
        {"first_name": "Peter", "last_name": "Venkman", "birthdate": dt.datetime(1960, 9, 6)}
    )
    member.first_name
    # 'Peter'
```

## Validation (1)

Validation à la désérialisation

- Champs obligatoires
- Validation des valeurs

```python
class MemberSchema(ma.Schema):
    first_name = ma.fields.String(validate=ma.validate.Length(min=2, max=50))
    last_name = ma.fields.String(required=True)
    birthdate = ma.fields.DateTime()
```

## Validation (2)

Structuration des messages d'erreur

```python
MemberSchema().load({"first_name": "V"})
# marshmallow.exceptions.ValidationError: {
#     'last_name': ['Missing data for required field.'],
#     'first_name': ['Length must be between 2 and 50.']
# }
```

## Questions

# Intégration ORM / ODM

## ORM / ODM

- ORM : _Object-Relation Mapping_
- ODM : _Object-Document Mapping_

- Couche d'abstraction entre objets et base de donnée
- Définit le modèle avec des schémas

## Intégration ORM/ODM - marshmallow

- Génération automatique de schémas marshmallow depuis le modèle
- Types et validateurs inférés des classes du modèle
- Permet de générer des schémas d'API en minimisant la duplication de code

```{.ascii-art}
                      ---------------------------
                     |                           |
 ----------          |          --------         ▼         ------------
|          |      Schema       |        |     Schema      |            |
| Database | <== ORM / ODM ==> | Object | <==   API   ==> | dict / str |
|          |                   |        |   marshmallow   |            |
 ----------                     --------                   ------------
```

## Exemples d'intégration

- SQLAlchemy → marshmallow-sqlalchemy
- peewee → marshmallow-peewee
- MongoEngine → marshmallow-mongoengine

## marshmallow-mongoengine

### Modèle

```python
import mongoengine as me

class Team(me.Document):
    name = me.StringField(max_length=40)
    members = me.ListField(me.ReferenceField("Member"))

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

team = Team.objects.get(name="Ghostbusters")

TeamSchema().dump(team)
# {'id': 1,
#   'name': 'Ghostbusters',
#  'members': [
#   {'first_name': 'Egon', 'last_name': 'Spengler', 'birthdate': '1958-10-02T00:00:00'},
#   {'first_name': 'Peter', 'last_name': 'Venkman', 'birthdate': '1960-09-06T00:00:00'}
# ]}

TeamSchema().load({"name": "This name is too long to pass validation."})
# marshmallow.exceptions.ValidationError: {'name': ['Longer than maximum length 40.']}
```

## umongo : ODM MongoDB

- Alternative à MongoEngine + marshmallow-mongoengine
- Utilise marshmallow pour sérialization/désérialisation MongoDB BSON
- Génère schemas marshmallow pour API
- Fonctionne avec PyMongo, TxMongo, Motor

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

@app.route("/teams/", methods=['POST'])
def post():
    # Désérialisation et validation
    try:
        team_data = team_schema.load(request.json)
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

@app.route("/", methods=['POST'])
@use_args(TeamSchema, location="json")
def post(team_data):
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
spec.init_app(app)
```
---------------------------------------------------

```python
@app.route("/teams/", methods=["POST"])
@use_args(TeamSchema, location="json")
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
    team = Team(**team_data)
    team.save()
    return TeamSchema().dump(team), 201

spec.path(view=post_team)
```

## webargs + apispec : limitations

- Duplication : docstring YAML
- Sérialisation manuelle

# flask-smorest

## Fonctionnalités

- Sérialisation / désérialisation des entrées / sorties : webargs
- Documentation OpenAPI automatique (ou presque) : apispec
- Pagination
- ETag

## Structuration d'une ressource

- ``Blueprint`` → ressource
- ``MethodView`` → GET, POST, PUT, DELETE

---------------------------------------------------

```python
@blp.route("/")
class Teams(MethodView):

    @blp.arguments(TeamQueryArgsSchema, location="query")
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

@blp.route("/<uuid:item_id>")
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

## Pagination

- Pagination `a posteriori` des resources renvoyant une liste
- Validation des paramètres d'entrée ``page`` et ``page_size`` (query args)
- Éléments de pagination renvoyés dans un Header

```python
headers["X-Pagination"]
# {
#     'total': 1000, 'total_pages': 200,
#     'page': 2, 'first_page': 1, 'last_page': 200,
#     'previous_page': 1, 'next_page': 3,
# }
```

## Pagination d'un curseur de base de donnée

```python
from flask_smorest import Page

class SQLCursorPage(Page):
    @property
    def item_count(self):
        return self.collection.count()


@blp.route("/")
class Teams(MethodView):

    @blp.arguments(TeamQueryArgsSchema, location="query")
    @blp.response(TeamSchema(many=True))
    @blp.paginate(SQLCursorPage)
    def get(self, args):
        """List teams"""
        return Team.query.filter_by(**args)
```

## ETag

- Identifie une version spécifique d'une ressource

- GET : Économie de bande passante
    - `If-None-Match: "686897696a7c876b7e"`
    - Optionnel dans la requête
    - Si ETag correspond (ressource non modifiée), `304 Not Modified`

- PUT/DELETE : Empêche les mises à jour simultanées
    - `If-Match: "686897696a7c876b7e"`
    - Obligatoire dans la requête
    - Si ETag manquant dans la requête, `428 Precondition required`
    - Si ETag ne correspond pas (ressource modifiée), `412 Precondition failed`

## Démo


# Communauté, feuille de route

## Cycle de publication (1)

Actuellement, deux branches de marshmallow maintenues.

|Branche|Python|Date de publication|
|-|-|-|
|2.x|2.7+, 3.4+|25 septembre 2015|
|3.x|3.5+|18 août 2019|

## Cycle de publication (2)

marshmallow est utilisé dans beaucoup de bibliothèques et _frameworks_.

Versions majeures peu fréquentes, nombreux changements.

|Version|Date de publication|
|-|-|
|1.0.0|16 novembre 2014|
|2.0.0a1|25 avril 2015|
|2.0.0|25 septembre 2015|
|3.0.0a1|26 février 2017|
|3.0.0|18 août 2019|

webargs, apispec : versions majeures plus fréquentes, changements limités.

## Bonnes pratiques

- Intégration continue, pytest, flake8, black, mypy
- Python 3 partout
- Communauté inclusive

# Nos projets

## Nobatek/INEF4

Institut National pour la Transition Énergétique et Environnementale du Bâtiment

## Proleps

Gestion énergétique de patrimoine immobilier

Planification de rénovation

- MongoDB / umongo
- marshmallow 2
- flask-smorest

https://www.nobatek.inef4.com/produits/proleps/

## BEMServer (Hit2Gap EU H2020)

BuildingEnergyManagement Server

Plateforme _open-source_ de gestion énergétique du bâtiment

Trois bases de données

- Modèle ontologique du bâtiment
    - Ontologie ifcOWL étendue
    - Jena SPARQL, webservice
    - Lecture / écrite en BDD manuelle, pas d'ORM

- Séries temporelles (HDF5)
    - Écriture via Pandas
    - Contourne marshmallow dans l'API (performance)

- Evènements (SQLite)
    - SQLAlchemy

https://www.bemserver.org/

## Sigopti

Plugin _open-source_ de QGis

Pré-étude de faisabilité de réseaux de chaleur

Calculs asynchrones sur serveur distant via API web

- Solver IPOPT
- Pyomo
- Celery
- Redis
- flask-smorest

## Nature4Cities (EU H2020)

Plateforme de comparaison et d'évaluation de solutions fondées sur la nature

Indicateurs socio-économiques, environnementaux, urbanisme...

Calculs synchrones sur serveur distant via API web

https://www.nature4cities.eu/

# Questions


# Liens


