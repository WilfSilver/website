---
title: Logseq v2 queries and where to find them
author: Wilf Silver
pubDatetime: 2026-10-17T19:00:00Z
slug: logseq_v2_queries
featured: true
draft: false
tags:
  - logseq
  - clojure
description: While trying out logseq, their V2 beta heavily changed how queries work so this is how I translated my queries from V1 to V2
---

Recently I moved away from [AnyType](https://anytype.io) as my primary note taking app due to the new direction (Anytwo) failing to capture the reason I loved AnyType in the first place. So I started looking for alternatives, which is annoyingly quite difficult as AnyType is such a unique take on note-taking.

This is where I started looking into [Logseq](https://logseq.com/), as I had heard some great things. I found that it had a much more primitive approach to note taking and their journalling system is just completely rules over all, making it easy to log important events or actions you need to take. So over the past few months I have been daily driving it... which is when I found out there's two versions, v1 and v2...

## Default queries

One of the really cool features in logseq is the journalling, where you effectively just write in the automatically created page for the current day. You then tag or link each block to the system you are taking about, so later you can look at the systems page to find what happened and when.

So when you are working in today's page only, you may want to show important information, such as your scheduled TODOs, anything you missed over the last couple days or even upcoming tasks or events. These all can be done via the default queries feature in the `config.edn` file.

So for the logseq v1 I just copied some examples from the internet and produced:

```clojure
{
 ;; Advanced queries `:result-transform` function.
 ;; Transform the query result before displaying it.
 :query/result-transforms
 {:sort-by-priority
  (fn [result] (sort-by (fn [h] (get h :block/priority "B")) result))}

 ;; The following queries will be displayed at the bottom of today's journal page.
 ;; The "NOW" query returns tasks with "NOW" or "DOING" status.
 ;; The "NEXT" query returns tasks with "NOW", "LATER", or "TODO" status.
 :default-queries
 {:journals
  [{:title "🔨 NOW"
    :query [:find (pull ?h [*])
            :in $ ?start ?today
            :where
            [?h :block/marker ?marker]
            [(contains? #{"NOW" "DOING"} ?marker)]
            [?h :block/page ?p]
            [?p :block/journal? true]
            [?p :block/journal-day ?d]
            [(>= ?d ?start)]
            [(<= ?d ?today)]]
    :inputs [:365d :today]
    :result-transform :sort-by-priority
    :group-by-page? false
    :collapsed? false}

   {:title "📅 Today"
    :query [:find (pull ?h [*])
            :in $ ?start ?next
            :where
            [?h :block/marker ?marker]
            (or [?h :block/scheduled ?d] [?h :block/deadline ?d])
            [(contains? #{"TODO" "LATER" "WAITING"} ?marker)]
            [(>= ?d ?start)]
            [(<= ?d ?next)]]
    :inputs [:today :today]
    :result-transform :sort-by-priority
    :group-by-page? false
    :collapsed? false}

   {:title "🔥 Overdue"
    :query [:find (pull ?h [*])
            :in $ ?start ?next
            :where
            [?h :block/marker ?marker]
            (or [?h :block/scheduled ?d] [?h :block/deadline ?d])
            [(contains? #{"NOW" "DOING" "TODO" "LATER" "WAITING"} ?marker)]
            [(>= ?d ?start)]
            [(< ?d ?next)]]
    :inputs [:365d :today]
    :result-transform (fn [result]
      (sort-by (fn [h] (get h :block/priority "B"))
        (sort-by (fn [h] (get h :block/scheduled (get h :block/deadline 0)))
          result)))
    :group-by-page? false
    :collapsed? false}
 
   {:title "📅 NEXT"
    :query [:find (pull ?h [*])
            :in $ ?start ?next
            :where
            [?h :block/marker ?marker]
            [(contains? #{"NOW" "LATER" "TODO"} ?marker)]
            [?h :block/page ?p]
            (or [?p :block/journal-day ?d]
                [?h :block/scheduled ?d]
                [?h :block/deadline ?d])
            [(> ?d ?start)]
            [(< ?d ?next)]]
    :inputs [:today :7d-after]
    :result-transform (fn [result]
      (sort-by (fn [h] (get h :block/scheduled (get h :block/deadline 0)))
        (sort-by (fn [h] (get h :block/priority "B"))
          result)))
    :group-by-page? false
    :collapsed? true}
   ]}
}
```

These set out some nice default queries where:

1. I can see the current tasks I am supposed to be doing right now (via "DOING" or "NOW" labels/markers), these could be going on the background and I'm supposed to be monitoring them or just tasks I'm doing in my designated timeslots (sorted by priority - which is specified by `[#A]` or `[#B]` or `[#C]` tags).
2. Tasks that I have scheduled or deadlined for today and they still need doing. This allows me to write a todo and just schedule it for a day where I have time to do it.
3. Tasks that were supposed to be completed before today and still have yet to be done
4. Tasks that are scheduled in the future (or are on journal days in the future).

These could definitely be improved but provided a nice base for me to work off of and get stuck into using logseq as my daily logging system. Additionally, unlike other methods that I have used, this takes little effort to actually manage, there are no planning meetings or selected work items, I just schedule a task to be revisted on a given day, if I cannot complete it I just reschedule it.

## What is Logseq V2?

Although logseq v1 provides a nice base, coming from AnyType you can see the rough edges. For example,

* The big one for me is asset management is quite basic, with the only way to tag assets and link them to specific projects or items, you have to create a highlights page.
* If you are not specifically using the advanced queries, it is quite hard to actually search for what you want and the interface is not particularly intuitive.
* When you type `/schedule` there is a good chance the text you just wrote will be deleted and you will have to rewrite it.
* No icons for pages, when you just want to add a bit of class (though these a plugin for that).
* The tagging system is quite basic, and only produces a list of pages on the website (and doesn't even list blocks).
* Builtin properties such as schedule break the usual formatting of properties.
* Putting `TODO` at the start of a block for the marker feels a bit hacky, and means when I want to change it to CANCELLED, in my queries I will get half way through writing when it just disappears.
* To sync your files to your phone you must use iCloud on iOS (which is not encrypted) or something like syncthing on android which is not easy for non-technical users.
* With much larger graphs, it starts to slow down.

This is where V2 comes in from logseq, it tries to solve a lot of these issues by moving towards storing all your notes in a sqlite database instead of in raw markdown. If you go to their website now and click download, this is the version you will download (even though the apps on the google play and apple app stores are the old version and are incompatible and its still in beta), I slightly disagree with this as it feels like Pop OS all over again but we will skim past this.

## The breaking changes to queries

However, as it is an official major version, there are some serious breaking changes and just regular changes in addition to a UI overhaul, luckily this is nicely listed in [their database version markdown file](https://github.com/logseq/docs/blob/master/db-version-changes.md):

* Advanced queries (on the page) no longer support `sort-by`, instead require you to use the sort parameters when the query appears.
* Tasks have been made legit, no longer using `markers`, and schedules and deadlines are proper priorities. And so properties are renamed:
  * `:block/marker` -> `:logseq.property/status`
  * `:block/scheduled` -> `:logseq.property/scheduled`
  * `:block/deadline` -> `:logseq.property/deadline`

  Additionally, as these are now database attributes, you must access them through: `(property h :logseq.property/deadline ?d)`
* `:today` no longer exists, nor does `:7d-after`, you now must instead specify `-end` and use `:+7d` or `:-7d`, so e.g. `:today-end` or `:today-start` or `:+7d-end` or `:-7d-start`... This one was definitely quite difficult to figure out given its not listed in the version doc.

Okay so simple we just update the queries to be of form:

```clojure
{:title "📅 Today"
  :query [:find (pull ?h [*])
        :in $ ?start ?next
        :where
  	(property ?h :logseq.property/status ?marker)
	  (or (property ?h :logseq.property/scheduled ?d)
    	  (property ?h :logseq.property/deadline ?d))
    [(contains? #{"Todo"} ?marker)]
    [(>= ?d ?start)]
    [(<= ?d ?next)]
  ]
  :inputs [:today-start :today-end]
```

But the default queries do not get the same luxury as advanced queries where you can change the sort in the interface, it seems you still have to define sort in the query itself.

Okay so we just add the translated `result-transform` right?

```clojure
{:title "📅 Today"
  :query [
    :find (pull ?h [*])
    :in $ ?start ?next
    :where
  	  (property ?h :logseq.property/status ?marker)
	    (or (property ?h :logseq.property/scheduled ?d)
    	    (property ?h :logseq.property/deadline ?d))
      [(contains? #{"Todo"} ?marker)]
      [(>= ?d ?start)]
      [(<= ?d ?next)]
  ]
  :inputs [:today-start :today-end]
  :result-transform
    (fn [result] (sort-by (fn [h] (get h :logseq.property/priority "B")) result))
```

But here is the issue, it results in an error. Wait so do I have to use `property` again? e.g.

```clojure
  :result-transform
    (fn [result] (sort-by (fn [h]
      (property h :logseq.property/priority ?prio)
      (get h ?prio "B")
    ) result))
```

No that doesn't work either, with specifically the error being along the lines `property is not defined` (yes I did have to open up the javascript console to start trying to figure out what the errors were). After doing some more research and experimentation, it seems that `property` can only be used within a database query operation as it requires access to the database... but we will get there.

## Debugging the queries

So the steps to debug a query in logseq are annoyingly difficult due to the fact that you don't really get any output or errors except within the JavaScript console. But I quickly found out that you can use `:result-transform` to print out some basic information.

So I setup an advanced query on a page so I could quickly interate, instead of going back and forth in `config.edn`.

```clojure
{
  :query [
    :find (pull ?row [*])
    :in $ ?start ?next
    :where
  	  (property ?row :logseq.property/status ?marker)
	    (or (property ?row :logseq.property/scheduled ?d)
    	    (property ?row :logseq.property/deadline ?d))
      [(contains? #{"Todo"} ?marker)]
      [(>= ?d ?start)]
      [(<= ?d ?next)]
  ]
  :inputs [:today-start :today-end]
  :result-transform
    (fn [result] (map (fn [r] {:prio (get-in r [:logseq.property/priority])} ) result))
}
```

Note here the use of `get-in` which effectively just expands to `(get (get r :prop-1) :prop-2)`. Although only using it for one attribute for now, you will see that it comes useful later on.

With this now you can see that it breaks the database table and outputs a list of attributes. With my test data I got:

```clojure
{:prio {:db/id 88}}
{:prio {:db/id 85}}
{:prio {:db/id 85}}
{:prio nil}
{:prio {:db/id 87}}
{:prio nil}
{:prio {:db/id 86}}
{:prio {:db/id 87}}
```

So we can now see that `:logseq.property/priority` is not sorting because it is an object. So technically we could sort with a transform like:

```clojure
  :result-transform
    (fn [result] (map (fn [r] {:prio (get-in r [:logseq.property/priority :db/id])} ) result))
```

But this feels a bit fickle and what happens if they are not always in the correct order by id?

Okay well how do we get access to its inner data? This is where I found out about `:keys` which acts similar to `SELECT` in sql:

```clojure
{
  :query [
    :find ?prio (pull ?row [*])
    :keys prio row
    :in $ ?start ?next
    :where
  	  (property ?row :logseq.property/status ?marker)
  	  (property ?row :logseq.property/priority ?prio)
	    (or (property ?row :logseq.property/scheduled ?d)
    	    (property ?row :logseq.property/deadline ?d))
      [(contains? #{"Todo"} ?marker)]
      [(>= ?d ?start)]
      [(<= ?d ?next)]
  ]
  :inputs [:today-start :today-end]
}
```

Which now outputted:

```clojure
{:prio "High", :row #uuid "6ad29385-aeff-44a1-ba7c-ce492b671625"}
{:prio "Medium", :row #uuid "6ad29375-a7cc-4242-a7cb-acabcf13cef5"}
{:prio "High", :row #uuid "6ad32da4-964d-4856-8c08-5572e0866788"}
{:prio "Low", :row #uuid "6acddc73-2d0b-494a-ab1b-99866a161b5b"}
{:prio "Low", :row #uuid "6acddc71-c60c-43ee-b61e-dc50e07f3d1c"}
{:prio "Urgent", :row #uuid "6acddc71-4b01-4ca0-b738-750bee4e9512"}
```

Wow look at that we got the priority, but can we now transform the row so we get a table again?

```clojure
:result-transform
  (fn [result] (map (fn [r] (get-in r [:row]) ) result))
```

With this now nothing is outputted. And all the other things I tried could not get logseq to detect it again and display a table.

Okay so what we have learned at this point?

* `:logseq.property/priority` is now a reference to a database row.
* We cannot pull from the database in `:result-transform`
* We can pass the information through with `:keys` but cannot then have logseq display it as a table.

### Side tangent: What does (pull) even mean?

Obviously this is getting the rows we want to look at from the database but what does it actually do?

From looking at logseq's dependencies we can find the query system they are using is [datascript](https://github.com/tonsky/datascript/blob/master/docs/queries.md) which then links to [datomic pull docs](https://docs.datomic.com/query/query-pull.html) as "similar syntax". As stated it is a way to describing "hierarchical (and possibly nested) selections".

So following their example, we can use this to limit what specifically we are selecting e.g.:

```clojure
{
  :query [
    :find (pull ?row [
      :logseq.property/status
      :logseq.property/scheduled
      :logseq.property/deadline
      :logseq.property/priority
    ])
    :in $ ?start ?next
    :where
  	  (property ?row :logseq.property/status ?marker)
	    (or (property ?row :logseq.property/scheduled ?d)
    	    (property ?row :logseq.property/deadline ?d))
      [(contains? #{"Todo"} ?marker)]
      [(>= ?d ?start)]
      [(<= ?d ?next)]
  ]
  :inputs [:today-start :today-end]
}
```

Now logseq only lists the selected columns as visible and does not suggest anymore. But if you add the result-transform from before you will notice that it is still listed with only `db/id` internally. So how do we get that information?

Here's where the hierarchical syntax comes into play, datomic has an example from below:

> ```clojure
> (d/pull db '[:track/name {:track/artists [:db/id :artist/name]}] ghost-riders)
> ```

This would suggest we can also pull additional attributes:

```clojure
{
  :query [
    :find (pull ?row [* {:logseq.property/priority [*]}])
    :in $ ?start ?next
    :where
  	  (property ?row :logseq.property/status ?marker)
	    (or (property ?row :logseq.property/scheduled ?d)
    	    (property ?row :logseq.property/deadline ?d))
      [(contains? #{"Todo"} ?marker)]
      [(>= ?d ?start)]
      [(<= ?d ?next)]
  ]
  :inputs [:today-start :today-end]
  :result-transform (fn [result] (map (fn [r] {:prio (get-in r [:logseq.property/priority])} ) result))
}
```

And now we get the result:

```clojure
{:prio #uuid "00000002-1996-4346-6700-000000000000"}
{:prio #uuid "00000002-2107-4537-4800-000000000000"}
{:prio #uuid "00000002-2107-4537-4800-000000000000"}
{:prio nil}
{:prio #uuid "00000002-5672-2766-8000-000000000000"}
{:prio nil}
{:prio #uuid "00000002-1829-3222-7800-000000000000"}
{:prio #uuid "00000002-5672-2766-8000-000000000000"}
```

Notice how the uuid's match on different attributes, this would suggest that we now have the UUID of the row for the priority. But how do we know the properties we have available?

Searching logseq's source code we find out a few:

* [`db/ident`](https://github.com/logseq/logseq/blob/32570a626aba699c089f20f9c940a6cb127c3add/src/test/frontend/components/property/value_test.cljs#L243) -- Identity used throught the code: `:logseq.property/priority.urgent`, `:logseq.property/priority.high`, `:logseq.property/priority.medium`, `:logseq.property/priority.low`
* [`logseq.property/icon`](https://github.com/logseq/logseq/blob/32570a626aba699c089f20f9c940a6cb127c3add/src/test/frontend/components/property/value_test.cljs#L244) - Some information about the icon itself

Then if you follow `logseq.property/icon` you can also find:

* [`block/order`](https://github.com/logseq/logseq/blob/32570a626aba699c089f20f9c940a6cb127c3add/src/main/frontend/handler/db_based/property/util.cljs#L48) -- random string which seems to help ordering
* [`block/title`](https://github.com/logseq/logseq/blob/32570a626aba699c089f20f9c940a6cb127c3add/src/main/frontend/handler/db_based/property/util.cljs#L46) -- Either "Urgent", "High", "Medium" or "Low"

I wasn't able to find any others but this should do what we need.

## How to sort by priority

Now that we have all the parts we can write a query that selects all the information from the priority and then sort based on the value

### The simple sort

If we wanted to just use `block/order` we can use:

```edn
  :sort-by-priority
    (fn [result] (reverse (sort-by (fn [h] (get-in h [:logseq.property/priority :block/order] "b09")) result)))
```

But I find this unreadable (especially with "b09") and needs a `reverse` which doesn't work when paring with other sorts.

### The robust sort

Instead we can use the map from looking at either `:block/title` or `:db/ident`. I personally prefer the `ident` as the title could change but the identity should not:

```clojure
  :sort-by-priority
    (fn [result] (sort-by (fn [h]
      (get
        {
          :logseq.property/priority.urgent 0,
          :logseq.property/priority.high 1,
          :logseq.property/priority.medium 2,
          :logseq.property/priority.low 3,
        }
        (get-in h [:logseq.property/priority :db/ident] :logseq.property/priority.medium)
      )
    ) result))
```

As you can see, we use `get` to map the priority identities to a number which can be changed to sort.

## Full translation

So pulling everything together we can finally fully translate the old queries to the new. Note that both `collapsed` and `group-by-page` have been removed as they were deprecated and no longer work as promised.

```clojure
{
  :query/result-transforms
  {
    :sort-by-priority
      (fn [result] (sort-by (fn [h]
        (get
          {
            :logseq.property/priority.urgent 0,
            :logseq.property/priority.high 1,
            :logseq.property/priority.medium 2,
            :logseq.property/priority.low 3,
          }
          (get-in h [:logseq.property/priority :db/ident] :logseq.property/priority.medium)
        )
      ) result))
    :sort-by-schedule-then-priority
      (fn [result] (sort-by (juxt 
        (fn [h] (get h :logseq.property/scheduled (get h :logseq.property/deadline 0)))
        (fn [h]
          (get
            {
              :logseq.property/priority.urgent 0,
              :logseq.property/priority.high 1,
              :logseq.property/priority.medium 2,
              :logseq.property/priority.low 3,
            }
            (get-in h [:logseq.property/priority :db/ident] :logseq.property/priority.medium)
          )
        )
      ) result))
    :sort-by-priority-then-schedule
      (fn [result] (sort-by (juxt 
        (fn [h]
          (get
            {
              :logseq.property/priority.urgent 0,
              :logseq.property/priority.high 1,
              :logseq.property/priority.medium 2,
              :logseq.property/priority.low 3,
            }
            (get-in h [:logseq.property/priority :db/ident] :logseq.property/priority.medium)
          )
        )
        (fn [h] (get h :logseq.property/scheduled (get h :logseq.property/deadline 0)))
      ) result))
   }

  :default-queries
  {:journals [
    {
      :title "🔨 NOW"
      :query [
        :find (pull ?h [* {:logseq.property/priority [*]}])
        :where
          (property ?h :logseq.property/status ?marker)
          [(contains? #{"Doing", "In Review"} ?marker)]
      ]
      :result-transform :sort-by-priority
    }

    {
      :title "📅 Today"
      :query [
        :find (pull ?h [* {:logseq.property/priority [*]}])
        :in $ ?start ?next
        :where
          (property ?h :logseq.property/status ?marker)
			    (or (property ?h :logseq.property/scheduled ?d)
            (property ?h :logseq.property/deadline ?d))
          [(contains? #{"Todo"} ?marker)]
          [(>= ?d ?start)]
          [(<= ?d ?next)]
      ]
      :inputs [:today-start :today-end]
      :result-transform :sort-by-priority
    }

    {
      :title "🔥 Overdue"
      :query [
        :find (pull ?h [* {:logseq.property/priority [*]}])
        :in $ ?start ?next
        :where
          (property ?h :logseq.property/status ?marker)
          (or (property ?h :logseq.property/scheduled ?d)
            (property ?h :logseq.property/deadline ?d))
          [(contains? #{"Doing", "Todo", "Backlog"} ?marker)]
          [(>= ?d ?start)]
          [(<= ?d ?next)]
      ]
      :inputs [:-365d-start :today-start]
      :result-transform :sort-by-priority-then-schedule
    }
 
    {
      :title "📅 NEXT"
      :query [
        :find (pull ?h [* {:logseq.property/priority [*]}])
        :in $ ?start ?next
        :where
          (property ?h :logseq.property/status ?marker)
          (or (property ?h :logseq.property/scheduled ?d)
            (property ?h :logseq.property/deadline ?d))
          [(contains? #{"Todo"} ?marker)]
          [(>= ?d ?start)]
          [(<= ?d ?next)]
      ]
      :inputs [:today-end :+7d-end]
      :result-transform :sort-by-schedule-then-priority
    }
  ]}
}
```

## Final notes

It has been quite an adventure trying to figure out the structure of the new query with only the database version file to go off of. Due to the lack of documentation around these changes and the outdated `:sort-by-priority` function that no longer works as requested I worry that they plan to replace or remove the default queries that have saved me so much, but this is thankfully **not** listed in [their public roadmap](https://logseq.io/page/3bc00ad3-f421-41e7-8c65-40861c298be5/6954ee2a-506b-4dd9-bd6d-0dc24db9c055).

It is definitely clear that it needs more work before I can fully switch over to V2, but from my experimentations and migrations it has helped me prepare for when I inevitably do as I do much prefer the asset organistion, simple queries and tagging system even if there is no longer a one click to mark a todo as done.

However I very much enjoyed this exploration, especially being introduced to clojure, which seems like an extremely interesting language with philosophies that I connect with as a Rust enthusiast.

Now I just have to figure out how to not have the default queries always collapsed...
