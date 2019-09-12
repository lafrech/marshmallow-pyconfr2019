%Marshmallow
%De la sérialization à la construction d'une API REST

## Sommaire

- La sérialisation
- marshmallow
- L'écosystème marshmallow: webargs, apispec
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

## json

Objet → JSON → Objet

- Avantages
  - Standard / Inter-opérable
  - Lisible

- Inconvénients
  - Ne représente que certains types Python de base

Note : Plus ou moins équivalent à YAML, XML.

## Bibliothèque de sérialisation (1)

Transforme un objet Python en dictionnaire de types simples, JSONisable.

Objet → _dict_ → Objet

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

# marshmallow

## Fonctionnalités

- Sérialisation vers _dict_ ou JSON
- Désérialisation depuis _dict_ ou JSON
- Validation lors de la désérialisation

## Schémas et champs

```python
import marshmallow as ma

class TeamSchema(ma.Schema):
    name = ma.fields.String()
    creation_date = ma.fields.DateTime()
```

## Sérialisation

```python
import orm

class Team(orm.Model):
    name = orm.StringField()
    creation_date = orm.DateTiemField()

team = Team(
    name='A-Team',
    creation_date=dt.datetime(1983, 1, 23)
)

schema = TeamSchema()

schema.dump(team)
# {'name': 'A-Team', 'creation_date': '1983-01-23T00:00:00'}
```


