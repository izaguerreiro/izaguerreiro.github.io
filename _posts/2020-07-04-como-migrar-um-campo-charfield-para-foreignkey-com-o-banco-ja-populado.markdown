---
layout: post
title: "Como migrar um campo Charfield para ForeignKey com o banco de dados já populado em Django?"
subtitle: ""
date: 2020-07-04 09:30:00
author: "Izabela Guerreiro"
header-img: "img/post-bg-06.jpg"
---

É comum que no decorrer de um projeto precisemos alterar a estrutura dos dados, seja por melhorias técnicas ou por necessidades da área de negócios. No meu caso uma necessidade da área de negócios em melhorar a usabilidade do django admin me levou a ter que migrar um campo do tipo Charfield para ForeignKey. Mas como fazer isso com o banco de dados populado e sem impactar os usuários?

Vamos imaginar um sistema de administração de livraria, atualmente ele possui a seguinte estrutura, um app chamado `bookstore` que contém o seguinte model:

```
from django.db import models


class Book(models.Model):
    title = models.Charfield(max_length=255)
    pages = models.IntegerField()
    author = models.Charfield(max_length=255)
```

Imaginando que o nome dos autores começaram a se repetir muito, criaremos um novo modelo para salvar os autores.

```
from django.db import models


class Book(models.Model):
    title = models.Charfield(max_length=255)
    pages = models.IntegerField()
    author = models.Charfield(max_length=255)


class Author(models.Model):
    name = models.Charfield(max_length=255)
```

Criaremos nossa migration, ela será responsável por criar a tabela `Author` no banco de dados.

```
./manage.py makemigrations
```

Agora que temos o model `Author` iremos adicionar uma ForeignKey `author_id` no model `Book`.

```
from django.db import models


class Book(models.Model):
    title = models.Charfield(max_length=255)
    pages = models.IntegerField()
    author = models.Charfield(max_length=255)
    author_id = models.ForeignKey(max_length=255)


class Author(models.Model):
    name = models.Charfield(max_length=255)
```

Feito isso, precisaremos gerar uma nova migration, para que essas alterações sejam aplicadas no banco de dados, mas agora teremos que criar uma migração vazia, pois queremos adicionar os registros existentes em `author` em nossa nova tabela `Author`. Para saber mais sobre migrações vazias [clique aqui](https://docs.djangoproject.com/en/3.0/topics/migrations/#data-migrations)

```
./manage.py makemigrations --empty bookstore
```

Com nossa migration vazia criada, iremos criar uma função que salvará os registros do campo `author` na tabela `Author` e em seguida iremos editar a campo `author` inserindo o objeto `Author`. Nosso arquivo de migração ficará assim:

```
from django.db import migrations


def transfer_author(apps, schema_editor):
    """ Transfer the author field from the Book model to the Author model. """

    Book = apps.get_model("bookstore", "Book")
    Author = apps.get_model("bookstore", "Author")

    for book in Book.objects.all():
        author, created = Author.objects.get_or_create(name=book.author)
        book.author = author
        book.save()


class Migration(migrations.Migration):

    dependencies = [
        ('bookstore', '0002_auto_20200518_0305'),
    ]

    operations = [
        migrations.RunPython(transfer_author),
    ]
```

O próximo passo é remover o campo `author`, pois ele não será mais utilizado.

```
from django.db import models


class Book(models.Model):
    title = models.Charfield(max_length=255)
    pages = models.IntegerField()
    author_id = models.ForeignKey(max_length=255)


class Author(models.Model):
    name = models.Charfield(max_length=255)
```

Criaremos uma nova migração.

```
./manage.py makemigrations
```

Agora podemos renomear o campo `author_id` para `author`.

```
from django.db import models


class Book(models.Model):
    title = models.Charfield(max_length=255)
    pages = models.IntegerField()
    author = models.ForeignKey(max_length=255)


class Author(models.Model):
    name = models.Charfield(max_length=255)
```

Essa será a última migração que criaremos.

```
./manage.py makemigrations
```

E por fim aplicaremos nossas migrações, utilizando o seguinte comando:

```
./manage.py migrate
```

Isso é tudo pessoal!!! Espero ter lhe ajudado de alguma forma. :)
