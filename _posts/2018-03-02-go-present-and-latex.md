---
layout: post
title: Fixing golang present tool limitations by LaTeX
tags:
- presentations
---

[Golang present](https://godoc.org/golang.org/x/tools/present) tool is very
handy to create quick, minimal presentations. Install the package, edit a text
file and start serving. Text files are easier to manage than a bloated
desktop presentation tool (looking at each one of them in particular) - I can
check in my files, make quick changes and take notes on the fly. Best of
all - I note down unanswered questions in the 'Q & A' section of my **.slide** file.
I can't tell you how many times it has helped me whip up a quick and a dirty
presentation to appease the higher management.

If you haven't played with it before, you can check out some sample presentations
created using the present tool on the [golang talks site](https://talks.golang.org/2017).

No doubt that it is a useful tool, but it some visible limitations - some of which
are deal breakers for those who treat their presentation game seriously:

1.**The unwanted corner preview of the next slide**:

Notice how the next slide is peeking out as highlighted:

<div class='pull-left' style="border: 0px solid black;">
{% include figure.html path="blog/go-present-latex/slide-preview-problem.png" alt="slide preview problem" url=url_with_ref %}
</div>

Some may argue that this is a feature. But when I am presenting, I feel that the
audience should have their full attention on the current slide. Having a preview
of the next slide distracts the audience. At the very least, the preview should be an
optional feature - which one should be able to turn off.

2.**Lack of progressive disclosure support**:

If I want to list 3 bullet points on a slide, I want to show them one by one to
the audience, so that I can explain the current point and do some build up for
the next point. It is human nature to read everything you show on the screen
which distracts the user from a deep, philosophical discussion around the first
question to a seemingly inconsequential thought on the third one as shown below: <br/>

<div class='pull-left' style="border: 0px so  lid black;">
{% include figure.html path="blog/go-present-latex/progressive-disclosure-problem.png" alt="progressive disclosure problem" url=url_with_ref %}
</div>

To rephrase the above 2 points in fancy words: <b><i>avoid cognitive overload.</i></b>

3.**Export to PDF breaks once in a while**:

After walking your engaging audience through a refreshing session on - **103 ways to
chew your pencil** - you certainly want to share the slide deck with them. The preferred
way to do so in the golang present world is to print and save the presentation as PDF,
which ever so often, breaks as follows:<br/>

<div class='pull-left' style="border: 0px so  lid black;">
{% include figure.html path="blog/go-present-latex/pdf-export-problem-1.png" alt="pdf export problem 1" url=url_with_ref %}
</div>

Some points tumble over to the next page, or worse - get stuck half way:<br/>

<div class='pull-left' style="border: 0px so  lid black;">
{% include figure.html path="blog/go-present-latex/pdf-export-problem-2.png" alt="pdf export problem 2" url=url_with_ref %}
</div>

Because, golang present exports slides to pdf, I wanted to know if there was an
intermediate step where in I could edit them. You know, like using (<b>shudder cue</b>)
[LaTeX](https://www.latex-project.org/). I am sure there are many who swear by its
simplicity. I can relate to that because I have been there. But getting into the
LaTeX zone takes time and I don't want to spend more time exploring my presentation
tool than necessary. And more importantly, I want to make my learnings stick. With
LaTeX - 10 minutes after my nice, shiny, slick - PDF document is created - I forget more
than half of what I learnt about LaTeX along the way.

But the very awesome [Sebastien Binet](https://github.com/sbinet) created
[present-tex](https://github.com/sbinet/present-tex) and I chanced upon it through
[this](https://www.reddit.com/r/golang/comments/3wrbng/presenttex_a_present_slide_to_latexbeamer/)
reddit thread.

All you need to do now is:

{% highlight text %}
$ present-tex my.slide > my.tex
$ pdflatex -shell-escape my.tex
{% endhighlight %}

The **pdflatex** tool is a part of the [MacTex](https://www.latex-project.org/get/) distribution on
Mac and [TexLive](http://www.tug.org/texlive/) on Linux. But is a huge package - it was around
2.9 GBs the last time I checked on a Mac. I searched for a docker container alternative just
in case the installation breaks and I don't want to clean up. Thankfully, another awesome
open source contributor - [Julian Didier](https://github.com/theredfish/docker-pdflatex) has
hosted it [here](https://github.com/theredfish/docker-pdflatex). So the exact steps I follow
are:

{% highlight text %}
$ present-tex my.slide > my.tex

$ pdflatex -shell-escape my.tex

# attach bash to the container named pdflatex
$ docker exec -it pdflatex bash
$ cd shared/folder

# Generate the pdf from tex file in the current directory
$ pdflatex -interaction=nonstopmode -halt-on-error \
$ -output-directory -shell-escape . my.tex  
{% endhighlight %}

And this does all the hard work gives me a PDF that just works. It solves:

1. **The unwanted corner preview of the next slide**: - because I can display a PDF document
   page by page.

2. **Export to PDF breaks once in a while problem** - because I am getting a full fledged
  PDF document so there isn't a need to export it.

But one problem - **Lack of progressive disclosure support** is still there, because
a PDF can't do progressive disclosure unless you want to go back to the world of
LaTeX and use the [powerdot](https://www.sharelatex.com/learn/Powerdot) package.
I wanted to keep my life simple, so I did the following hack:

I edited the **.tex** file created by **present-tex** and I made changes to
the slides where I wanted progressive disclosure, i.e I changed:

{% highlight text %}
\begin{frame}[fragile]
\frametitle{Who let the dogs out:}

\begin{itemize}
\item Who?
\item Who?
\item Who?
\item Who?
\end{itemize}

\end{frame}
{% endhighlight %}

to

{% highlight text %}

\begin{frame}[fragile]
\frametitle{Who let the dogs out:}

\begin{itemize}
\item Who?
\end{itemize}

\end{frame}

\begin{frame}[fragile]
\frametitle{Who let the dogs out:}

\begin{itemize}
\item Who?
\item Who?
\end{itemize}

\end{frame}

\begin{frame}[fragile]
\frametitle{Who let the dogs out:}

\begin{itemize}
\item Who?
\item Who?
\item Who?
\end{itemize}

\end{frame}

\begin{frame}[fragile]
\frametitle{Who let the dogs out:}

\begin{itemize}
\item Who?
\item Who?
\item Who?
\item Who?
\end{itemize}

\end{frame}


{% endhighlight %}

which creates the following effect as I move from one slide to the next:

<div class='pull-left' style="border: 0px solid black;">
{% include figure.html path="blog/go-present-latex/progressive-disclosure.gif" alt="progressive disclosure dry violation" url=url_with_ref %}
</div>
