---
layout: post
title:  "Jenkins: test: Чистим змей"
date:   2013-11-28 00:20:22 +0400
author:
    name: Merkushev Kirill
    email: lanwen+blog@yandex.ru
    gravatar: 6ee51971263d8c9a1e70e1dac7418d36
categories: [jenkins]
tags: [ci, jenkins, jira]
comments: true
published: true
---

# ...или быстрое прототипирование на python для jenkins.



## Запуск тестов
Так вышло, что тесты у нас крутятся отдельно, а их запускалка имеет REST-API. Вообще говоря,
тут вполне можно пускать например вебные тесты на селениум-гриде.

В параметрах нам потребуется адрес машинки (чтобы его легко можно было сменить).

Именно здесь, из-за наличия рестового апи у нас полный простор для творчества. Для быстрого
старта был написан питоновский скрипт, который и совершает манипуляции с апи + пишет в консоль текущее состояние.
Скрипт сам находится в git репо, и при старте джобы клонируется на тестовую машинку, после чего запускается shell-шагом.

> **Кстати:** бывает на тестовой машинке нет гита, поэтому вполне можно организовать джобу, которая будет клонировать на мастере,
а дальше засылать на нужный стенд скрипт по ssh.

Весь код я приводить не буду, укажу только подводные камни, на которые мы напоролись.
Начало скрипта выглядит так:

{% highlight python %}
#!/usr/bin/env python
# -*- coding: utf-8 -*-

import sys
import httplib
import json
import time
{% endhighlight %}

Нам потребовалось обновить питон на конечной машинке, потому что тот, что стоял изначально не
обладал необходимым набором библиотек. Еще одна непрятная особенность - вывод русских
символов и сразу. Для этого пришлось обзавестись функцией:

{% highlight python %}
def printWithFlush(str):
    if isinstance(str, unicode):
        str = str.encode('utf-8')
    print str
    sys.stdout.flush()
    
{% endhighlight %}

Без такого способа печати, скрипт сначала накапливал буфер, а потом выплевывал все разом в
консоль (а нам то интересно тотчас узнавать что случилось). При этом на русских символах жалобно скулил и ничего дальше не делал.

Общением с рестовыми ручками занимаются такие методы:

{% highlight python %}
  def connect(self):
        return httplib.HTTPConnection(self.host)

    def get(self, path, maxErrorRate):
        #print path
        errorRate = 0;
        while True :
            conn = self.connect()
            conn.request('GET', path)
            response = conn.getresponse()
            if response.status != 200 :
                errorRate += 1
                if errorRate > maxErrorRate :
                    raise Exception("Bad http status %i" % (response.status) )
                time.sleep(1)
            else :
                data = response.read()
                #print data
                return data
{% endhighlight %}

Возможно, вечный цикл с выходом в возврат или эксепшен не лучшее решение, но это вполне рабочий пример.

У нас в планах написать полноценный плагин, который будет в дальнейшем этим заведовать вместо питона.

Помимо скрипта на запуск тестов, у нас на питоне написан скрипт для сборки метапакетов (пакетов, состоящих только из зависимостей),
скрипт для запуска юнит тестов для Си++ кода и предварительных проверок нужных хостов на живость, прежде чем с ними пытаться общаться
важными частями джобы.

> #### Статьи из этой серии:
>* [Прежде чем готовить кастрюлю][prestart] - вводная
>* [Рубим скорлупу][shell] - билд пакета, установка на машину
>* [Варим кофе][coffee] - есть место и для java
>* [Смешиваем][mix] - как все это превратить в цепочку

  [prestart]: {% post_url 2013-11-28-jenkins-jira-build-chain-0-prestart %}
  [shell]: {% post_url 2013-11-28-jenkins-jira-build-chain-1-shell %}
  [snakes]: {% post_url 2013-11-28-jenkins-jira-build-chain-2-snake %}
  [coffee]: {% post_url 2013-11-28-jenkins-jira-build-chain-3-coffee %}
  [mix]: {% post_url 2013-11-28-jenkins-jira-build-chain-4-mix %}