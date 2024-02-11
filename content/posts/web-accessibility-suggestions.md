+++
title = "Web Accessibility & Suggestions"
date = 2022-07-01T16:52:58.775Z
tags = ["accessibility", "english"]
slug = "web-accessibility-suggestions"
+++

Web accessibility defines a broad spectrum of practices that make browsing the
internet more accessible. Through the increased focus on accessibility of the
web pages, it becomes possible to make a website inclusive to a variety of
disability groups including but not limited to those with visual, auditory, and
motor impairments. The accessibility of web pages gained more importance as the
world wide web became a standard and the implementation of web browsers ended
up in many devices we use today. Thanks to this distributed aspect of the web,
it has become easier to draft standards for it, since websites rely on the same
protocol, and not every device needs adjustment based on their capabilities.

The history on web accessibility dates back to the early days of internet; in
fall of 1996, the organization “Web Accessibility Initiative (WAI)” was founded
under the foundation of US-based World Wide Web Consortium (Dardailler, 2009).
Since then, this initiative has provided and directed standards on how to make
web pages more accessible. The organization also serves as the formal
documentation/guideline for web developers, who implement the standards issued.
The organization also focuses on research and rapid development on
accessibility. For example, recent research addresses the inaccessibility of
CAPTCHA (a technology used to distinguish users from web automation bots),
which relies on identification and selection of curated visuals (Hollier et
al., 2021).

Web accessibility is also subject to legislation in many countries. In the
European Union, a law under “Web and Mobile Accessibility Directive” was passed
in 2016. The aim of the law is to make the websites and mobile applications of
public sector organizations accessible to people with disabilities, which
accounts for approximately eighty million people in the EU. The law not only
forces the implementation of accessibility practices in public sector
applications, but also requires an implementation of a feedback mechanism in
which people could report the issues they are having with the applications they
use (“Directive 2016/2102”, 2016). In the United States there are a variety of
laws that concern the technology and accessibility for disability groups, the
latest one being “Twenty-First Century Communications and Video Accessibility
Act” (n.d.), which also covers private sector (“Web Accessibility Laws &
Policies”, 2018). In Turkey, there is no law specifically concerning the
accessibility of the web, however it is required by Act №5378 that measures be
taken to ensure the accessibility of technological communication tools.
Research conducted by Doğan (2019) concluded that the majority of the websites
of the public sector in Turkey failed to deliver the accessibility standards,
and more emphasis and audition is necessary to enforce the accessibility of
these websites.

---

To understand the mechanisms of web accessibility, it is helpful to look at
most common issues websites have and how they could be solved. One of the most
prominent issues is the presentation of visual material, namely images. When
images appear on a web page, users with blindness or partial visual impairment
cannot comprehend the structure of the content, and naturally they have to
resort to the context and guess what’s in the picture. To make serving of
images more accessible, web developers need to provide an “alt text” for
images, an HTML attribute which could be caught by screen readers or similar
accessibility software. Of course, the images are not always static, nowadays,
user generated content fills the web. In this case, developers need to
implement solutions so that this “alt text” would be entered by the user
itself.

Another issue regards the partially blind. It is very common to see web pages
consisting of blocks of text with a very low contrast rate. Similarly, small
font-sizes are also a problem. To fix these issues, developers can use
automated tools which suggest darker or lighter colors depending on the
context as well as suggestions on font size, using WAI recommendations (such
tools include “axe DevTools” and “Google Lighthouse”).

Another common issue is related to the structure of a webpage. It is important
to have good semantic distinction of the sections of a web page so that the
users can navigate easily while only using the keyboard. For example, the main
sections of the website should be marked so that users can frequently make
transitions from one section to another. It is also helpful to mark certain
elements of the page with proper HTML counterparts. For example, the main
heading (or title) of the page should be marked as “h1” and following
subheadings should continue using “h2”, “h3” and so on, depending on the
level. This semantic markup ensures a more meaningful and easier navigation
experience, as well as giving a robust idea on the page structure for the user.
However, many developers neglect this issue and mark all elements as “div” (a
general tag that has no semantic value) which leads to the page being a load of
unstructured text.

Lastly, accessibility issues are raised when users try to fill a form.
Generally, developers neglect putting proper labels for the forms and users
have no idea which field is being filled. For example, it is common to see a
search box at the top of the web page, and the only indicator is a button with
a magnifier icon (which has no alt text). In such a case, the user navigating
with a screen reader may struggle to figure out what that input does. Another
example is when the users fill invalid information. When the only changing
thing is input’s border being red, users with screen readers have no way to
tell they are entering an invalid value. Considering these, it is important to
mark all form elements with a label, as well as putting a description or title
describing the purpose of the form. It is also important to announce the
dynamic changes in the web page to the user. Solutions for these problems can
be implemented by using “ARIA” attributes, namely “aria-label” and “aria-live”.

---

As it can be seen, the technical aspect of accessibility has a long-running
history thanks to contributions of WAI. However, today it is still the case
that accessibility practices on the web are neglected. One of the most
prominent reasons is that, to account for accessibility developers need to make
extra effort; that means learning more technical knowledge. By this fact, it is
apparent that it is of the utmost importance that developers incorporate
accessibility issues while they are learning the basics of web development. To
achieve this, online tutorials, learning materials and university courses could
be adjusted to include accessibility considerations. For example, instead of
opening an HTML tutorial with the “div” tag, it could be adjusted to introduce
semantic tags first so that the learner can understand that they can use
different constructs to convey meaning on a web page, through code.

Most of the learning material on the web lacks such consideration as they are
generally intended to be fast-learning programs. Since the web sector rapidly
looks for and hires new programmers, corporations’ influence on programmer
profile is very important. For this reason, the private sector could make the
biggest influence by requiring programmers to know about accessibility.
However, one issue is that coding for accessibility costs more, due to
increased development time and considerations; so, the private sector would be
reluctant to invest in such a program. This urges social mobilization to
rapidly demand and prefer accessibility so that the private sector would have
to adapt. This mobilization could be achieved by civil campaigns, through NGO’s
and the intervention of the government. As an example of a civil campaign, a
website could be established which showcases/exposes websites with worst and
best accessibility practices. Corporations would then feel obliged to fix their
website so that they are not on the top of the “worst” list (or they could
increase the accessibility more to be on top of the “best”).

Of course, this process of civil mobilization is a difficult and lengthy one.
During this process, alternative solutions could be developed to reduce
workload. For example, emphasis on automating accessibility could be increased.
Static code analysis tools could be further improved to reduce the burden on
the developer. Development environments such as IDE’s and text editors could be
preloaded with accessibility tools which automatically check issues, for
increased awareness.

In summary, the accessibility issue calls for the voice of society (as it is
always the case for such matters), however since there is a technical aspect to
it, improvements can still be made by increasing computer automation that would
fix erroneous developer design. Awareness on a personal level is also quite
important since it takes one persons’ effort to make a website from unusable to
usable; that signifies the inducing of awareness through learning material.

## References

Dardailler, D. (2009, June).  _WAI History_. W3C. Retrieved June 1, 2022,
from  [https://www.w3.org/WAI/history](https://www.w3.org/WAI/history)

_Directive 2016/2102_. EUR-Lex. (2016, December). Retrieved June 1, 2022,
from  [https://eur-lex.europa.eu/eli/dir/2016/2102/oj](https://eur-lex.europa.eu/eli/dir/2016/2102/oj)

Doğan, A. (2019).  _Türkiye’deki Devlet Kurumlarına Ait Web Sitelerinin Web
Erişilebilirliği Durumu ve Bölgeler Arasındaki Farklılıkların İncelenmesi_.

Hollier, S., Sajka, J., White, J., & Cooper, M. (Eds.). (2021, December).
_Inaccessibility of CAPTCHA_. W3C. Retrieved June 1, 2022,
from  [https://www.w3.org/TR/turingtest/](https://www.w3.org/TR/turingtest/)

_Twenty-First Century Communications and Video Accessibility Act_. Federal
Communications Commission. (n.d.). Retrieved June 1, 2022,
from  [https://www.fcc.gov/general/twenty-first-century-communications-and-video-accessibility-act-0](https://www.fcc.gov/general/twenty-first-century-communications-and-video-accessibility-act-0)

_Web Accessibility Laws & Policies_. Web Accessibility Initiative (WAI).
(2018). Retrieved June 1, 2022,
from  [https://www.w3.org/WAI/policies/](https://www.w3.org/WAI/policies/)
