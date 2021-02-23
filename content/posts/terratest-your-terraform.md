---
title: "Terratest your Terraform (Plans)"
date: 2021-02-23T14:03:00Z
draft: true
---

## Problem statement

As an infrastructure engineer writing Terraform, I'd like to test code before I apply it. Some constraints: I'd like this to work with Terraform Cloud and workspaces.

Workspaces represent `app-region-env` tuples more or less. They all apply the same code, and each are 99% the same, but some have slight differences. These differences are what I'd like to test.

