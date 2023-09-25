# Helm for Botiga Backend

- This is a helm chart for the Botiga Backend application.

## Debug Templates

- To debug the templates, you can use the following command:

```bash
helm template prod . --debug > templates.yaml
```

- This will dump the template manifests into a file called `templates.yaml` in the current directory
- You can then use this file to debug the templates
