---
title: "Simple but safe pip requirements managment"
date: 2019-03-24T09:23:40+11:00
draft: true
---

In the beginning, virtualenvs were just a twinkle in someone's eye.
When you needed a python package, you didn't muck around:

{{< highlight shell >}}
sudo pip install requests
{{< / highlight  >}}
... which usually did not end well: every python app had to use the same package version, which meant any time you upgrade a package,
all your apps suddenly get the new version, for good or ill.

Virtualenvs were a step forward, but one big problem remained: do you use the package names "bare", implying the latest compatible version,
or do you specify the exact version of each?

`pipenv` tries to solve this, but makes it more complex than I believe is necessary. This is what I do:

1. create a directory in your project root called .requirements. Insert there any
(create stub repo and link to it)
