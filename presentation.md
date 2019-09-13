%Marshmallow
%De la sérialization à la construction d'une API REST

## Sommaire

- La sérialisation
- marshmallow
- L'écosystème marshmallow
- Construction d'une API REST: flask-rest-api

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

## json (2)

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

team = {'name': 'A-Team', 'creation_date': dt.datetime(1983, 1, 23)}

class TeamSchema(ma.Schema):
    name = ma.fields.String()
    creation_date = ma.fields.DateTime()

schema = TeamSchema()

schema.dump(team)
# {'name': 'A-Team', 'creation_date': '1983-01-23T00:00:00'}

schema.dump(team)
# '{"name": "A-Team", "creation_date": "1983-01-23T00:00:00"}'
```

## Séparation modèle / vue

Modèle

```python
import orm

class Team(orm.Model):
    name = orm.StringField()
    creation_date = orm.DateTiemField()

```

Vue

```python
from .models import Team
from .schemas import TeamSchema

team = Team.get(name='A-Team')

schema = TeamSchema()

schema.dump(team)
# {'name': 'A-Team', 'creation_date': '1983-01-23T00:00:00'}
```



## TODO

- nested fields

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


# Écosystème

## Intégration ORM/ODM

- sqlalchemy
- peewee
- mongoengine

## µmongo : ODM MongoDB

## webargs : Parse request arguments

## apispec : Generate OpenAPI documentation

## flask-rest-api

- Injection de paramètres avec webargs
- Doc auto avec apispec, sans YAML
- MethodView + Blueprint
- ETag
- Pagination


# Utilisateurs

## Star / watch GitHub

## Nos projets



# Questions


# Contact

