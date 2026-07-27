---
# Leave the homepage title empty to use the site title
title: 
date: 2025-10-08
type: landing

sections:
  - block: hero
    content:
      title: Compilers and systems for faster intelligent computing.
      image:
        filename: welcome.jpg
      text: |
        CAT Lab at George Mason University builds compilers, architecture, and performance tools that make AI systems easier to understand and faster to run.
      cta:
        label: Explore Software
        url: software/
        icon_pack: fas
        icon: code
      cta_alt:
        label: View Publications
        url: publication/
      cta_note:
        label: Compiler · Architecture · Toolkit
  
  - block: collection
    content:
      title: Latest News
      subtitle: What we are publishing, presenting, and building.
      text: ''
      count: 3
      filters:
        author: ''
        category: ''
        exclude_featured: false
        publication_type: ''
        tag: ''
      offset: 0
      order: desc
      page_type: news
    design:
      view: compact
      columns: '1'
  
  # - block: markdown
  #   content:
  #     title:
  #     subtitle: ''
  #     text:
  #   design:
  #     columns: '1'
  #     background:
  #       image: 
  #         filename: coders.jpg
  #         filters:
  #           brightness: 1
  #         parallax: false
  #         position: center
  #         size: cover
  #         text_color_light: true
  #     spacing:
  #       padding: ['20px', '0', '20px', '0']
  #     css_class: fullscreen

  - block: collection
    content:
      title: Featured Publications
      subtitle: Recent research from across the computing stack.
      text: ''
      count: 4
      filters:
        folders:
          - publication
        featured_only: true  
    design:
      view: citation
      columns: '1'

  - block: markdown
    content:
      title:
      subtitle:
      text: |
        {{% cta cta_link="./people/" cta_text="Meet the team" %}}
    design:
      columns: '1'
---
