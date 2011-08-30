---
layout: post
title: Taking Jekyll For A Test Drive
---

I've been trying to make regular blogging a part of my routine for a
few years now. I'm pretty sure a large part of what has been stopping
me is the impedance mismatch between the tools I use daily and the
tools that exist for blogging. Then, a couple weeks ago, I stumbled on
[this post][1] by [GitHub][2]'s Tom Preston-Warner. "Ah-ha!" I said,
"this sounds more like it!" and I resolved to give it a try ("some
day.")

It just so happens that I have *also* been wanting to migrate off of
my SliceHost box to a t1.micro over on AWS. I finally decided to
buckle down and work on this project today, and when I reached the
part where I needed to migrate my blog, I thought to myself: "Self,
now would be a good time to ditch WordPress and try that 'Blogging
Like A Hacker' thing."

The tool chain was pretty easy to set up. I'm an emacs junkie, of
course, so I went looking for emacs integration right away. A quick
Google brought me to [Jack Moffitt's post][3] about managing jekyll
from emacs. *Excellent*. Next step was installation. I ran into some
confusion here, as I couldn't get `gem` to find the correct version of
jekyll (0.11.0, as I want to host this using [GitHub Pages][4].) I
ended up correcting that by updating to a newer gem release via `gem
update --system`. Syntax highlighting for code is a must-have, that
was as easy as `pip install pygments` (of course I already had pip
installed, don't you?) Finally, I downloaded [jekyll.el][5] into my
emacs library directory. I added this to my emacs config:

{% highlight cl linenos %}
;; Jekyll
(require 'jekyll)
(setq jekyll-directory "~/src/misc/plathrop.github.com/")
(global-set-key (kbd "C-c b n") 'jekyll-draft-post)
(global-set-key (kbd "C-c b P") 'jekyll-publish-post)
(global-set-key (kbd "C-c b b") (lambda ()
                                  (interactive)
                                  (dired "~/src/misc/plathrop.github.com")))
{% endhighlight %}

as suggested by the documentation for `jekyll.el`. I did run into one
problem. Jekyll was checking for draft posts like this:

{% highlight cl linenos %}
  (cond
   ((not (equal
          (file-name-directory (buffer-file-name (current-buffer)))
          (concat jekyll-directory jekyll-drafts-dir)))
    (message "This is not a draft post.")
    ...
  ))
{% endhighlight %}

With my `jekyll-directory` set to `~/src/misc/plathrop.github.com`, this
code would end up comparing
`/Users/plathrop/src/misc/plathrop.github.com` to
`~/src/misc/plathrop.github.com`. I *could* have set my
`jekyll-directory` more explicitly, but I thought the code could be a
little more robust:

{% highlight diff linenos %}
   (cond
    ((not (equal
           (file-name-directory (buffer-file-name (current-buffer)))
-          (concat jekyll-directory jekyll-drafts-dir)))
+          (expand-file-name (concat jekyll-directory jekyll-drafts-dir))))
     (message "This is not a draft post.")
    ...
  ))
{% endhighlight %}

I'll have to remember to submit a pull request.

The next step (obviously), is to roll up my sleeves and dive into some
CSS because damn, this be ugly. All in all I'm very happy with the new
setup, but we'll see how it wears after a week or two. New things
always have that glitter and shine, right?

[1]: http://tom.preston-werner.com/2008/11/17/blogging-like-a-hacker.html
[2]: http://github.com
[3]: http://metajack.im/2009/01/02/manage-jekyll-from-emacs/
[4]: http://pages.github.com
[5]: https://github.com/metajack/jekyll/blob/master/emacs/jekyll.el
