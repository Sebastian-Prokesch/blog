---
title: "{{ replace .File.ContentBaseName `-` ` ` | title }}"
date: "{{ time.Now.Format "2006-01-02" }}"
# weight: 1
# aliases: ["/first"]
# tags: ["first","second"]
# categories: ["hello", "test"]
author: "Sebastian"
# author: ["Me", "You"] # multiple authors
showToc: true
TocOpen: false
draft: true
hidemeta: false
comments: false
# canonicalURL: "https://canonical.url/to/page"
disableHLJS: true # to disable highlightjs
disableShare: false
hideSummary: false
searchHidden: true
ShowReadingTime: true
ShowBreadCrumbs: true
ShowPostNavLinks: true
ShowWordCount: true
ShowRssButtonInSectionTermList: true
UseHugoToc: true
cover:
    # image: "<image path/url>" # image path/url
    # alt: "<alt text>" # alt text
    # caption: "<text>" # display caption under cover
    # relative: false # when using page bundles set this to true
    hidden: true # only hide on current single page
editPost:
    # URL: "https://github.com/<path_to_repo>/content"
    # Text: "Suggest Changes" # edit text
    appendFilePath: true # to append file path to Edit link

# summary: A Blog post about hello world
---
