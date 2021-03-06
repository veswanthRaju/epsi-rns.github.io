---
layout: post
title:  "HerbstluftWM Idle Event in Haskell"
date:   2017-06-18 17:35:15 +0700
categories: desktop
tags: [coding, haskell, herbstluftwm, statusbar]
author: epsi

excerpt:
  HersbtluftWM Event Idle using Dzen2 or Lemonbar.
  Modularized implementation in Haskell language.
  An advance case of Pipe and Fork.

related_link_ids: 
  - 17061135  # Event Idle Overview
  - 17061235  # Event Idle BASH
  - 17061335  # Event Idle Perl
  - 17061435  # Event Idle Python
  - 17061535  # Event Idle Ruby
  - 17061635  # Event Idle PHP
  - 17061735  # Event Idle Lua
  - 17061835  # Event Idle Haskell

---

### Preface

> Goal: Manage Herbstclient process trigerred by idle event

	Focusing in "herbstclient --idle". 

![HerbstluftWM: Tag Status][image-hlwm-01-event-idle]{: .img-responsive }

This is the next Part, of the previous Tutorial.
This tutorial cover Lemonbar.
In order to use Dzen2, any reader could use the source code in github.

-- -- --

#### Reading

Before you jump off to scripting,
you might desire to read this overview.

*	[HerbstluftWM Tag Status Overview][local-overview]

*	[Modularized HerbstluftWM in Haskell][local-haskell-config]

*	[Piping and Forking in Haskell][local-haskell-pipe]

#### All The Source Code:

Impatient coder like me, like to open many tab on browser.

*	**Dzen2**: 
	[github.com/.../dotfiles/.../haskell/][dotfiles-dzen2-haskell]

*	**Lemonbar**: 
	[github.com/.../dotfiles/.../haskell/][dotfiles-lemon-haskell]

#### The PipeHandler Source File:

Let's have a look at <code class="code-file">pipehandler.hs</code> in github.

*	**Dzen2**: 
	[github.com/.../dotfiles/.../haskell/pipehandler.hs][dotfiles-dzen2-haskell-pipehandler]

*	**Lemonbar**: 
	[github.com/.../dotfiles/.../haskell/pipehandler.hs][dotfiles-lemon-haskell-pipehandler]

-- -- --

{% include post/2017/06/herbstlustwm-event-idle-language.md %}

-- -- --

### Statusbar Screenshot

#### Dzen2

![Statusbar: Dzen2 Screenshot][image-hlwm-ss-dzen2]{: .img-responsive }

#### Lemonbar

![Statusbar: Lemonbar Screenshot][image-hlwm-ss-lemon]{: .img-responsive }

-- -- --

### Without Idle event

Let's have a look at our main 
<code class="code-file">panel.hs</code> in github.
At the end of the script, we finally call lemonbar
with <code>detachLemon</code> function.

{% highlight haskell %}
main = do
    ...

    -- remove all lemonbar instance
    system "pkill lemonbar"

    -- run process in the background
    detachLemon monitor lemonParameters
{% endhighlight %}

#### View Source File:

*	**Dzen2**: 
	[github.com/.../dotfiles/.../haskell/panel.hs][dotfiles-dzen2-haskell-panel]

*	**Lemonbar**: 
	[github.com/.../dotfiles/.../haskell/panel.hs][dotfiles-lemon-haskell-panel]


#### Run Lemon, Run !

This <code>detachLemon</code> function.
is just a function that enable 
the lemonbar running process to be detached,
using <code>forkProcess()</code>.

{% highlight haskell %}
detachLemon :: Int -> [String] -> IO ProcessID
detachLemon monitor parameters = forkProcess 
    $ runLemon monitor parameters
{% endhighlight %}

The real function is <code>runLemon</code>.
You must be familiar with this <code>createProcess</code>.

{% highlight haskell %}
runLemon :: Int -> [String] -> IO ()
runLemon monitor parameters = do
    let command_out = "lemonbar"

    (Just pipe_lemon_in, _, _, ph) <- 
        createProcess (proc command_out (parameters ++ ["-p"])) 
        { std_in = CreatePipe }

    contentInit monitor pipe_lemon_in    
    hClose pipe_lemon_in
{% endhighlight %}

Note: that we want to ignore idle event for a while.
And append the <code>-p</code> for a while,
to make the statusbar persistent.

#### Statusbar Initialization

Here we have the <code>contentInit</code>.
It is just an initialization of global variable.
We are going to have some loop later in different function,
to do the real works.

{% highlight haskell %}
contentInit :: Int -> Handle -> IO ()
contentInit monitor pipe_lemon_in = do
    setTagValue monitor 
    setWindowtitle ""
    
    text <- getStatusbarText monitor

    hPutStrLn pipe_lemon_in text
    hFlush pipe_lemon_in
{% endhighlight %}

Now is time to try the panel, on your terminal.
**Note**: that we already reach this stage in our previous article.
These two functions, <code>setTagValue</code>
and <code>setWindowtitle</code>, have already been discussed.

#### View Source File:

Simple version. No idle event. Only statusbar initialization.

*	**Lemonbar**: 
	[github.com/.../dotfiles/.../haskell/MyPipeHandler.01-init.hs][dotfiles-lemon-haskell-pipehandler-init]

{% include post/2017/06/herbstlustwm-event-idle-pipehandler.md %}

-- -- --

### With Idle event

Consider this <code>contentWalk</code> call,
after <code>contentInit</code> call,
inside the <code>runLemon</code>.

{% highlight haskell %}
runLemon :: Int -> [String] -> IO ()
runLemon monitor parameters = do
    let command_out = "lemonbar"

    (Just pipe_lemon_in, _, _, ph) <- 
        createProcess (proc command_out parameters) 
        { std_in = CreatePipe }

    contentInit monitor pipe_lemon_in
    contentWalk monitor pipe_lemon_in  -- loop for each event
    
    hClose pipe_lemon_in
{% endhighlight %}

#### Wrapping Idle Event into Code

<code>contentWalk</code> is the **heart** of this script.
We have to capture every event,
and process the event in event handler.

	Walk step by step, Process event by event

After the event handler,
we will get the statusbar text, in the same way,
we did in <code>content_init</code>.

{% highlight haskell %}
contentWalk :: Int -> Handle -> IO ()
contentWalk monitor pipe_lemon_in = do
    let command_in = "herbstclient"

    (_, Just pipe_idle_out, _, ph) <- 
        createProcess (proc command_in ["--idle"]) 
        { std_out = CreatePipe }

    forever $ do
        -- wait for next event 
        event <- hGetLine pipe_idle_out 
        handleCommandEvent monitor event
 
        text <- getStatusbarText monitor

        hPutStrLn pipe_lemon_in text
        hFlush pipe_lemon_in

    hClose pipe_idle_out
{% endhighlight %}

-- -- --

### The Event Handler

For each idle event, there are multicolumn string.
The first string define the event origin.

![HerbstluftWM: Tag Status][image-hlwm-04-event-origin]{: .img-responsive }

The origin is either <code>reload</code>, or <code>quit_panel</code>,
<code>tag_changed</code>, or <code>tag_flags</code>, 
or <code>tag_added</code>, or <code>tag_removed</code>,
or <code>focus_changed</code>, or <code>window_title_changed</code>.
More complete event, can be read in herbstclient manual.

All we need is to pay attention to this two function.
<code>set_tag_value</code> and <code>set_windowtitle</code>.

{% highlight haskell %}
getColumnTitle :: [String] -> String
getColumnTitle column
  | length(column) > 2 = column !! 2
  | otherwise          = ""

handleCommandEvent :: Int -> String -> IO ()
handleCommandEvent monitor event
  | origin == "reload"      = do system("pkill lemonbar"); return ()
  | origin == "quit_panel"  = do exitSuccess; return ()
  | elem origin tagCmds     = do setTagValue monitor
  | elem origin titleCmds   = do setWindowtitle $ getColumnTitle column
  where
    tagCmds   = ["tag_changed", "tag_flags", "tag_added", "tag_removed"]
    titleCmds = ["window_title_changed", "focus_changed"]

    -- find out event origin
    column = splitOn "\t" event
    origin = column !! 0
{% endhighlight %}

Actually that's all we need to have a functional lemonbar.
This is the minimum version.

#### View Source File:

With idle event. The **heart** of the script.

*	**Lemonbar**: 
	[github.com/.../dotfiles/.../haskell/MyPipeHandler.02-idle.hs][dotfiles-lemon-haskell-pipehandler-idle]

-- -- --

### Lemonbar Clickable Areas

This is specific issue for lemonbar,
that we don't have in dzen2.

Consider have a look at 
<code class="code-file">output.hs</code>.

{% highlight haskell %}
    -- clickable tags
    textName  = "%{A:herbstclient focus_monitor \"" 
        ++ show(monitor) ++ "\" && " ++ "herbstclient use \"" 
        ++ tagIndex ++ "\":} " ++ tagName ++ " %{A} "
{% endhighlight %}

**Issue**: Lemonbar put the output on terminal
instead of executing the command.

![HerbstluftWM: Tag Status][image-hlwm-05-clickable]{: .img-responsive }

Consider going back to
<code class="code-file">pipehandler.hs</code>.

We need to pipe the lemonbar output to shell.
It means Lemonbar read input and write output at the same time.
<code>createProcess</code> does good with bidirectional pipe.

{% highlight haskell %}
runLemon :: Int -> [String] -> IO ()
runLemon monitor parameters = do
    let command_out = "lemonbar"

    (Just pipe_lemon_in, Just pipe_lemon_out, _, ph) <- 
        createProcess (proc command_out parameters) 
        { std_in = CreatePipe, std_out = CreatePipe }

    (_, _, _, ph) <- 
        createProcess (proc "sh" []) 
        { std_in = UseHandle pipe_lemon_out }

    contentInit monitor pipe_lemon_in
    contentWalk monitor pipe_lemon_in  -- loop for each event
    
    hClose pipe_lemon_in
    hClose pipe_lemon_out
{% endhighlight %}

#### How does it work ?

	UseHandle take care of this.

{% highlight haskell %}
        { std_in = UseHandle pipe_out }
{% endhighlight %}

#### View Source File:

Piping lemonbar output to shell, implementing lemonbar clickable area.

*	**Lemonbar**: 
	[github.com/.../dotfiles/.../haskell/MyPipeHandler.03-clickable.hs][dotfiles-lemon-haskell-pipehandler-clickable]

-- -- --

### Interval Based Event

We can put custom event other than idle event in statusbar panel.
This event, such as date event, called based on time interval in second.

It is a little bit tricky, because we have to make, 
a combined event that consist of,
idle event (asynchronous) and interval event (synchronous).
Merging two different paralel process into one.

This is an overview of what we want to achieve.

![HerbstluftWM: Custom Event][image-hlwm-06-event-custom]{: .img-responsive }

In real code later, we do not need the timestamp.
<code>interval</code> string is enough to trigger interval event.

#### View Testbed Source File:

Before merging combined event into main code,
consider this test in an isolated fashion.

*	[github.com/.../dotfiles/.../haskell/11-testevents.hs][dotfiles-lemon-haskell-testevents]

-- -- --

### Combined Event

#### Preparing The View

This is what it looks like, an overview of what we want to achieve.

![Statusbar: Event Screenshot][image-hlwm-ss-event]{: .img-responsive }

Consider make a progress in 
<code class="code-file">output.hs</code>.

{% highlight haskell %}
module MyOutput ( ..., setDatetime, ...) where

segmentDatetime :: IORef String
segmentDatetime = unsafePerformIO $ newIORef ""    -- empty string
 
wFormatTime :: FormatTime t => t -> String -> String
wFormatTime myUtcTime myTimeFormat = formatTime 
    Data.Time.Format.defaultTimeLocale myTimeFormat myUtcTime

getStatusbarText :: Int -> IO String
getStatusbarText monitor = do
    tags <- readIORef tagsStatus
    
    let tagText = "%{l}" ++ (join $ map (outputByTag monitor) tags)
    timeText  <- ("%{c}" ++) <$> outputByDatetime
    titleText <- ("%{r}" ++) <$> outputByTitle

    let text = tagText ++ timeText ++ titleText
    return text

outputByDatetime :: IO String
outputByDatetime = do
    segment <- readIORef segmentDatetime
    return segment

formatDatetime :: ZonedTime -> String
formatDatetime now = dateText ++ "  " ++ timeText
  where
    ...

setDatetime :: IO ()
setDatetime = do
    now <- getZonedTime     
    writeIORef segmentDatetime $ formatDatetime now
{% endhighlight %}

And a few enhancement in 
<code class="code-file">MyPipeHandler.hs</code>.

{% highlight haskell %}
wSleep :: Int -> IO ()
wSleep mySecond = threadDelay (1000000 * mySecond)

handleCommandEvent :: Int -> String -> IO ()
handleCommandEvent monitor event
  ...
  | origin == "interval"    = do setDatetime
  where
    ...

contentInit :: Int -> Handle -> IO ()
contentInit monitor pipe_lemon_in = do
    ... 
    setWindowtitle ""
    setDatetime

    ...
{% endhighlight %}

#### Expanding The Event Controller

All we need to do is to split out <code>content_walk</code> into

*	<code>content_walk</code>: combined event,
	with the help of <code>cat</code> process.

*	<code>content_event_idle</code>: HerbstluftWM idle event. 
	Forked, as background processing.

*	<code>content_event_interval</code> : Custom date time event. 
	Forked, as background processing.

{% highlight haskell %}
contentEventIdle :: Handle -> IO ()
contentEventIdle pipe_cat_in = do
    let command_in = "herbstclient"

    (_, Just pipe_idle_out, _, ph) <- 
        createProcess (proc command_in ["--idle"]) 
        { std_out = CreatePipe }

    forever $ do
        -- wait for next event 
        event <- hGetLine pipe_idle_out

        hPutStrLn pipe_cat_in event
        hFlush pipe_cat_in

    hClose pipe_idle_out
{% endhighlight %}

{% highlight haskell %}
contentEventInterval :: Handle -> IO ()
contentEventInterval pipe_cat_in = forever $ do
     hPutStrLn pipe_cat_in "interval"
     hFlush pipe_cat_in

     wSleep 1
{% endhighlight %}

{% highlight haskell %}
contentWalk :: Int -> Handle -> IO ()
contentWalk monitor pipe_lemon_in = do
    (Just pipe_cat_in, Just pipe_cat_out, _, ph) <- 
        createProcess (proc "cat" []) 
        { std_in = CreatePipe, std_out = CreatePipe }

    forkProcess $ contentEventIdle(pipe_cat_in)
    forkProcess $ contentEventInterval(pipe_cat_in)
    
    forever $ do
        -- wait for next event 
        event <- hGetLine pipe_cat_out
        handleCommandEvent monitor event
 
        text <- getStatusbarText monitor

        hPutStrLn pipe_lemon_in text
        hFlush pipe_lemon_in

    hClose pipe_cat_out
    hClose pipe_cat_in
{% endhighlight %}

This above is the most complex part.
We are almost done.

#### View Source File:

Combined event consist of both,
synchronous interval event and asynchronous idle event.

*	**Lemonbar**: 
	[github.com/.../dotfiles/.../haskell/MyPipeHandler.04-event.hs][dotfiles-lemon-haskell-pipehandler-event]

-- -- --

### Dual Bar

The idea of this article comes from the fact
that <code>herbsclient --idle</code> is asynchronous event.
If you need another bar, just simply use <code>Conky</code> instead.

*	**Dzen2**: 
	![HerbstluftWM: Dzen2 Conky][image-hlwm-ss-dzen2-conky]{: .img-responsive }

*	**Lemonbar**: 
	![HerbstluftWM: Lemonbar Conky][image-hlwm-ss-lemon-conky]{: .img-responsive }

We only need one function to do this in
<code class="code-file">pipehandler.pm</code>.

{% highlight perl %}
detachLemonConky :: [String] -> IO ()
detachLemonConky parameters = do
    -- Source directory is irrelevant in Haskell
    -- but we'll do it anyway for the sake of learning
    dirName <- getCurrentDirectory
    let conkyFileName = dirName ++ "/../conky" ++ "/conky.lua" 

    (_, Just pipeout, _, _) <- 
        createProcess (proc "conky" ["-c", conkyFileName])
        { std_out = CreatePipe } 

    (_, _, _, ph)  <- 
        createProcess (proc "lemonbar" parameters) 
        { std_in = UseHandle pipeout }
      
    hClose pipeout
{% endhighlight %}

And execute the function main script in
<code class="code-file">panel.pl</code>.

{% highlight perl %}
-- This is a modularized config for herbstluftwm tags in lemonbar

import System.Environment
import System.Process

import MyHelper
import MyPipeHandler

-- initialize

panelHeight = 24

-- ----- ----- ----- ----- ----- ----- ----- ----- ----- ----- -----
-- main

main = do
    args <- getArgs
    let monitor = getMonitor args

    geometry <- getGeometry monitor

    system "pkill lemonbar"
    system $ "herbstclient pad " ++ show(monitor) ++ " "
        ++ show(panelHeight) ++ " 0 " ++ show(panelHeight) ++ " 0"

    -- run process in the background

    let paramsTop = getParamsTop panelHeight geometry
    detachLemon monitor paramsTop

    let paramsBottom = getParamsBottom panelHeight geometry
    detachLemonConky paramsBottom

    -- end of IO
    return ()
{% endhighlight %}

#### View Source File:

Dual Bar, <code>detach_lemon_conky</code> function.

*	**Lemonbar**: 
	[github.com/.../dotfiles/.../haskell/MyPipeHandler.05-conky.hs][dotfiles-lemon-haskell-pipehandler-conky]

-- -- --

### Avoid Zombie Apocalypse

Zombie are scary, and fork does have a tendecy to become a zombie.
Application that utilize several forks should be aware of this threat.
The reason why I use fork instead of thread is,
because the original herbstluftwm configuration coming from bash,
and this bash script is using fork.

However, you can use this short script to reduce zombie population.
It won't kill all zombie, but works for most case.
You might still need <code>htop</code>,
and <code>kill -9</code> manually.

{% highlight haskell %}
killZombie :: IO ()
killZombie = do
    system "pkill -x dzen2"
    system "pkill -x lemonbar"
    system "pkill -x cat"
    system "pkill conky"
    system "pkill herbstclient"
    
    return ()
{% endhighlight %}

-- -- --

### Putting Them All Together

I also created compact for version,
for use with main HerbstluftWM configuration,
in <code class="code-file">~/.config/herbstluftwm/</code> directory.
After reunification, they are not very long scripts after all.

*	**Dzen2**: 
	[github.com/.../dotfiles/.../haskell/panel-dzen2.hs][dotfiles-hlwm-haskell-dzen2-compact]

*	**Lemonbar**: 
	[github.com/.../dotfiles/.../haskell/panel-lemonbar.hs][dotfiles-hlwm-haskell-lemon-compact]

{% include post/2017/06/herbstlustwm-event-idle-panel.md %}

-- -- --

#### Desktop Screenshot

Fullscreen, Dual Panel, Zero Gap.

[![HerbstluftWM: Screenshot Dual Panel][image-ss-hlwm-dualpanel]{: .img-responsive }][photo-ss-hlwm-dualpanel]

-- -- --

Enjoy the statusbar !
Enjoy the window manager !


[//]: <> ( -- -- -- links below -- -- -- )

{% assign asset_path = site.url | append: '/assets/posts/desktop/2017/06' %}
{% assign dotfiles_dzen2 = 'https://github.com/epsi-rns/dotfiles/blob/master/standalone/dzen2-hlwm' %}
{% assign dotfiles_lemon = 'https://github.com/epsi-rns/dotfiles/blob/master/standalone/lemon-hlwm' %}
{% assign dotfiles_hlwm  = 'https://github.com/epsi-rns/dotfiles/blob/master/herbstluftwm' %}

[image-hlwm-01-event-idle]:   {{ asset_path }}/herbstclient-01-event-idle.png
[image-hlwm-02-tag-status]:   {{ asset_path }}/herbstclient-02-tag-status.png
[image-hlwm-04-event-origin]: {{ asset_path }}/herbstclient-04-event-origin.png
[image-hlwm-05-clickable]:    {{ asset_path }}/herbstclient-05-lemonbar-clickable-areas.png
[image-hlwm-06-event-custom]: {{ asset_path }}/herbstclient-06-event-custom.png

[image-hlwm-ss-dzen2]: {{ asset_path }}/hlwm-dzen2-ss.png
[image-hlwm-ss-lemon]: {{ asset_path }}/hlwm-lemon-ss.png
[image-hlwm-ss-event]: {{ asset_path }}/hlwm-event-ss.png
[image-hlwm-ss-dzen2-conky]: {{ asset_path }}/hlwm-dzen2-conky-ss.png
[image-hlwm-ss-lemon-conky]: {{ asset_path }}/hlwm-lemon-conky-ss.png

[image-ss-hlwm-dualpanel]: {{ asset_path }}/herbstluftwm-dualpanel.png
[photo-ss-hlwm-dualpanel]: https://photos.google.com/share/AF1QipMO53TtSJVXrkn8R0s4wre4QWgX7_G5CoaSkFMneVHFp9Tu5STBmdjW3M3fpA2eEw/photo/AF1QipPqMNt9e3_UypKHqASPs_njHBQPX7Kn8X_O9aTp?key=WGIySDVOaVpibkJCRkV5NWVZUUs3UnNLNHR1MVpn

[dotfiles-lemon-haskell-testevents]:  {{ dotfiles_lemon }}/haskell/11-testevents.hs
[dotfiles-hlwm-haskell-dzen2-compact]: {{ dotfiles_hlwm }}/haskell/panel-dzen2.hs
[dotfiles-hlwm-haskell-lemon-compact]: {{ dotfiles_hlwm }}/haskell/panel-lemonbar.hs

[local-haskell-config]: {{ site.url }}/desktop/2017/05/03/herbstlustwm-modularized-haskell.html
[local-haskell-pipe]:   {{ site.url }}/code/2017/04/16/haskell-pipe-and-fork.html

[local-bash-config]: {{ site.url }}/desktop/2017/05/02/herbstlustwm-modularized-bash.html
[local-bash-pipe]:   {{ site.url }}/code/2017/04/15/bash-pipe-and-fork.html

[local-overview]: {{ site.url }}/desktop/2017/06/11/herbstlustwm-event-idle-overview.html

[dotfiles-lemon-haskell-pipehandler-init]:      {{ dotfiles_lemon }}/haskell/MyPipeHandler.01-init.hs
[dotfiles-lemon-haskell-pipehandler-idle]:      {{ dotfiles_lemon }}/haskell/MyPipeHandler.02-idle.hs
[dotfiles-lemon-haskell-pipehandler-clickable]: {{ dotfiles_lemon }}/haskell/MyPipeHandler.03-clickable.hs
[dotfiles-lemon-haskell-pipehandler-event]:     {{ dotfiles_lemon }}/haskell/MyPipeHandler.04-event.hs
[dotfiles-lemon-haskell-pipehandler-conky]:     {{ dotfiles_lemon }}/haskell/MyPipeHandler.05-conky.hs

[dotfiles-dzen2-haskell-panel]:       {{ dotfiles_dzen2 }}/haskell/panel.hs
[dotfiles-dzen2-haskell-helper]:      {{ dotfiles_dzen2 }}/haskell/MyHelper.hs
[dotfiles-dzen2-haskell-output]:      {{ dotfiles_dzen2 }}/haskell/MyOutput.hs
[dotfiles-dzen2-haskell-pipehandler]: {{ dotfiles_dzen2 }}/haskell/MyPipeHandler.hs

[dotfiles-lemon-haskell-panel]:       {{ dotfiles_lemon }}/haskell/panel.hs
[dotfiles-lemon-haskell-helper]:      {{ dotfiles_lemon }}/haskell/MyHelper.hs
[dotfiles-lemon-haskell-output]:      {{ dotfiles_lemon }}/haskell/MyOutput.hs
[dotfiles-lemon-haskell-pipehandler]: {{ dotfiles_lemon }}/haskell/MyPipeHandler.hs
