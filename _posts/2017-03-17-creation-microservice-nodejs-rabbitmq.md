---
layout: post
title:  "Comment créer un microservice NodeJs pour gérer les files d'attentes de RabbitMQ ?"
date:   2017-03-17 08:23:36 +0200
tags: ["NodeJs", "Microservice", "RabbitMQ"]
---

## Introduction

En fonction de l'architecture du projet, la création d'un microservice pour traiter les messages d'une file d'attente peut s'avérer nécessaire.

Dans cet article, nous allons voir comment on peut initialiser un projet de ce type.

Nous allons donc :
 * réaliser l'installation et la configuration de RabbitMQ (un logiciel de gestion de file d'attente).
 * créer un microservice qui va traiter les messages de la file d'attente.
 * créer une API pour ajouter des messages.

Pour simplifier la tâche, nous allons tester ces éléments dans Docker. Si votre poste n'est pas configurer, je vous invite a installer les prérequis.

# Prérequis

 * Installation de [Docker Community Edition](https://store.docker.com/search?type=edition&offering=community)
 * Installation de [Docker Compose](https://docs.docker.com/compose/install/)

# Installation

 * Cloner et placer vous dans le projet

```
git clone 
```

 * Création de trois containers (2 pour NodeJs et un pour RabbitMQ) :

```
docker-compose up -d
```

 * Vérification du statut, les containers doivent être UP. Si ce n'est pas le cas relancer la commande précédente :

```
docker-compose ps
```

Ensuite, on va configurer RabbitMQ avec les trois commandes suivantes :

 * Création d'un "exchange" items

```
curl -i -u john:password -H "content-type:application/json" -XPUT -d'{"type":"direct","durable":true}' http://localhost:4002/api/exchanges/%2f/items
```

 * Création d'une "queue" items

```
curl -i -u john:password -H "content-type:application/json" -XPUT -d'{"auto_delete":false,"durable":true}' http://localhost:4002/api/queues/%2f/items
```

 * Connexion de "exchange" avec "queue"

```
curl -i -u john:password -H "content-type:application/json" -XPOST -d'{}' http://localhost:4002/api/bindings/%2f/e/items/q/items
```

# Microservice

Le microservice va traiter chaque message de la file d'attente.

Voici le code présant dans le projet :

``` javascript
'use strict';

const amqplib = require('amqplib');

const amqpUrl = `amqp://${process.env.RABBITMQ_USER}:${process.env.RABBITMQ_PASSWORD}@${process.env.RABBITMQ_HOST}${process.env.RABBITMQ_VHOST}`;
const rabbitmqConnection = amqplib.connect(amqpUrl);

rabbitmqConnection.then((conn) => {
  return conn.createChannel();
}).then((ch) => {
  // One by one message
  ch.qos(1);
  return ch.assertQueue(process.env.QUEUE, {durable: true}).then(() => {
    return ch.consume(process.env.QUEUE, (msg) => {
      handler(msg.content.toString(), () => {
        const currentdate = new Date();
        console.log(` => Callback at : ${currentdate.toLocaleTimeString()}`);
        // Message is done
        ch.ack(msg);
      });
    }, {noAck: false});
  });
}).catch(console.warn);

const handler = (message, callback) => {
  // Handle message
  console.log(`Message : ${message}`);

  // Execute callback
  callback();
};

```

# API

L'API va permettre l'ajout de message dans la file d'attente.

Voici le code présant dans le projet :

``` javascript
'use strict';

const express = require('express');
const app = express();
const port = process.env.PORT || 4001;

const rabbit = {
  generateMessage: function(nbMessage) {
    let lists = [];
    for (let i = 0; i < nbMessage; i++) {
      lists.push({
        message: `ID ${i}`
      });
    }

    return lists;
  },

  publishList: function(listContent, callback) {
    const amqpUrl = `amqp://${process.env.RABBITMQ_USER}:${process.env.RABBITMQ_PASSWORD}@${process.env.RABBITMQ_HOST}${process.env.RABBITMQ_VHOST}`;
    return amqplib.connect(amqpUrl).then(function(conn) {
      return conn.createChannel().then(function(ch) {
        const ok = ch.assertQueue(process.env.QUEUE, {durable: true});
        return ok.then(function(_qok) {
          listContent.forEach((content) => {
            ch.sendToQueue(process.env.QUEUE, new Buffer(JSON.stringify(content)));
          });

          return ch.close();
        });
      }).finally(function() { conn.close(); });
    }).catch(console.warn.bind(this, "catch"));
  },
};

app.get('/messages/:nbMessage', (req, res) => {
  // Generate messages
  const messages = rabbit.generateMessage(req.params.nbMessage);

  // Publish message to queue
  return rabbit.publishList(messages).then(() => {
    res.json({
      status: 'success',
      nbMessage: req.params.nbMessage
    });
  });
});

app.listen(port, () => {
  console.log('Server started: ' + port);
});

```

# Exécution

Après avoir vu l'installation du projet, le code d'exemple pour le microservice et l'API, nous allons pouvoir passer a l'exécution :

## Ajouter des messages (grâce a l'API)

Pour bien voir le mécanisme, je vais ajouter 10000 messages générés automatiquement.

```
curl http://localhost:4001/messages/10000
```

## Vérification dans RabbitMQ

RabbitMQ propose une interface pour connaitre l'état de la file d'attente

Les identifiants:
 * login: john
 * mot de passe : password

```
firefox http://localhost:4002/#/queues/%2F/items
```

Sur le graphique "Message rate", on peut voir une augmentation de la courbe correspondant au message que l'on vient d'ajouter

## Vérification dans les logs

Dans notre exemple, le microservice affichait juste le message de la file d'attente et le callback dans les logs.

```
docker-compose logs microservice
```

# Conclusion

J'espère que ce projet d'exemple vous aura aiguiller sur la manière de faire communiquer NodeJs et RabbitMQ.
