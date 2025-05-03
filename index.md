---
title: GitHub Actions を使用して Azure Load Testing を自動化する
permalink: index.html
layout: home
---

次の演習は、Azure Load Testing を使用してロード テストの実行を自動化する GitHub Actions とワークフローを実装する実践的な学習エクスペリエンスを提供するように設計されています。 

## 演習
<hr/>


{% assign labs = site.pages | where_exp:"page", "page.url contains '/Instructions/Labs'" %} {% for activity in labs  %}
* [{{ activity.lab.title }}]({{ site.github.url }}{{ activity.url }}) <br/> {{ activity.lab.description }} {% endfor %}
