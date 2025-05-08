---

author: "{{ .Params.author }}"
title: "{{ .Params.title }}"
description: "{{ .Params.description }}"
date: {{ .Date }}
showToc: {{ .Params.showToc }}
tags:
{{- if .Params.tags }}
{{- range .Params.tags }}

* "{{ . }}"
  {{- end }}
  {{- else }}
  \[]
  {{- end }}

---
