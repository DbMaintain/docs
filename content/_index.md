---
title: "Overview"
date: 2019-01-11T17:41:47+01:00
draft: false
---

Overview
===============

DbMaintain enables automatic roll-out of updates to a relational database. It brings database scripts into version control just like regular source code to transparently deploy databases from development to production.
	
DbMaintain keeps track of which database updates have been deployed on which database.

Updates are performed incrementally: Only what has been changed since the last deployment is applied. Features such as repeatable scripts, postprocessing scripts, multi-database and database user support and support for patches turn DbMaintain into a complete solution for the enterprise.


Getting Started
---------------

* If you are using maven, an overview of the available maven goals can be found on the [maven-goals](./docs/maven-goals) page.
* If you are using ant, an overview of the available ant tasks can be found on the [ant-tasks](./docs/ant-tasks) page.
* The [command-line](./command-line) page gives an overview on how to launch DbMaintain from the command line.
* Finally the [documentation](./docs) page gives an overview of all possible configuration settings.