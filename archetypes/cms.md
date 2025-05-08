---
author: {{ .Params.author | default "Luqinthar Sudarsono" }}
title: {{ .Params.title | default "" }}
description: {{ .Params.description | default "" }}
date: {{ .Date }}
showToc: {{ .Params.showToc | default true }}
tags:
{{ range .Params.tags }}  - {{ . }}
{{ end }}
---
