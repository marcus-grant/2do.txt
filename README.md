`TODO.txt` CLI Client & Server
============================


Overview
--------

In my quest to become almost entirely self-hosted, I've decided to make a todo service that manages all of my tasks. There's already a great idea in this space ([and many many more][03]), where someone created a [plaintext todo.txt file format][01]. The reason this is a great idea is because it is a plain text file format that is sortable & filterable in the ways that are useful for to-do programs, simply by coming up with a format for entering information that just by alphabetic sorting and easy to search characters does all these things in most text editors. Because it is just plain text, it is easy to make it work on almost any computer system where the file is accessible. Meaning a flash drive, or Dropbox share is all that's needed to make the file available wherever you may be. Since this is for self-hosting, the first and maybe only distribution method for this service is to use with [Nextcloud][02], which is basically a Dropbox, *and then-some* that you can host on your own computers or hosting services.

The reason I'm not just using [Todo.txt][01] right off the get-go, is because their mobile apps are having issues with Dropbox's new API update that has stopped development on the apps for a while. Also, they implement their command line UI in shell script, and although I can modify shell scripts I prefer not to use shell script where possible, and I want to practice the [Go][04] language and customize my service to my specific use cases. And I have a very specific idea in mind for the mobile & web versions of this app in terms of design for most efficient workflows that no previous Todo.txt app I've found yet addresses.

Foreseeing that eventually this will have to become a mobile app, this repository will likely also contain an API server, written in Go, that will update & serve the relevant info in the todo.txt file, without sending the whole thing back and forth on the internet. This may not be all that necessary, because assuming each todo entry uses about 240 characters (*conservative estimate*) and there's 1000 todo entries *(right now I have just under 500)*, it would only take 240KB of data to transfer. However, if this must be done, every single time someone checks out the todo client using this service it might add up to problematic amounts of bandwidth, especially on mobile clients. We shall see.


CLI Client
----------

Before venturing any further with this self-hosted service, probably the best place to start is a CLI client that works with any given `TODO.txt` file. As a software developer who loves command line work for its speed and efficiency as an interface for textual information, and wanting to learn the intriguing new programming language, [Go][04] which is great for CLI utilities, this is probably the best place to start before the project grows too much in complexity.

At first, there will only be a `STDIN` & `STDOUT` program that takes commands and arguments as `STDIN` and then spits out results in the terminal's `STDOUT` and/or writes the results to the specified `TODO.txt` file. This will be handled with subcommands recognized by the terminal. There are several libraries in Go that are great for [input parsing][05], [making interactive prompts][06], [styling console outputs][07], [autocompleting commands & arguments in bash][08], [termbox-style UIs][09], [more minimal frameworks for cli UIs][10], and even [terminal dashboards][11]

Initially the command line is going to be relatively simple. Just using `STDIN` for input, probably using a [Go input parser][05] & printing to `STDOUT` using an [output styler][07]. After that works, probably the next step should be to make commands [autocomplete][08]. To make interactions more seamless.

First a `TODO.txt` parser should be created. This will have to conform to the rules of [`TODO.txt`][01] format. This will be a good place to conform to Test Driven Development practices, and using a [Go unit testing package][12]. Then a file writer/updater will need to be created to update the file. Also, some kind of local configuration file should be used so entering the text file location won't be necessary.


A Simple Improvement to the TODO.txt Specification
--------------------------------------------------

Due dates for tasks are at least as important as the priority of a task. It certainly is more important than the creation date of a task, when sorting incomplete tasks. The way the specification is defined right now, **completion** dates are only supposed to appear in front of a **creation** date when a task is completed. This means that incomplete tasks will be plaintext sorted by priority first and creation dates second. This isn't terribly useful when planing out tasks. A small change to the specification would change this.

### Rule 2: A task's creation date may optionally appear directly after priority and a space

### Rule 3: A task's due date is the first of two dates appearing directly after priority or as the first two tags on a line

* A due date takes the form of `YYYY-MM-DD`, and is followed by a space, just like a creation date.
* The second of two adjacent dates after a priority, occurring either first in the line, or directly after a priority and space is now the **creation date**
* The first of the same two dates after a priority, occurring either first in the line, or directly after a priority and space is now the **due date**
* With two adjacent dates as either the first text in a line or after a priority, the **creation** date specified in **Rule 2** is now the second date.
* This allows tasks to first be sorted by a priority *if it exists*, then by creation *if it exists*, then finally by due date *if it exists*.
* Tasks with **creation**, but not **due** dates will appear before tasks that have **due** dates, which will call attention to tasks that require scheduling.
* If a task was specified without a **creation** date and are later given **due** dates will be given **creation** dates of the day the **due** is given.
* This creates the possibility that **due** dates chronologically precedes **creation** dates
  * It should be up to the developer to decide how to handle this:

These tasks have due dates:

```
2017-05-11 2017-05-14 +home-improvement @tidying Clean toilet
(A) 2018-03-28 2018-03-29 Conquer the world @self-improvement
```

These tasks do not have due dates:

```
2017-05-11 Buy train tickets going anywhere +travel @anywhere
(B) 2017-12-31 Party like it's 2018 @nye
(A) Learn to moonwalk 2016-04-23 2016-04-14 +dancing
```

### Rule 4: Same as previous rule 2

### Rule 2 (in completed tasks): The date of completion appears directly after the x, separated by a space.

* The **completion** date must appear directly after an `x` & a space
* The optional second date appears directly after the **completion** date and represents the **creation** date, if there is one.
* The **due** date is no longer important as the **completion** and **creation** date
* So the first date which could've been either a **due** date or a **creation** date before being completed, now becomes the **completion** date.
* If a second date appears directly after the **completion** date, it is the **creation** date.
* If a **due** date was the first date before the task is completed, it is now removed entirely because it is no longer necessary to track.
* If the due date should be remembered even when completed, put it in a **key:value** tag, (e.g. `due:2014-09-03`).
* Same as before, priority is no longer tracked, and so is removed.
* If the priority should be remembered after completion, store it in a **key:value** tag (e.g. `pri:A`).


API Server
----------

Two reasons exist for implementing an API server that holds the central copy of the file:

1. For this to be useful to me in every day use as fast as possible, I'll need apps on my phone/tablet that can view & edit the file. If by using Nextcloud it can sync the file onto the phone easily, this will be a moot point. However, just sharing it won't be enough, there will have to be some kind of editor that can edit the synced copy with at least a search and filter feature.

2. For the todo clients to be useful a certain degree of reliability is needed, and I'm not sure my *(or anyone else's)* ISP is going to be reliable enough to host the file to the internet. I don't want to make a web server on the cloud have any kind of access to my Nextcloud server for security reasons, and I don't have time initially to implement the kind of integration needed for it to work with Dropbox until the mobile app is pretty far along.

3. Mobile apps would need to send the whole file back and forth whenever being opened or modifying the list. Although the aforementioned 240KB isn't a whole lot of data to pass back & forth by itself, having it do so on every opening of the app or app changing the list could quickly add up to expensive phone bills. An API could instead use a diffing algorithm to only send the changes.

4. I do a lot of work with APIs in home automation, web services, app development and it is a lot easier to interact with data when RESTful or query based APIs stand in between the data and the endpoint.

This will be investigated further as the project goes along. If the WebDAV protocol that Nextcloud uses allows for atomic changes, which it seems like it might, and using it to host the file is reliable enough on my ISP the API server might just be ignored altogether.


Web Client
----------

The web client is going to be made in React, because that's what I'm good at. And I'm good at it, because I happen to think it's the best front-end framework for the web. This is because it is fast, well supported, can be used to make apps work just as well as native mobile & desktop apps, and has one of the best architectures I've seen on the web. *Vue might be better architecturally, but the other benefits outdo it in my view*. That's not to put down experts in other frameworks, there's tons of good reasons to use Vue, Angular, etc. but for me React makes the most sense in this situation.

These are the most features I want implemented quickly:

1. Focus on keyboard entry:
    * Todoist does this *mostly* well, and that was my favorite To-Do app
    * Keyboard interaction will almost always be faster than mouse
2. Parsing of task input for keywords that add metadata to the task:
    * Again todoist did this and I loved them for it
    * This makes entering tasks **much** faster
    * `!!A tomorrow Fix To-Do App @sideproject @react +todo.txt`
      * the above would add priority `A`, due date `tomorrow`, and a bunch of contexts and projects
3. Guessing the next words, or metadata applied by using machine learning applied to the `TODO.txt` file and reading input fields on the client.
    * Speed is one of the biggest factors in me choosing a todo app, you have to be able to capture ideas and tasks ASAP for a To-Do client to be useful to me
4. Quick deferral and metadata editing UIs
    * Todoist, again was great here, swiping on the task in the mobile version brought up a quick modal view to move the task's due date by a day, week, etc.
    * Any.Do had a great interface that goes through the day one task at a time and let you reschedule, commit to doing the task, edit other metadata like tags, and schedule on a calendar/reminder app
5. A today/week view to quickly see the things that need to be done
    * When planning the tasks that need to be done this is really useful to plan your day, and to change plans if needed

Many of these things should also be implemented in the command line client because that's where I do the most work and can view/modify tasks the quickest. This will be harder to implement in an intuitive way, and may require a more interactive interface like [termbox-go][09] or [gocui][10].


Mobile Client
-------------

The mobile client is where this project gets the most tough. React Native makes it much easier, but because of the closed environment of iOS, and the data costs of mobile internet, and the differences that still need to be addressed on the UI front between Android & iOS, it will require much more work. The biggest bits of research to decide how this turns out will be how useful Nextcloud is in iOS 11 with its new integrated file manager, and how it syncs files. If file syncing with Nextcloud can be made atomic and easy with external iOS apps and doesn't cause any integration problems then the task becomes much easier and can be carried out before creating an API server. Besides the file syncing problem, the rest of the development will just be about making the web version of the client more applicable to a mobile app context.


References
----------

1. [Github: todotxt/todo.txt - format][01]
2. [Nextcloud.com: Official Homepage][02]
3. [FreeCodeCamp (Medium): Everytime You Build a To-Do App a Puppy Dies][03]
4. [Golang.org: Official Go Programming Language Homepage][04]
5. [Github: mkideal/cli - A Package for Building CLI Apps with Go][05]
6. [Github: c-bara/go-prompt - Build Powerful Interactive Prompts in Go][06]
7. [Github: ttacon/chalk - A Go Package for Styling Console Output][07]
8. [Github: posener/complete - Bash Completion with Go & Bash][08]
9. [Github: nsf/termbox-go - Pure Go Termbox Implementation][09]
10. [Github: jroimartin/gocui - Minimalist Go Module for Creating Console UIs][10]
11. [Github: gizak/termui - Go Library for Terminal UIs Dashboards][11]
12. [GoConvey: Write Go Tests in Editor, Get Live Results in Browser (Homepage)][12]

[01]: https://github.com/todotxt/todo.txt "Github: todotxt/todo.txt - format"
[02]: https://nextcloud.com/ "Nextcloud.com: Official Homepage"
[03]: https://medium.freecodecamp.org/every-time-you-build-a-to-do-list-app-a-puppy-dies-505b54637a5d "FreeCodeCamp (Medium): Every Time You Build a To-Do App a Puppy Dies"
[04]: https://golang.org/ "Golang.org: Official Go Programming Language Homepage"
[05]: https://github.com/mkideal/cli "Github: mkideal/cli - A Package for Building CLI Apps with Go"
[06]: https://github.com/c-bata/go-prompt "Github: c-bara/go-prompt - Build Powerful Interactive Prompts in Go"
[07]: https://github.com/ttacon/chalk "Github: ttacon/chalk - A Go Package for Styling Console Output"
[08]: https://github.com/posener/complete "Github: posener/complete - Bash Completion with Go & Bash"
[09]: https://github.com/nsf/termbox-go "Github: nsf/termbox-go - Pure Go Termbox Implementation"
[10]: https://github.com/jroimartin/gocui "Github: jroimartin/gocui - Minimalist Go Module for Creating Console UIs"
[11]: https://github.com/gizak/termui "Github: gizak/termui - Go Library for Terminal UIs Dashboards"
[12]: http://goconvey.co/ "GoConvey: Write Go Tests in Editor, Get Live Results in Browser (Homepage)"
