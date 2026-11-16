---
title: "2016 - 2017, Doctor At Work"
order: 3
sidebar:
  - title: "Role"
    text: "Senior Software Engineer"
  - title: "Tech stack"
    text: ".NET Core, RabbitMQ, MSSQL"
  - title: "Location"
    text: "Moscow, Russia"
  - title: "Website"
    text: "[Google Play](https://play.google.com/store/apps/details?id=ru.doktornarabote.telemedicine)"
gallery:
  - url: /assets/images/experience/telemedicine-1.png
    image_path: /assets/images/experience/telemedicine-1.png
  - url: /assets/images/experience/telemedicine-2.png
    image_path: /assets/images/experience/telemedicine-2.png
  - url: /assets/images/experience/telemedicine-3.png
    image_path: /assets/images/experience/telemedicine-3.png
---

Doctoratwork is no ordinary company. It's a startup that initially started out as a professional social network for doctors, and has since amassed over 550,000 registered users. In 2016, I joined as their sole Senior Software Developer to help them achieve their ambitious goal - to develop and launch Russia's first telemedicine project.

It was an incredible year for me, as I gained valuable insights into working with cutting-edge technologies while keeping up with the company's pace of innovation. I was responsible for designing the microservice architecture for the project, creating multiple API platforms and libraries to consume them, leading the technical excellence of the project, and helping to grow the team to seven members.

Being able to work on such a grand project, driving and leading it, was an unforgettable experience. The feeling of pride that came with bringing new innovations to the world and making a change around us was indescribable. It was an intense time, but also incredibly rewarding. I'm honored to have been a part of it and will always cherish the memories.

{% include gallery %}

## My Contribution

- Implemented main platform with ASP.NET WebAPI – provided access to partners, provisioned services to assist communication with doctors from the social network, calculated costs of services and remuneration of doctors.
- Implemented authorization services (using OAuth protocol) and microservices using Quartz.NET task scheduler and Topshelf ([notes on OAuth refresh tokens and OWIN, in Russian]({% post_url 2016-12-25-API_developer_notes_part_1 %})).
- Created a service bus using RabbitMQ and [MassTransit]({% post_url 2016-11-3-My_experience_with_MassTransit_3_0 %}) to be used as a means of communication for the distributed system components.
- Developed and launched web applications and services, [used TeamCity, wrote PowerShell scripts, and configured Web Deploy]({% post_url 2016-11-27-Build_and_deploy_net_with_teamcity %}).
- Implemented a script describing the logic of the establishment and the behavior of the telephone connection between subscribers in JavaScript for the VoxImplant communication platform.
- Successfully crafted affiliate Web API platform – provided access to patients, such as the company’s mobile application users.

## Skills

- **Leadership:** Technical Leadership, Team Building (grew the team to 7)
- **Architecture:** Solution Architecture, Microservice Architecture, Distributed Systems
- **Backend:** .NET Core, ASP.NET Core, REST API, OAuth, Quartz.NET, Topshelf
- **Messaging & Data:** RabbitMQ, MassTransit, Microsoft SQL Server
- **Delivery:** TeamCity, PowerShell, Web Deploy

## Related posts

Written in Russian during this project:

- [Опыт использования MassTransit 3.0]({% post_url 2016-11-3-My_experience_with_MassTransit_3_0 %})
- [Разворачиваем ASP.NET приложения и windows сервисы с использованием TeamCity]({% post_url 2016-11-27-Build_and_deploy_net_with_teamcity %})
- [Заметки ASP.NET Web API/Backend разработчика (часть 1)]({% post_url 2016-12-25-API_developer_notes_part_1 %})
- [Использование единого IoC Container'a в рамках HTTP запроса между Web API и OWIN Middleware]({% post_url 2016-9-25-Implement_unity_container_per_http_request_middleware %})
- [Простейшее расширение для перехвата запроса в owin pipeline]({% post_url 2017-01-13-Simple_uptime_monitoring_middleware %})
- [NHibernate & JSON mapping: реализуем собственный CustomType (IUserType)]({% post_url 2017-02-22-NHibernate_custom_type_implementation_for_json %})
