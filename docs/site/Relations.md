---
lang: en
title: 'Relations'
keywords: LoopBack 4.0, LoopBack 4
tags:
sidebar: lb4_sidebar
permalink: /doc/en/lb4/Relations.html
summary:
---

## Overview

Model relation in LoopBack 3 is one of its powerful features which help users
define real-world mappings between their models, access sensible CRUD APIs for
each of the models, and add querying and filtering capabilities for the relation
APIs after scaffolding their LoopBack applications. In LoopBack 4, with the
introduction of [repositories](Repositories.md), we aim to simplify the approach
to relations by creating constrained repositories. This means that, based on the
relation definition, certain constraints need to be honoured by the target model
repository, and thus, we produce a constrained version of it as a navigational
property on the source repository.

Here are the currently supported relations:

- [HasMany](HasMany-relation.md)
