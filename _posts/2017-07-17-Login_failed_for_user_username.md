---
title: Login failed for user - когда все решения уже перепробованы
tags: MSSQL
redirect_from: "/Login_failed_for_user_username/"
---

Если предоставляя доступ к Microsoft SQL Server новому пользователю вы перепробовали уже всевозможные решения, а в ответ продолжаете получать одну и ту же ошибку "Login failed fo user 'username' (Error: 18456)" то знайте, что вы не одиноки. И есть ещё кое-что, что вы могли забыть.

Итак, checklist того, что вы могли забыть:

## Выдать права на вкладке Login / Server Roles

![instantiate function](/assets/images/posts/login_failed_1.png)

## Выдать права на подключение на вкладке Login / Securables

![instantiate function](/assets/images/posts/login_failed_2.png)

## Разрешить смешанный вид аутентификации на вкладке Server / Security

![instantiate function](/assets/images/posts/login_failed_3.png)
