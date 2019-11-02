% Marshmallow \
  De la sérialization \
  à la construction d'une API REST
% Jérôme Lafréchoux
% PyConFR - 3 novembre 2019


# Plan

- La sérialisation
- marshmallow
- L'écosystème marshmallow
- Construction d'une API REST: flask-smorest
- Développer avec marshmallow
- Nos projets

![](assets/marshmallow-logo-white.png)


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
    - ...

## Pickle

```{.ascii-art}
 --------                -------------
|        | === dump ==> |             |
| Object |              | Byte stream |
|        | <== load === |             |
 --------                -------------
```

- Avantages
    - Bibliothèque standard
    - Rapide

- Inconvénients
    - Compatibilité : désérialisation nécessite Python
    - Sécurité : injection de code

## json (1)

```{.ascii-art}
 --------                ------
|        | === dump ==> |      |
| Object |              | JSON |
|        | <== load === |      |
 --------                ------
```

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

|JSON|Python|
|-|-|
|object|dict|
|array|list|
|string|str|
|number (int/float)|int/float|
|boolean|bool|
|null|None|


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

```{.ascii-art}
 --------                ------
|        | === dump ==> |      |
| Object |              | dict |
|        | <== load === |      |
 --------                ------
```

Surcouche de `json`

```{.ascii-art}
 --------                ------                 ------
|        | === dump ==> |      |  === dump ==> |      |
| Object |              | dict |               | JSON |
|        | <== load === |      |  <== load === |      |
 --------                ------                 ------
```

## Bibliothèque de sérialisation (2)

- Avantages
    - Standard / Inter-opérable
    - Lisible
    - Pas limité aux types simples

- Inconvénients
    - Nécessite de définir la sérialisation des objets non standards
    - Bibliothèque non standard


# marshmallow {data-background-image="assets/marshmallow-stay-puft.webp"}

## Fonctionnalités

- Sérialisation vers _dict_ ou JSON
- Désérialisation depuis _dict_ ou JSON
- Validation lors de la désérialisation

```{.ascii-art}
 --------                           -------------
|        | ===      dump       ==> |             |
| Object |                         | dict / JSON |
|        | <== load & validate === |             |
 --------                           -------------
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

## Validation

Validation à la désérialisation

- Champs obligatoires
- Validation des valeurs
- Structuration des messages d'erreur

```python
class MemberSchema(ma.Schema):
    first_name = ma.fields.String(validate=ma.validate.Length(min=2, max=50))
    last_name = ma.fields.String(required=True)
    birthdate = ma.fields.DateTime()

MemberSchema().load({"first_name": "V"})
# marshmallow.exceptions.ValidationError: {
#     'last_name': ['Missing data for required field.'],
#     'first_name': ['Length must be between 2 and 50.']
# }
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
    first_name = ma.fields.String()
    last_name = ma.fields.String()

members = Member.get_all()

schema = MemberSchema(many=True)

schema.dump(members)
# [
#     {'first_name': 'Egon', 'last_name': 'Spengler',},
#     {'first_name': 'Peter', 'last_name': 'Venkman',},
# ]
```

## Schémas imbriqués

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
 ----------          |          --------         ▼         -------------
|          |      Schema       |        |     Schema      |             |
| Database | <== ORM / ODM ==> | Object | <==   API   ==> | dict / JSON |
|          |                   |        |   marshmallow   |             |
 ----------                     --------                   -------------
```

## Exemples d'intégration

- SQLAlchemy → marshmallow-sqlalchemy
- peewee → marshmallow-peewee
- MongoEngine → marshmallow-mongoengine

## marshmallow-mongoengine (1)

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

## marshmallow-mongoengine (2)

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

- Alternative à { MongoEngine + marshmallow-mongoengine }
    - Utilise marshmallow pour sérialization/désérialisation MongoDB BSON
    - Génère schemas marshmallow pour API

- Fonctionne avec PyMongo, TxMongo, Motor

# webargs

## webargs : désérialisation de requêtes

Désérialise et valide les requêtes HTTP

Injecte le contenu de la requête dans la fonction de vue

## Sans webargs

```python
from flask import Flask, request

app = Flask(__name__)

team_schema = TeamSchema()

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

## Compatibilité

Prend en charge nativement les principaux serveurs web :

Flask, Django, Bottle, Tornado, Pyramid, webapp2, Falcon, aiohttp


# apispec

## apispec : Documentation OpenAPI (Swagger)

Génération de la documentation OpenAPI

Introspection des schémas marshmallow

![](assets/openapi-logo.png)

## webargs + apispec : exemple (1)

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

## webargs + apispec : exemple (2)

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

# flask-smorest {data-background-image="assets/smore.gif" .dark}

## Présentation

- Environnement
    - marshmallow, webargs, apispec
    - Flask
- Fonctionnalités
    - Sérialisation / désérialisation des entrées / sorties : webargs
    - Documentation OpenAPI automatique (ou presque) : apispec
    - Pagination
    - ETag
- Anciennement flask-rest-api

## Structuration d'une ressource (1)

- ``flask.Blueprint`` → ressource
- ``flask.MethodView`` → GET, POST, PUT, DELETE

## Structuration d'une ressource (2)

```python
@blp.route("/")
class Teams(MethodView):

    @blp.arguments(TeamQueryArgsSchema, location="query")
    @blp.response(TeamSchema(many=True))
    def get(self, args):
        """List teams"""
        return list(Team.query.filter_by(**args))

    @blp.arguments(TeamSchema)
    @blp.response(TeamSchema, code=201)
    def post(self, new_team):
        """Add a new team"""
        team = Team(**new_team)
        db.session.add(team)
        db.session.commit()
        return team
```

## Structuration d'une ressource (3)

```python
@blp.route("/<uuid:team_id>")
class TeamsById(MethodView):

    @blp.response(TeamSchema)
    def get(self, team_id):
        """Get team by ID"""
        return Team.query.get_or_404(team_id)

    @blp.arguments(TeamSchema)
    @blp.response(TeamSchema)
    def put(self, new_team, team_id):
        """Update an existing team"""
        team = Team.query.get_or_404(team_id)
        TeamSchema().update(team, new_team)
        db.session.add(team)
        db.session.commit()
        return team

    @blp.response(code=204)
    def delete(self, team_id):
        """Delete a team"""
        team = Team.query.get_or_404(team_id)
        db.session.delete(team)
        db.session.commit()
```

## Pagination (1)

- Pagination `a posteriori` des resources renvoyant une liste
- Validation des paramètres d'entrée ``page`` et ``page_size`` (_query args_)
- Éléments de pagination renvoyés dans un Header

```python
from flask_smorest import Page

@blp.route("/")
class Teams(MethodView):

    @blp.arguments(TeamQueryArgsSchema, location="query")
    @blp.response(TeamSchema(many=True))
    @blp.paginate(Page)
    def get(self, args):
        """List teams"""
        return Team.get_list(filters=args)

headers["X-Pagination"]
# {
#     'total': 1000, 'total_pages': 200,
#     'page': 2, 'first_page': 1, 'last_page': 200,
#     'previous_page': 1, 'next_page': 3,
# }
```

## Pagination (2)

Pagination d'un curseur de base de donnée

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

## Démo {data-background-image="assets/demo.gif"}


# Développer avec marshmallow

## Cycle de publication

Actuellement, deux branches de marshmallow maintenues

|Branche|Python|Date de publication|
|-|-|-|
|2.x|2.7+, 3.4+|25 septembre 2015|
|3.x|3.5+|18 août 2019|

- marshmallow
    - Utilisé dans beaucoup de bibliothèques et _frameworks_
    - Versions majeures peu fréquentes, nombreux changements
- webargs, apispec,...
    - Versions majeures plus fréquentes, changements limités

## Bonnes pratiques

- Intégration continue, pytest, flake8, black, mypy
- Python 3, annotations
- Communauté inclusive


# Nos projets

## Nobatek/INEF4

Institut National pour la Transition Énergétique et Environnementale du Bâtiment

![](assets/nobatek-inef4-logo.png)

[https://www.nobatek.inef4.com](https://www.nobatek.inef4.com)

## Proleps

Gestion énergétique de patrimoine immobilier

Planification de rénovation

- MongoDB / umongo
- flask-smorest

![](assets/proleps-logo.png)

[https://www.nobatek.inef4.com/produits/proleps/](https://www.nobatek.inef4.com/produits/proleps/)

## BEMServer (Hit2Gap EU H2020) (1)

BuildingEnergyManagement Server

Plateforme _open-source_ de gestion énergétique du bâtiment

![](assets/bemserver-logo.png)

[https://www.bemserver.org/](https://www.bemserver.org/)

## BEMServer (Hit2Gap EU H2020) (2)

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

![](assets/nature4cities-logo.png)

[https://www.nature4cities.eu/](https://www.nature4cities.eu/)


# Questions


# Liens

[https://lafrech.github.io/marshmallow-pyconfr2019/](https://lafrech.github.io/marshmallow-pyconfr2019/)

[https://github.com/marshmallow-code](https://github.com/marshmallow-code)

[https://github.com/lafrech/flask-smorest-sqlalchemy-example](https://github.com/lafrech/flask-smorest-sqlalchemy-example)
