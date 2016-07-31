---
layout: post
title: "Syncing Shiny Modules"
description: ""
category: 
tags: [shiny]
---
{% include JB/setup %}

## Why Shiny Modules

So you might ask why use Shiny modules? 

- Namespace management
- All interactions with the container (or other modules) must be Explicit.
- Clearer development and maintenance.
- Promote reusability
- And had extremely high cool factor.

Convinced? If not, go through the tutorials in 

- [Modularzing Shiny](https://www.rstudio.com/resources/videos/modularizing-shiny-app-code/)
- [Shiny module design patterns: Pass module inputs to other modules](http://itsalocke.com/tag/shiny-design-patterns/)

## After taking the Tour

Ok, so after learning about Modules, and seeing how they cans solve
much of the messiness involved in developing and maintaining a Shiny
applications, it seemed only appropriate to try some simple experiments. 

Let’s assume that we all seen some examples of using modules which provide
support for performing the sequence of  selecting/reading and plotting data. 

But what about things not quite so linear?  That’s what I like to
investigate at here. But first a quick review.
 
## Quick Review

Shiny Modules consist of 2 components

- a UI component (named tinyUI below)
- a Server component (named tiny below)

If running the

- *UI/Server Configuration*, it’s best to place the module in a separate
file and source it in a file name *global.R*
- *app configuration*, probably want to just include in the app.file (as below) 

## A Minimalistic Example: Tiny

```
library(shiny)

# A module named Tiny
tinyUI <- function(id, input, output) { # module ui
  ns <- NS(id)
  tagList(radioButtons(ns("RB"), label="To Toggle", choices=letters[1:3]))
}
tiny<-function(input,output,session){} # module server (do nothing here)

# The shiny app 
shinyApp(
  fixedPage( tinyUI("toggle") ), #ui
  function(input, output, session) {callModule(tiny, "toggle" )} #server
)
```


![A tiny module](/assets/posts/2016-07-29-syncing-shiny-modules/tiny.png)



## The Tale of Two Toggles

In this section, in order to illustrate how to pass information between modules, we consider a somewhat contrived problem:

- two sets input radio buttons built as tiny Modules 
- both residing on the same page 
- both intended to do the same work. 

That means the the values for each
should be synched together. 

However, as a first step
let’s just create 2 instances of toggles. 

To this end we must add 2 instances of tinyUI to the ui shiny ui

```
fixedPage( tinyUI("Tom"), tinyUI("Tim") )
```

and to add two callModules for those instances to the server portion

```
  function(input, output, session) { #app server
    callModule(tiny, "Tom" )
    callModule(tiny, "Tim" )
  } 
```

Additionally, to distinguish our controls, lets change the
label in the tinyUI using the id

```
label=paste("Tiny", id)
```

Thus we have

```
tinyUI <- function(id, input, output) {
  ns <- NS(id)
  tagList(radioButtons(ns("RB"), label=paste("Tiny", id), choices=letters[1:3]))
}
tiny<-function(input,output,session){} # do nothing

# The shiny app 
shinyApp(
  fixedPage( tinyUI("Tom"), tinyUI("Tim") ), #ui
  function(input, output, session) {# app server
    callModule(tiny, "Tom" )
    callModule(tiny, "Tim" )
  } 
)
```

And upon running we see

![Two tinies](/assets/posts/2016-07-29-syncing-shiny-modules/tiny2.png)


However these two sets of toggles are not connected in anyway. Connecting them will require using reactives. Again the goal is

- make Toms choice propagate to Tim.
- make Tims choice propagate to Tom.

To do that
we modify the server portion of the tiny module to 

- add a new variable choice as a parameter to the tiny module.
- inside tiny, add aa observer for choice to update the choice of input$RB
- finally, return the value of input$RB as a reactive expression

```
tiny<-function(input,output,session, choice){ 
  observe({
    updateSelectInput( session, "RB",  selected=choice() )
  }) 
  reactive({input$RB})
} 
```

Next we edit the app server to take and return these choices.
That is, we want

- Tom to output his choice to Tim
- Tim to output his choice to Tom

So our app server looks like this

```
  function(input, output, session) {
    tomsChoice=callModule(tiny, "Tom", choice=timsChoice )
    timsChoice=callModule(tiny, "Tim", choice=tomsChoice )
  } 
```
The final result looks like


```
library(shiny)

#a module named tiny
tinyUI <- function(id, input, output) { 
  ns <- NS(id)
  tagList(radioButtons(ns("RB"), label=paste("Tiny", id), choices=letters[1:3]))
}
tiny<-function(input,output,session, choice){ 
  observe({
    updateSelectInput( session, "RB",  selected=choice() )
  }) 
  reactive({input$RB})
} 

# The shiny app 
shinyApp(
  fixedPage( tinyUI("Tom"), tinyUI("Tim") ), #app ui
  function(input, output, session) { # app server
    tomsChoice=callModule(tiny, "Tom", choice=timsChoice )
    timsChoice=callModule(tiny, "Tim", choice=tomsChoice )
  } 
)
```

Now if we change Tom to ‘b’, Tim immediately changes to ‘b’,
![Change Tom to b, Tim follows](/assets/posts/2016-07-29-syncing-shiny-modules/tiny3.png)

and if we change Tim to ‘c’, Tom immediately changes to ‘c’
![Change Tim to c, Tom follows](/assets/posts/2016-07-29-syncing-shiny-modules/tiny4.png)


## Keeping a navPanel in Sync

Now for a marginally  less artificial problem, Suppose we have the same
control appear on different views and we want the values to be in sync. 
For this example 

- Our will ui consists of a single navlistPanel
- Each tabPanel in our ui navListPanel will contain an instance of our module
- Each module will have a single selection control
- A change in the selection control of one module should propagate to all other modules.

Our strategy follows pretty much the same flow as the previous exercise but with one exception.

- We introduce a reactive expression involving a switch statement based 
on the value of the navBar’s current tab.

We do this because we needed a way to decide which reactive return ]should be used to 
always get the latest change.

```
library(shiny)
# Keeping the same ui module controls  on three different pages in sync.

aSelectModuleUI <- function(id, input, output) { 
  ns <- NS(id)
  tagList(
    selectInput(
      ns("selector"), 
      paste("tab", id, "is selected") , 
      choice=as.list(letters)
    )
  ) 
} 
aSelectModule<-function(input, output, session, choice ){
  observe({
      updateSelectInput( session, "selector",  selected=choice() )
  }) 
  reactive({input$selector})
} 

# The shiny app 
shinyApp(
  fixedPage( 
    navlistPanel(
      tabPanel("tab1", aSelectModuleUI("one")),
      tabPanel("tab2", aSelectModuleUI("two")),
      tabPanel("tab3", aSelectModuleUI("three")),
      id='navList'
    )
  ), #ui
  function(input, output, session) { # server
    # make a reactive expression to feed to all modules
    choice<-reactive({ 
      switch(input$navList, 
          tab1=choice1(),
          tab2=choice2(),
          tab3=choice3()
      )
    })
    # add to server each module, and assign to choiceX 
    # the reactive returned by moduleX
    choice1<-callModule(aSelectModule,  "one", choice)
    choice2<-callModule(aSelectModule,  "two", choice)
    choice3<-callModule(aSelectModule,  "three", choice)
  }
) 
```

Initially all values are *a*. In what follows I change the selector on tab *two*
to *d*

![Changing Two d](/assets/posts/2016-07-29-syncing-shiny-modules/select1.png)

Next we verify the value of tab3 has been updated to *d*

![Checking three](/assets/posts/2016-07-29-syncing-shiny-modules/select2.png)

And we also verify the value of tab1 has been updated to *d*

![Checking three](/assets/posts/2016-07-29-syncing-shiny-modules/select3.png)

One should note, that when the choice of the selector on the current tab changes, that value is propagated to all selectors. This may, or may not be desirable, depending on your particular application.





