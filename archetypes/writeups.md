+++
title = '{{ replace .File.ContentBaseName "-" " " | title }}'
date = '{{ .Date }}'
draft = true

# One-liner used in listings, RSS, and social previews.
description = ''

# Pick exactly one so the topic accent is unambiguous:
# ctf | bugbounty | cve | research
categories = ['ctf']
tags = []

# Multi-part? Give every part the same series name.
# series = ['']

# featuredImage = ''
+++

## TL;DR

<!-- Two sentences. What was broken, what it let you do. -->

## Target

{{</* scope title="Target" */>}}
- `` — in scope
{{</* /scope */>}}

## Recon

## The bug

## Exploitation

## Impact

## Timeline

{{</* timeline */>}}
{{ .Date.Format "2006-01-02" }} | Reported
{{</* /timeline */>}}
