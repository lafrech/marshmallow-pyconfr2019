Support de présentation marshmallow

    pandoc -t revealjs -s presentation.md -o ../index.html -V width="'100%'" -V height="'100%'" --css marshmallow-pyconfr2019/custom.css --slide-level 2

    pandoc -t revealjs -s presentation.md -o index.html -V width="'100%'" -V height="'100%'" -V revealjs-url=https://cdnjs.cloudflare.com/ajax/libs/reveal.js/3.8.0 --css custom.css --slide-level 2
