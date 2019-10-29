Support de présentation marshmallow

Génération en local

    pandoc -t revealjs -s presentation.md -o ../index.html -V width="'100%'" -V height="'100%'" --css marshmallow-pyconfr2019/custom.css --slide-level 2

Génération pour gh-pages

    pandoc -t revealjs -s presentation.md -o index.html -V width="'100%'" -V height="'100%'" -V revealjs-url=https://cdnjs.cloudflare.com/ajax/libs/reveal.js/3.8.0 --css custom.css --slide-level 2


gh-pages : https://lafrech.github.io/marshmallow-pyconfr2019/

Exemple :
https://github.com/lafrech/flask-smorest-sqlalchemy-example
https://team-manager-demo.jolimont.fr/redoc
https://team-manager-demo.jolimont.fr/swagger-ui

Démo : https://mybinder.org/v2/gh/lafrech/flask-smorest-sqlalchemy-example/pycon2019?filepath=demo%2Fdemo.py
