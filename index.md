---
# Entries live in _data/publications.yml and _data/talks.yml — add new ones there.
# This page is just the template; it should rarely need editing.
layout: home
---

## Papers

{% include entry-list.html entries=site.data.publications %}

## Conference Talks

{% include entry-list.html entries=site.data.talks %}

![visitors](https://visitor-badge.laobi.icu/badge?page_id=takahiroharada/takahiroharada.github.io) (since 2025/9)

<style>
.pub-list { list-style: none; margin-left: 0; padding-left: 0; }
.pub-year { border-bottom: 1px solid #444; padding-bottom: .2em; margin-top: 1.4em; }
.pub { margin-bottom: 1em; overflow: hidden; }
.pub-thumb { float: right; width: 64px; margin: .2em 0 .5em 1em; }
.pub-title { font-weight: 600; }
.pub-authors { color: #aaa; font-size: .95em; }
/* Venue and all links share one line to keep the list compact. */
.pub-meta { font-size: .95em; line-height: 1.9; }
.pub-venue { font-style: italic; color: #aaa; margin-right: .4em; }
.pub-meta a,
.pub-meta .pub-link-dead {
  display: inline-block;
  text-decoration: none;
  padding: 0 .5em;
  margin-right: .3em;
  border: 1px solid #5aa9ff;
  border-radius: 4px;
  color: #5aa9ff;
  font-size: .8em;
  line-height: 1.7;
  vertical-align: baseline;
  transition: background-color .15s ease, color .15s ease;
}
.pub-meta a:hover { background-color: #5aa9ff; color: #181818; }
/* Labels with no URL yet — shown muted rather than as a clickable button. */
.pub-meta .pub-link-dead { border-color: #555; color: #777; }
.pub-note { color: #d4a24c; font-size: .85em; margin-left: .2em; }
</style>
