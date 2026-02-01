---
layout: post
title: "About Undev Gitlab fork"
date: 2014-09-29
meta: true
comments: true
uglyURLs: true
aliases:
- /2014/09/29/undev_gitlab_fork
- /2014/09/29/undev_gitlab_fork.html
---

## About article

I get a lot of mail with questions about our ([Undev](http://undev.ru)) [Gitlab fork](https://github.com/Undev/gitlabhq).
![](https://camo.githubusercontent.com/66d74af59caab883bd6f82e5cefb63d0480a4db3e997b04f62aa5147656e13f8/68747470733a2f2f7261772e6769746875622e636f6d2f7a7a65742f6769746c616268712f6d61737465722f7075626c69632f756e6465765f6769746c61625f6c6f676f2e706e67) And... I lazy to reply every times similar letters so mach as I decided to write article about it :) In this article I try to describe fork and them main features.

I do not planned wrote about features a lot of information. If you have some question or can't understand something - please write comments, I'll update article.

## Start. Gitorious.

At September 2012 I was joined to Infrastructure projects in Undev. It was very interesting... but... old Gitorious, Rails 2, ruby 1.8.7ee... Shit. Ok. After some time of terrible work I started discussion with our manager about another system. We have a lot of plans (tickets), problems with Gitorious and support of them.

## Gitlab

Challenge accepted and we started research. At November 2012 we had big (very big) table with comparison of difficult systems. After some testing Gitlab wined and...

No!!!... Gitlab do not had some important for us features...

![](http://i.imgur.com/kACvxT7.jpg)

Okay. Let's go, Guys! We started new features.

* At first - postgresql support. Yes, I do not like MySQL :)
* Migration script from Gitorious to Gitlab
* Mail notifications
* Teams support for permissions management
* Snippets (but now we do not use this feature, because we prefer similar Wiki %) )
* etc.

At May 2013 we had good installation. But with some changes in architecture, about which I'll write later. We run Gitlab in production at May 2013.

## Most valuable Changes

Below I describe most important features from our fork. Difference between our fork and official repository huge and we can not send all features to upstream because of different reason. Such example as very big changes diff with hard migration - for features, which will be useful for big company.

### Teams support

In Gitlab 3 we do not had teams. Management of user access in a lot of projects was terrible work. If we want give for some user permissions to push to project - we must add user in project team directly. What about hundreds users and thousands projects? As result - we started **Teams feature**. First version  of this feature was not beautiful, some times - hardcoded, and in Gitlab 6 this feature was replaced with Groups by Dmitry Zaporozhets. I agree with them - for little company and teams - Team feature is overhead. But we can not abandon Team feature and rewrite them. Now we support this feature in new implementation.

We make 3 level of user access to projects:

* Add user directly to project
User -> Project

* Add user to group, in which located project
User -> Group -> Project

* Add user in Team, which can be assigned to project directly, or to group of projects
User -> Team -> Project
User -> Team -> Group -> Project

This schema very comfortable for our Managers, Project Masters/Owners ;)

### New event model

Then we (with [Andrey Kulakov](https://github.com/Andrew8xx8)) started **mail notification** feature, we selected event-based mail generation way. User can subscribe on base Entities: **Project**, **Group**, **Team**, **User**. After create Event on User action we create mail notifications, based on this Event, and async send them. More detailed I'll describe mail notification later. Now about events.

At beginning we had one huge problem. All events in Gitlab related to project. We had not events for Group or Team or another Entity. **Only Project**.

``` ruby
Event(id:          integer,
      target_type: string,  # Polymorth association with
      target_id:   integer, # Entity, in which was event
	  data:        text,    # Serialized event data
	  project_id:  integer, # Only project ;,( It so sadly...
	  created_at:  datetime,
	  action:      integer, # Action number.
	  author_id:   integer)
```

And all events described with 9 action constant:

``` ruby
  CREATED   = 1
  UPDATED   = 2
  CLOSED    = 3
  REOPENED  = 4
  PUSHED    = 5
  COMMENTED = 6
  MERGED    = 7
  JOINED    = 8 # User joined project
  LEFT      = 9 # User left project
```
So, it was a problem... Here could be image with angry Cat, but we started **new Events** feature.

We had next requirements:

* Any entity in system can be Event target (entity, related to which was created event)
* Any entity can be Event source (entity, which triggered event)
* Ability to describe action with human like name
* Store event data to future usage of them

As result we have some solution:
Any entity can be target
<!-- ![In new events any entity may be target](http://puu.sh/bcPBR/ececc5fafb.png) -->

Any Entity can be source
<!-- ![Simple example of relation between different entities with project](http://puu.sh/bcPGQ/83a806b906.png) -->

And rich action description (part or them):

``` ruby
  GENERAL = [
    :created,
    :updated,
    :commented,
    :deleted,
    :added,
    :removed,
    :joined,
    :left,
    :transfer,
  ]

  COMMENTS = [
    :commented_merge_request,
    :commented_issue, # not used in our fork
                      # because we use another issue tracker
    :commented_commit,
    :commented,       # not used after remove Project Wall
  ]

  MERGE_REQUESTS = [
    :opened,
    :closed,
    :reopened,
    :merged,
    :assigned,
    :reassigned,
    :resigned,
  ]

  MASS = [
    :imported,
    :members_added,
    :members_updated,
    :members_removed,
    :teams_added,
    :teams_removed,
    :groups_added,
    :projects_added
  ]

  GIT = [
    :pushed,
    :created_branch,
    :deleted_branch,
    :created_tag,
    :deleted_tag,
    :protected,
    :unprotected,
    :blocked,
    :activate,
  ]

  # NOTE actions which can be parent
  BASE = [
      :create,
      :update,
      :delete,
      :open,
      :close,
      :reopen,
      :merge,
      :block,
      :activate
  ]
```

Event can have parent event. For example case:

* User pushed code to server
	* After push was created note
	* And was closed MergeRequest
	* And was closed Issue

We create events for any actions in system. But save parent-child relation.

``` ruby
Push created |- Event(action: pushed)
             |- Event(action: commented_commit)
             |- Event(action: closed) # for MergeRequest
             |  |- Event(action: created) # Note was created
             |     |- Event(action: commented_merge_request)
             |
             |- Event(action: closed) # for Issue
                |- Event(action: created) # Note was created
                   |- Event(action: commented_issue)
```

Based on this events we can send email for different subscriptions without duplications. And we can research source of some troubles. I think it awesome! :)

After rewrite events we have:

* Ability detect who and what was doing in Gitlab on fuckups
* Full information dashboard for Main and Project, Group, Team, User pages.
* Flexible mail subscriptions with notifications
* Ability to send mail digests

<!-- ![](http://www.thinkuknow.co.uk/Global/swf/ThinkUKnow%205_7/nice.jpg) -->

### Awesome mail notifications

I am not exaggerating. Why I wrote some header you understand later.

After rewriting events we created own mail notifications.

Our workflow:

* User subscribe to base entity
	* **Project**
	* **Group**
	* **Team**
	* **User**

<!-- ![](http://puu.sh/bcRvH/ae02e0d349.png) -->

* User can edit detailed settings for any subscriptions

<!-- ![](http://puu.sh/bcRpA/90114af5ae.png) -->

* Another user do something in Gitlab
* After user actions - Gitlab trigger event creating process, and we have tree of events (described above)
* On created events base we create notifications for subscribers
* And send mails to subscribers in async queue.

At this moment we have more 100 different mails for different cases.
And can be more :)

**TODO**: Our plans create notification page with option to show notifications or send them on email.
**TODO**: Add ability to subscribe on MergeRequest and Issue. Replace participants with auto subscriptions.
**TODO**: Add ability to create *ignore* subscriptions.

### New Services logic

Gitlab has integration with different services and it's OK, but it implementation is ok for them, as SaaS. Why? In Gitlab code developers describe fields and logic of services. If we open project services page - Gitlab create empty records for everyone enabled in Gitlab service. For really needed service user enter some settings. Every time some similar settings user fill for different projects. It's so sadly... Service can only send web hooks. But, what about provide access to project code? Write comments/issue?

We rewrote services, because:

* For one organization it is ugly to write some configs many time while service added to different projects. We want to write config one time and enable service for different projects many time.
* We want to have ability for add deploy key for service.
* We want to have ability for integration of Gitlab with different other systems
* etc

Now:

* In Admin panel we add Service pattern with default values

<!-- ![](http://puu.sh/bcSXJ/40ec7e593e.png)
![](http://puu.sh/bcSTK/a62e2073f8.png) -->

* We can add sha-key for service (like deploy key)

<!-- ![](http://puu.sh/bcT25/0da1c3a2e3.png) -->

* In Any project we can enable this service pattern (we can edit config, if it need)

<!-- ![](http://puu.sh/bcT7s/1132856552.png) -->

If you create service pattern without default settings - user must fill service settings every time. Like in upstream %).


### Elasticsearch as search engine

What about search functionality in Gitlab?

<!-- ![](http://puu.sh/bdOe9/dcf7287ea1.png) -->

In official Gitlab CE we can search:

* Projects
* Groups *(autocomplete)*
* MergeRequests *(In selected project)*
* Issues *(In selected project)*
* Code *(in selected project)*

Projects, Groups, MergeRequest, Issue - `%like%` query.
Code - `git grep`...

At this moment Gitlab core team prefer PostgreSQL. So, in PostgreSQL we have good full-text search, but. We want more flexible search (sometimes user search with mistakes in query). And code search across all repositories. On this reason we replaced search with ElasticSearch.

Example of results:

We can search in different entities:

<!-- ![](http://puu.sh/bdOxK/bec6432c97.png) -->

We can search code across different repositories (as you can see - we can search filter search with different *Language*)

<!-- ![](http://puu.sh/bdOB6/0730e13767.png) -->

Resume: At this moment search available in:

* Project
	* Name
	* Path
	* NamespaceName
	* Description
* Group
	* Name
	* Path
	* Description
* User
	* Username
	* Name
	* etc (if you want)
* Team
	* Name
	* Path
	* Description
* MergeRequest
	* Tilte
	* Descriptions
* Issue
	* Title
	* Descriptions
* Code across all repositories
	* Files by content
	* Files by file-path and file-name
* Commits across all repositories
	* Commit author name, email
	* Commiter name, email
	* Commit message
	* commit sha

For integration with ElasticSearch I wrote [gem](https://github.com/zzet/elasticsearch-git). I tried save interface, but it is impossible without code rewrite. We have plans to create PR into official repository, but not in the fast time.

### Jenkins integration

So historically, that our company uses Jenkins. And we started research, how we can integrate Gitlab with Jenkins.

<!-- ![](http://puu.sh/bdQdN/d390f1fbaa.png) -->

We found [Gitlab Hook plugin](https://wiki.jenkins-ci.org/display/JENKINS/Gitlab+Hook+Plugin).

<!-- ![](http://puu.sh/bdQfS/8598e59bc9.png) -->

Ok. Jenkins + plugin -> Gitlab...

<!-- ![](http://puu.sh/bdQhr/c6629b390d.png) -->

Ok. A lot of projects.... Huge count of MergeRequests... As result we get fulltime pulling. How do you think that happened in the end? Yes. Gitlab is Die. Sadly.

We wrote [own plugin](https://github.com/alsor/gitlab-mergerequest-builder) for Jenkins. Thanks to [Alexey Sorokin](https://github.com/alsor).

How it work now:

```
-> User pushed code
   |- Gitlab run services hooks
     |- Gitlab service select data and send them to Jenkins
       |- Jenkins run build and after them send build data JSON to Gitlab
         |- Gitlab parse data and show them in Web UI.
```

As result we know - which push broke code and can fix them in short time.

<!-- ![](http://puu.sh/bdQRg/ed3e73c2ad.png) -->

And such results in Merge Requests :)

### Favorited projects (entities)

Many developers are involved in many projects. Find some of the projects is difficult. Necessary to use the search. This extra step and inconvenience. We decided to somehow identify projects in which the user is currently involved, or watched over. Result of brainstorm - we started **Favorited projects** feature.

User can add project in Feature list.

<!-- ![](http://puu.sh/bdQV1/7aa633d03e.png) -->

When user added project to this list - marked projects rendered in top of projects in dashboard sidebar:

<!-- ![](http://puu.sh/bdQWb/30a3e9d3e1.png) -->

And user can filter dashboard events feed. Show events for only favorited projects.

<!-- ![](http://puu.sh/bdQXB/7ab0f31c9b.png) -->

List of favorited projects can be edited in profile section.

<!-- ![](http://puu.sh/bdQZ2/f85ed49f68.png) -->

And all this features available for Group, Teams and Users.

### More performance with websockets

We replaces pulling for new notes in Merge Request, Commits and Issues with websockets. PR [here](https://github.com/gitlabhq/gitlabhq/pull/6822).

### Access to files via token

<!-- ![](http://puu.sh/bdR9t/8a6a59f513.png) -->

<!-- ![](http://puu.sh/bdRaY/cb3babfc7d.png) -->

<!-- ![](http://puu.sh/bdR6d/e20903ecc9.png) -->

### Git protocol support

<!-- ![](http://puu.sh/bdR3f/bd7f7ce770.png) -->

<!-- ![](http://puu.sh/bdR21/0e1bbb2df0.png) -->

## Architecture changes

Users:

* Git
	* Run application
	* Work with code
* Gitlab
	* Deploy application

<!-- ![](http://www.burnetts.com/wp-content/uploads/2012/02/top-secret.jpg) -->

```
.
├── some_git_home_path
│   └── git
│        ├── .ssh
│        ├── gitlab-shell # symlink to /some/apps_path/gitlab-shell/current
│        ├── ...
│        └── repositories
├── some
│   └── apps_path
│         ├── gitlab
│         │    ├── releases
│         │    │    ├── release_1
│         │    │    ├── release_2
│         │    │    ├── release_3
│         │    │    ├── release_4
│         │    │    └── release_5 # here only code
│         │    ├── shared
│         │    │    ├── bin
│         │    │    ├── bundle
│         │    │    ├── cache
│         │    │    ├── gitlab-satellites
│         │    │    ├── log
│         │    │    ├── pids
│         │    │    ├── public
│         │    │    ├── .secret
│         │    │    ├── system
│         │    │    ├── tmp
│         │    │    └── uploads
│         │    └── current # symlink to last release
│         └──  gitlab-shell
│              ├── releases
│              │    ├── release_1
│              │    ├── release_2
│              │    ├── release_3
│              │    ├── release_4
│              │    └── release_5
│              └── current # symlink to last release
└── some_services_path
     ├── gitlab-resque-main
     ├── gitlab-resque-gitlab-shell
     ├── gitlab-resque-main
     ├── gitlab-resque-elasticsearch
     ├── gitlab-web-unicorn
     ├── gitlab-web-unicorn-api
     ├── gitlab-web-faye
     └── etc
```

### Deploy via capistrano

## Related links

[Fork address (Undev/gitlabhq)](https://github.com/Undev/gitlabhq)

[Gitlab-shell fork for git protocol ability](https://github.com/zzet/gitlab-shell)

[Our vagrant vw for development](https://github.com/zzet/gitlab-wvm)

[Elasticsearch-git gem for Integration with ElasticSearch](https://github.com/zzet/elasticsearch-git)
