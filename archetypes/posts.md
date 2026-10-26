---
title: "{{ replace .File.ContentBaseName "-" " " | title }}"
slug: "{{ .File.ContentBaseName }}"
date: {{ .Date }}
draft: true
description: ""   # one sentence; used for search results and social cards
projects: []      # aicc | freesbc | cascade-realtime-gateway | mlx-realtime | web-sip-phone
tags: []          # sip, freeswitch, realtime, rtp, testing, ...
---

<!--
Checklist before removing `draft: true`:
- Open with the problem. No preamble.
- At least one concrete artifact: commit permalink (press `y` on GitHub), log,
  capture, measurement, or state diagram.
- Code links use commit SHAs, never `main`.
- No performance claims that are not backed by a published measurement.
- End with a single line linking the repository.
-->
