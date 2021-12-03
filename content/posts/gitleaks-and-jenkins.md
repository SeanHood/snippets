---
title: "Formatting Gitleaks for Jenkins' Warnings-ng Plugin"
date: 2021-12-03T14:47:00Z
draft: false
---

I wanted Warnings-ng Plugin to pick up issues reported by Gitleaks. Sadly this isn't a parser that currently exists. However I've managed to do such a thing using `jq`.

First up, [Gitleaks](https://github.com/zricethezav/gitleaks) it a tool which scans a git repo and reports credendials found in plain text. And [Warnings-ng](https://plugins.jenkins.io/warnings-ng/) is a Jenkins Plugin used to find and parse bug/warnings/errors in code and report them neatly in Jenkins UI.

Will skip over much of what gitleaks is/how to use it, their docs will be way better. Anyway I generated my report file like so: `gitleaks detect -r gitleaks.json --redact -v -f json`

Now, we have a file called `gitleaks.json` which will look something like this:

```json
[
  {
    "Description": "AWS",
    "StartLine": 1,
    "EndLine": 1,
    "StartColumn": 15,
    "EndColumn": 34,
    "Context": "access_key = REDACTED",
    "Secret": "REDACT",
    "File": "config/aws.py",
    "Commit": "",
    "Entropy": 0,
    "Author": "",
    "Email": "",
    "Date": "",
    "Message": "",
    "Tags": [],
    "RuleID": "aws-access-token"
  }
]
```

For the conversion, `jq` supports putting it's expression into a file. Since it's much easier to read a multiline expression that's what I'm doing:

```jq
# gitleaks-issues.jq
{
    "issues": [
        .[] | 
        {
            fileName: .File,
            description: .Description,
            category: .RuleID,
            lineStart: .StartLine,
            lineEnd: .EndLine,
            columnStart: .StartColumn,
            columnEnd: .EndColumn
        }
    ]
}
```

Running `jq` with our filter, we get the following output. In Jenkins we'd pipe this to a file.
```shell
$ jq -f gitleaks-issues.jq gitleaks.json
{
  "issues": [
    {
      "fileName": "config/aws.py",
      "description": "AWS",
      "category": "aws-access-token",
      "lineStart": 1,
      "lineEnd": 1,
      "columnStart": 15,
      "columnEnd": 34
    }
  ]
}
```


I've also skipped over the Warnings-ng plugin format, this is semi-documented here: https://github.com/jenkinsci/warnings-ng-plugin/blob/master/doc/Documentation.md#export-your-issues-into-a-supported-format

There exists a Native Format for Warnings-ng which is what my filter translates into.

To put it all together, in your `Jenkinsfile`, things would roughly look like:



```groovy
pipeline {
  stages {
    stage('Scan Creds') {
      // Scan for creds using gitleaks
      sh 'gitleaks detect -l debug -v --redact -f json -r gitleaks-output.json || true'
      
      // Translate gitleaks format into Warnings-ng's native format
      sh 'jq -f gitleaks-issues.jq gitleaks-output.json > gitleaks-formatted.json'
      
      // Tell Warnings-ng to scan for these warnings
      recordIssues(
        tool: issues(
          name: 'Gitleaks',
          pattern: 'gitleaks-formatted.json'
        ),
        qualityGates: [[threshold: 1, type: 'NEW', unstable: true]]
      )
    }
  }
}
```


