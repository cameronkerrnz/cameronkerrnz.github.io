---
title: "Github Actions Step Summary"
date: 2025-04-20T13:34:20+12:00
summary: "Do you find yourself wasting time trawling through logs and downloading CI artefacts to find common types of errors? Use GitHub Actions Step Summaries to accellerate your workflow."
tags: ['GitHub Actions', 'DevOps', 'SRE', 'Programming', 'Software Development', 'SRE', 'logging']
---

In my current role I've been doing a lot more front-end development than I have done before, and that means I have developed a special --- if pained --- appreciation for all the effort that has been put into our browser integration tests. While they are slow, hard to debug and maintain, the value continues to be proven, so we take pains to maintain them as I make changes on our product, which neccessarily or accidentally, do often cause the integration tests to fail when doing front-end work.

But they often fail in particularly non-obvious ways, and typically the test case that fails is that the test harness times out waiting for some UI state which will never occur. In this case, when they do fail I have to:

- view the logs for a failing step -- this is very slow (>10 seconds) to load
- scan the logs typically looking for red text indicating test failures... but if I scroll too quickly I'll miss it. Unfortunately I can't search for the colour 'red' in the UI.
- I'll likely need to download the particular artefacts (logs) saved by the test automation
- unpack the artefacts, navigate the directory structure and look at:
  - captured screenshot... only sometimes useful
  - python app server logs, might contain an unhandled exception, or possibly a client-reported Javascript exception

**The key point here being that there are a small handful of ways that errors might manifest in the logs, and that with some automation I could search for these kinds of errors and display them easily, saving a whole lot of time.**

So... how to expose this in GitHub Actions? This is where [GitHub Actions Step Summary](https://github.blog/news-insights/product-news/supercharging-github-actions-with-job-summaries/) comes into it. This functionality was introduced in 2022, so its likely that well-established projects may not yet be making use of this.

Step Summaries are essentially the ability to write to a Markdown file that GitHub Actions will look for (`$GITHUB_STEP_SUMMARY`), and then will display that content, if it exists and contains any content. If your workflow contains multiple steps, each step will have its own summary.

It's quite possible to use shell scripting... but it gets complex pretty quickly, particularly if you need to:

- parse structured formats such as XML from xunit formatted unit-tests
- parse JSON blobs captured within logs and extract information from those

What I present here is just an example to get you started. Obviously, your environment will be very different from mine, so I've tried to remove much of the specifics to show just a few examples.

## Basic Structure

Top tip here is that when developing this, you want to avoid doing tests through you CI much. Download the artefacts, and then develop against those. In my case, our CI scripts set `$BUILD_ARTIFACTS_DIR` to point to where it collects the various log files etc. When developing this script, I would download the build artefacts and then point `$BUILD_ARTIFACTS_DIR` when executing the script.

I'm going to show some examples that use XML, JSON, extracting Python stack frames, and extracting red text, but first, here's the basic structure:

```python
#!/usr/bin/env python3

import glob
import io
import json
import re
import sys
import textwrap
import os
import subprocess
import xml.etree.ElementTree as ET

# ...)

def main():
    global output
    global build_artifacts_dir

    if os.getenv('GITHUB_STEP_SUMMARY') is not None:
        output = open(os.getenv('GITHUB_STEP_SUMMARY'), 'a')
    else:
        output = sys.stdout

    build_artifacts_dir = os.getenv('BUILD_ARTIFACTS_DIR', 'build-artifacts')

    # ... functions that look for different types of common errors
    unittest_section()
    browser_integration_section()
    javascript_fatal_on_client()
    python_unhandled_exception()

if __name__ == "__main__":
    main()
```

## Parsing Unit Test Results

Plenty of test frameworks in different languages can report their results, including any stdout/stderr output, in an XML format called XUnit. So it makes some sense to separate out the XUnit specifics from the specifics of which files to look for. You might have multiple tools in your CI environment that can make use of the XUnit format.

```python
def xunit_section(glob_pattern):
    global output
    global build_artifacts_dir

    def render(testcase, failure):
        classname = testcase.attrib['classname']
        name = testcase.attrib['name']
        message = failure.attrib['message']

        output.write(textwrap.dedent('''
            ### Class: {classname}  Test: {name}
            
            > `{message}`
            
            ''').format(classname=classname, name=name, message=message))

        output.write('\n'.join(['    ' + line for line in failure.text.split('\n')]))

    files = glob.glob(os.path.join(build_artifacts_dir, glob_pattern))

    for file in files:
        tree = ET.parse(file)
        testsuite = tree.getroot()
        for testcase in testsuite.findall('testcase'):
            failure = testcase.find('failure')
            if failure is not None:
                render(testcase, failure)

def unittest_section():  ## Eg. specialise for your unittest frameworks.
    xunit_section('path/to/unittests/*-xunit.xml')
```

## Finding Red Text (ANSI escape sequences)

You might have errors that are reported in logs, but not in some structured format such as XUnit. In this particular example, I'm able to use the observation than one of my testing tool emits ANSI escape sequences to emit test results in either green or red. So here I'm looking to show just the red text. That's easily achieved with `grep`, so I let it do the heavy lifting, and use Python to strip out the escape sequences and format the Markdown.

```python
def browser_integration_section():
    global output
    global build_artifacts_dir

    files = glob.glob(os.path.join(build_artifacts_dir, 'path/to/browser_integration_tests/report-*.txt'))

    for file in files:
        report_file = file.split('/')[-1]

        red_text = subprocess.run(['grep', '\033\\[31m', file], capture_output=True, check=False).stdout.decode('utf-8')

        if red_text == '':
            continue

        output.write(textwrap.dedent('''
            ### Report file: {report_file}

            ''').format(report_file=report_file))

        for line in red_text.split('\n'):
            # strip out red and reset escape seqeuences, and extract just the message
            stripped = line.replace('\033[31m', '').replace('\033[0m', '')
            parts = stripped.split(' ', 3)
            if len(parts) >= 4:
                output.write('    ' + parts[3] + '\n')
```

And here's the sample output. This will likely be emitted with another section, such as an Unhandled Python Exception, or a Javascript Browser-Reported Exception. There would also be a captured screenshot showing what the screen looked like, but I'll discuss that later.

```
### Report file: report-prepare-environment.txt

    prepare-environment.js - Prepare environment - Products - opens admin page
    Passed.
    Passed.
    Passed.
    timeout: timed out after 60000 msec waiting for the home page to load
```

## Javascript Browser-Reported Exceptions

It's really useful to make sure any Javascript client-side errors get reported. You might have a solution such as Sentry or any other APM tool. But does this include your CI environment? Otherwise, if a browser test fails waiting for a button-click to action something, but the button had an exception, then you're not going to know what happened until you try and manually reproduce the problem.

```python
def javascript_fatal_on_client():
    global output
    global build_artifacts_dir

    decoder = json.JSONDecoder()

    files = glob.glob(os.path.join(build_artifacts_dir, 'path/to/python_app_server/*.log'))
    for file in files:
        grepped = subprocess.run(
            ['grep', '-o', 'JavaScript FATAL on client:.*', file],
            capture_output=True, check=False).stdout.decode('utf-8')

        for line in grepped.split('\n'):
            if line.strip() == '':
                continue

            try:
                likely_starts_with_json = line.split(': ', 1)[1]
                event, _where_end = decoder.raw_decode(likely_starts_with_json)

                output.write('\n### JavaScript FATAL on client:\n\n````\n')

                for l in event['exception'].split('\r\n')[:10]:
                    if l != 'Stack trace:' and not l.startswith('Exception:'):
                        output.write('  ')
                    output.write(l.strip() + '\n')
                output.write('```\n')
                output.write('\nMessage:\n\n> ' + event['message'] + '\n')
            except ValueError:
                output.write('\nExtract:\n\n> ' + line[:100] + '\n')
```

And here is the example output. In this instance I deliberately caused an exception by calling a method on `undefined`, which is a pretty common type of exception.

````
### JavaScript FATAL on client:

```
Exception: undefined is not an object (evaluating 'undefined.deliberately_breaking_the_build')
Stack trace:
  main_page@http://127.0.0.1:6544/static/js/testing/MainPage/MainPage.js
  init@http://127.0.0.1:6544/static/js/testing/MainPage/MainPage.js
  http://127.0.0.1:6544/static/js/testing/main.js:46:44
  collect@http://127.0.0.1:6544/static/js/lib/underscore-1.7.0.min.js
  initApplication@http://127.0.0.1:6544/static/js/testing/main.js:44:45
  http://127.0.0.1:6544/static/js/testing/main.js:66:28
  execCb@http://127.0.0.1:6544/static/js/lib/require.min.js:29:234
  check@http://127.0.0.1:6544/static/js/lib/require.min.js:18:391
```

Message:

> Cannot load all resources: require; null; undefined is not an object (evaluating 'undefined.deliberately_breaking_the_build'); 
````

## Unhandled Python Exceptions

Here's a sample of an unhandled exception... you likely have errors such as this which appear in your CI environment but are not relevant outside of it. Pragmatically it's useful to have a strategy to ignore such errors.

```
### Uncaught Python exception

    Traceback (most recent call last):
      File "redacted/utils/security_tweens.py", line 14, in csrf_tween
        if request.POST and request.path not in whitelist_urls:
      File "redacted/local/lib/python2.7/site-packages/webob/request.py", line 787, in POST
        self.make_body_seekable()
      File "redacted/local/lib/python2.7/site-packages/webob/request.py", line 930, in make_body_seekable
        self.copy_body()
      File "redacted/local/lib/python2.7/site-packages/webob/request.py", line 954, in copy_body
        data = input.read(min(todo, 65535))
      File "redacted/local/lib/python2.7/site-packages/webob/request.py", line 1540, in readinto
        data = self.file.read(sz0)
      File "redacted/local/lib/python2.7/site-packages/paste/httpserver.py", line 491, in read
        data = self.file.read(length)
      File "/usr/lib/python2.7/socket.py", line 384, in read
        data = self._sock.recv(left)
    timeout: timed out
```

You could do this a number of ways... here I'm just making a regular expression to apply against the generated Markdown. I've left `grep` to do again do the heavy lifting to find the beginning of the message, and then simplify matters by having `grep` output the next 50 lines, rather than trying to use something like `sed` or `awk` to find the end of the message. I just have Python find the end of the message instead with the observation that the subsequent lines are indented by 2 spaces... plus I want the line after that. Certainly something that `sed` and `awk` are capable of... but Python is easier for others to read, maintain, debug and reapproriate to other uses.


```python
def python_unhandled_exception():
    global output
    global build_artifacts_dir

    tmpbuf = io.StringIO()

    files = glob.glob(os.path.join(build_artifacts_dir, 'path/to/python_app_server/*.log'))
    for file in files:
        grepped = subprocess.run(
            [
                '/usr/bin/egrep', '-A', '50', '--color=never',
                r'Uncaught exception during processing',  # tailor this to your needs
                file
            ],
            capture_output=True, check=False)

        if grepped.returncode != 0:
            continue

        tmpbuf.write('\n### Uncaught Python exception\n\n')

        captured = grepped.stdout.decode('utf-8')

        lines = captured.split('\n')[1:]  # throw away "Uncaught exception ..."
        # Next line should be '^Traceback (most recent call last):'
        # then line pairs with '^  File ' and '^    code...'
        # then finally (normally?) the exception such as 'ValueError: '
        # At the moment, not trying to strip uninteresting stack frames
        for line in lines:
            tmpbuf.write('    ' + line + '\n')
            if not line.startswith('Traceback ') and not line.startswith('  '):
                break
        tmpbuf.write('\n')

        tmpbufstr = tmpbuf.getvalue()

        # https://regex101.com is the fastest way to debug regular expressions.

        if re.search(
            r'''
            ^\s+File[ ]".*/site-packages/webob/request\.py".*in[ ]readinto$\n
            (?:^.*$\n){5}  # skip ahead 5 lines
            ^\s+timeout:[ ]timed[ ]out
            ''',
            tmpbufstr, re.MULTILINE | re.VERBOSE
        ) is not None:
            continue

        # ... other matches to ignore ...

        output.write(tmpbufstr)
```

Here's a sample output, using a different exception message, which I generated by throwing an exception:

```
### Uncaught Python exception

    Traceback (most recent call last):
      File "redacted/lib/python2.7/site-packages/pyramid/tweens.py", line 22, in excview_tween
        response = handler(request)
      File "redacted/lib/python2.7/site-packages/pyramid/router.py", line 158, in handle_request
        view_name
      File "redacted/lib/python2.7/site-packages/pyramid/view.py", line 547, in _call_view
        response = view_callable(context, request)
      File "redacted/lib/python2.7/site-packages/pyramid/viewderivers.py", line 302, in _secured_view
        return view(context, request)
      File "redacted/lib/python2.7/site-packages/pyramid/viewderivers.py", line 257, in wrapper
        response = view(context, request)
      File "redacted/lib/python2.7/site-packages/pyramid/viewderivers.py", line 442, in rendered_view
        result = view(context, request)
      File "redacted/lib/python2.7/site-packages/pyramid/viewderivers.py", line 147, in _requestonly_view
        response = view(request)
      File "redacted/views/main_page.py", line 52, in main_page
        raise ValueError({'message': 'Cameron is breaking the build again'})
    ValueError: {'message': 'Cameron is breaking the build again'}
```

## Screenshots

In browser integration tests, it's often desirable to harvest a screenshot. Though admittedly, when you've done the work to highlight the actual error that lead to a timeout situation (ie. waiting for a UI state that would never come) then the screenshot isn't as useful as you might expect.

Markdown can certainly display images, but the problem is that in the context of a GitHub Action Step Summary, the image would have be publically available. You _could_ imagine uploading such screenshots to the likes of an S3 bucket with web serving and appropriate lifecycle rules, but I didn't wish to proceed along that path.

I did, however try to make the image smaller, convert to lower-quality grayscale JPEG and store it inline as a `data:` URI image. This was readily achievable, particularly with ChatGPT writing much of the code to run the ImageMagick tooling to resize the image and convert to Base64.

However, while that does work in a regular Markdown document, it does not work in the context of a Markdown documented surfaced by GitHub (excluding perhaps GitHub Pages). Such Markdown goes through a sanitisation layer and the `data:` URI gets stripped from the `src` attribute of the `img`. This may have worked once upon a time, but has not worked since perhaps 2014, and despite many requests to get this fixed, there is no indication that this will ever come to pass. See [github/markup issue #270](https://github.com/github/markup/issues/270) and [gjtorikian/html-pipeline pull-request #227](https://github.com/gjtorikian/html-pipeline/pull/227) for more insight.
