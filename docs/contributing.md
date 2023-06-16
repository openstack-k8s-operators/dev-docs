Contributing to documentation
=============================

## Rendering documentation locally

Install docs build requirements into virtualenv:

```
python3 -m venv local/docs-venv
source local/docs-venv/bin/activate
pip install -r docs/doc_requirements.txt
```

Serve docs site on localhost:

```
mkdocs serve
```

Click the link it outputs. As you save changes to files modified in your editor,
the browser will automatically show the new content.


## Patterns and tips for contributing to documentation

* If possible, try to make code snippets copy-pastable. Use shell
  variables if the snippets should be parametrized. Use `oc` rather
  than `kubectl` in snippets.
