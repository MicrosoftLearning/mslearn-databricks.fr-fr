---
title: Instructions hébergées en ligne
permalink: index.html
layout: home
---

# Exercices Azure Databricks

Ces exercices sont conçus pour accompagner le contenu des formations suivantes sur Microsoft Learn :

- [Implémenter une solution d’analytique données avec Azure Databricks](https://learn.microsoft.com/training/paths/data-engineer-azure-databricks/)
- [Implémenter une solution Machine Learning avec Azure Databricks](https://learn.microsoft.com/training/paths/build-operate-machine-learning-solutions-azure-databricks/)

Vous aurez besoin d’un abonnement Azure dans lequel vous disposez d’un accès administratif pour réaliser ces exercices.

{% assign exercises = site.pages | where_exp:"page", "page.url contains '/Instructions'" %} {% for activity in exercises  %}
- [{{ activity.lab.title }}]({{ site.github.url }}{{ activity.url }}) | {% endfor %}